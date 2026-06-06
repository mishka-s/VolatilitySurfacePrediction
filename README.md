# NIFTY Options Implied Volatility Surface Reconstruction

This repository contains a quantitative framework designed to reconstruct a smooth, continuous, and arbitrage-free Implied Volatility (IV) surface from sparse or fragmented NIFTY50 options data. The implementation resolves data omissions caused by liquidity gaps or market-maker quoting drops using a two-tier interpolation scheme: localized cross-sectional geometry reconstruction paired with a joint spatio-temporal radial basis function (RBF) override for contract expiration days.

## Core Algorithmic Pipeline

Reconstructing an option surface requires maintaining baseline empirical realities—specifically, the asymmetric volatility smile across moneyness (skew) and the continuous evolution over time (term structure). The model implements these restrictions via two distinct modules.

### Phase 1: Cross-Sectional Coordinate Scaling and Reconstruction
For standard intraday intervals, missing data points are resolved independently per row (timestamp) and split by option side (Calls vs. Puts) to respect distinct directional skew dynamics.

1. **Log-Moneyness Standardization:**
   To insulate the estimation process from shifting index levels over time, strike prices ($K$) are mapped to a standardized log-moneyness coordinate space relative to the underlying spot/forward price ($F$):
   $$k = \ln\left(\frac{K}{F}\right)$$

2. **Interior Local Hybrid Blend:**
   For a missing contract whose strike falls within the bounds of currently active/observed contracts, the model uses a blended interpolation approach:
   * A global linear spline determines a safe baseline.
   * A localized cubic polynomial (`INTERIOR_DEG = 3`) is fitted over a sliding neighborhood of the nearest 6 active contracts (`INTERIOR_NPTS = 6`).
   * The final interior imputation is a weighted combination ($\alpha = 0.7$) of both:
     $$\text{IV}_{\text{final}} = (1 - \alpha) \cdot \text{IV}_{\text{linear}} + \alpha \cdot \text{IV}_{\text{cubic}}$$
   This captures the natural non-linear curvature of the option smile while strictly suppressing the wild edge-oscillations typical of pure higher-degree polynomial interpolation.

3. **Regularized Wing Extrapolation:**
   For unobserved target strikes out-of-the-money (OTM) or deep in-the-money (ITM) that sit outside the liquid boundary, the model drops higher-degree fits to prevent runaway values. It isolates the outermost two active contracts (`WING_POINTS = 2`) to determine a local linear trend line. The slope ($s$) is projected outward and regularized via directional damping factors (`LO_WING_DAMP = 0.90`) to prevent unstable or negative volatility extensions at the extreme wings.

### Phase 2: Spatio-Temporal Terminal Patching (2D RBF Override)
Standard cross-sectional methods break down on expiration days because time-to-expiry ($\tau$) collapses rapidly toward zero, inducing highly localized, non-linear volatility spikes across the contract chain. Cross-sectional methods cannot capture this temporal decay because they evaluate timestamps individually.

To resolve this, Phase 2 implements a joint spatio-temporal surface over all active January 27 observations simultaneously:
1. **Feature Engineering:** The optimization problem is mapped into a 2D coordinate system using Log-Moneyness ($k$) to resolve the cross-sectional curve, and the square root of time-to-expiry in hours ($\sqrt{\tau_{\text{hours}}}$) to smooth and stabilize the temporal rate of change.
2. **Thin Plate Spline RBF Interpolation:** A bivariate `RBFInterpolator` fits a global thin-plate spline kernel directly across the joint space of all *originally observed* data points. A small regularization hyperparameter (`smoothing = 1e-4`) acts as a ridge regression constraint to filter high-frequency market noise.
3. **Selective Sub-Surface Replacement:** The 2D joint surface model is evaluated exclusively to overwrite the initial cross-sectional estimates for cells on the expiration date that were originally missing. True market observations are left completely untouched.

---

## Codebase Architecture

The script is structured sequentially to minimize execution overhead and maintain transparency during deployment:

* **Data Parsing (`detect_datetime_col`, `_extract_strike`, `parse_sides`):** Standardizes input structures. It strips arbitrary header prefixes using regular expressions, extracts numerical strikes from mixed string/alphanumeric formats, classifies contract types into `CE`/`PE` arrays, and sorts strings by strike order.
* **`reconstruct_side`:** Executes the core mathematical logic for a single side of the contract chain at a unique timestamp (coordinate scaling, linear/cubic blending, regularized wing extrapolation).
* **`apply_imputer`:** Manages row-by-row matrix processing, applies causal temporal forward/backward fill fallbacks for intervals where entire cross-sections are missing, and handles hard boundary clipping (`[IV_FLOOR, IV_CEILING]`).
* **`apply_jan27_rbf`:** Filters the dataset for the designated expiry window, collects observed baseline data points, optimizes the joint 2D thin-plate spline surface, and applies targeted prediction overrides.

---

## Configuration Parameter Reference

The pipeline's optimization and regularizations are dictated by explicit global hyperparameters at the top of the script:

| Constant | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `XSPACE` | `str` | `"logm"` | Coordinate system format (`"logm"` for log-moneyness, `"money"` for simple moneyness) |
| `INTERIOR_DEG` | `int` | `3` | Degree of the localized polynomial fit for interior points |
| `INTERIOR_NPTS` | `int` | `6` | Minimum number of adjacent contracts used to construct local polynomial windows |
| `ALPHA_INT` | `float` | `0.7` | Weight assigned to the local polynomial curve vs. the linear baseline spline |
| `WING_POINTS` | `int` | `2` | Number of terminal liquid contracts used to calculate extrapolation trends |
| `LO_WING_DAMP` | `float` | `0.90` | Damping multiplier applied to the left-wing extrapolation slope |
| `IV_FLOOR` / `IV_CEILING` | `float` | `0.0` / `8.0` | Physical clipping bounds to ensure outputs remain realistic |
| `RBF_KERNEL` | `str` | `"thin_plate_spline"` | Geometric basis kernel utilized for the 2D spatio-temporal override surface |
| `RBF_SMOOTH` | `float` | `1e-4` | Ridge-type smoothing penalty to filter microstructure noise during RBF fitting |

---

### Input Data Layout
The framework dynamically identifies core structural markers. Your input file should contain:
* A non-numeric timestamp vector (e.g., named `datetime`).
* A column containing the baseline asset price (e.g., containing keywords like `underlying`, `spot`, or `forward`).
* Continuous column sets ending in standard option tags (`CE`, `PE`, `call`, `put`).

### Execution
Run the production pipeline script directly:
```bash
python main.py
```


## Deployment & Usage

### Setup Dependencies
Ensure your environment satisfies the required scientific computing stack:
```bash
pip install numpy pandas scipy
```

### Outputs
Upon processing, the architecture exports two distinct files to the working directory:

1. **`filled_dataset.csv`**: A dense matrix matching the original dataframe shape where all missing elements are fully populated and string representations of time are unified under the `%d-%m-%Y %H:%M` convention.
2. **`submission.csv`**: A flattened key-value pairing designed for immediate submission or downstream calculation engines, pairing unique identifiers with their respective optimized implied volatility predictions:

```csv
id,value
27-01-2026 09:15||NIFTY26JAN2723100CE,0.1425
27-01-2026 09:15||NIFTY26JAN2723100PE,0.1182

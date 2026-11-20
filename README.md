# Purely data-driven and physically-informed prediction of breakthrough curves and their moments from hydraulic parameters in rivers

Code accompanying the manuscript *"Purely Data-Driven and Physically-Informed Prediction of Breakthrough-Curves and their Moments from Hydraulic Parameters in Rivers"* (Chapter 6 of the PhD thesis *"Advances in Numerical Modeling, Physical Representation, and Parameterization of Solute Transport in Rivers"*), by Manuel M. Reyna.

The notebook reproduces every regression, cross-validation experiment, table, and figure of the manuscript.

## What this repository does

Given easily measured reach-averaged hydraulic parameters of a river (mean velocity, shear velocity, width, depth, distance from injection), the code predicts:

1. the temporal moments $m_1$–$m_4$ of a solute breakthrough curve, and
2. the parameters $(v, D, \beta, k)$ of a first-order memory-function exchange model (MFE-FO), from which the moments (and the full breakthrough curve) are then recovered mechanistically.

Two regression models are compared, both working on dimensionless, log-transformed variables:

| Model | Description |
|---|---|
| **log-MLR** | Multilinear regression in log space, i.e. a power law. Exhaustive search over all $2^5$ subsets of inputs. |
| **HGBR** | `sklearn.ensemble.HistGradientBoostingRegressor` using all inputs. |

Predictions are validated with leave-one-out cross-validation (LOOCV) at three increasingly demanding grouping levels: **breakthrough curve**, **tracer test**, and **river**.

The notebook also implements a purely data-driven breakthrough curve reconstruction (**KL-RBF**): a Karhunen–Loève expansion of normalized breakthrough curves, where a Gaussian-RBF ridge regression predicts the null-space shape coefficients from skewness and kurtosis. The first four moments are enforced exactly through a null-space parametrization. This is compared against reconstruction with the MFE-FO model.

## Repository contents

```
.
├── data-driven-prediction.ipynb      # Full analysis (data prep → regressions → CV → breakthrough curve reconstruction → figures)
├── estimated_parameters_breakthrough_curves_with_hydraulic_data.csv   # MFE-FO parameter estimates + hydraulic data per breakthrough curve
├── river_solute_tracer_data.csv     # NOT INCLUDED: raw TIERRAS breakthrough curve time series (see "Data")
└── README.md
```

## Data

The analysis combines two files:

- **`river_solute_tracer_data.csv`**: the raw breakthrough-curve time series and metadata from the **TIERRAS** dataset (Rodríguez et al., 2025), used exactly as downloaded. Place it in the same folder as the notebook. https://www.tierras.org/
- **`estimated_parameters_breakthrough_curves_with_hydraulic_data.csv`**: breakthrough curve hydraulic data (discharge, velocity, shear velocity, slope, width, depth, cross-sectional area), tracer metadata, and the point estimates of the transport-model parameters (first-order and power-law memory functions, with several boundary conditions) obtained in: Reyna, M. M., & Tartakovsky, A. M. (2025). Parameter Estimation in River Transport Models With Immobile Phase Exchange Using Dimensional Analysis and Reduced-Order Models (2510.19664v1). arXiv. https://arxiv.org/abs/2510.19664v1


### Data processing
- Units conversions applied in the notebook: time h → s, distance km → m, slope ‰ → m/m, and the fitted $v$, $D$, $k_r$ from per-hour units to SI.
- Missing hydraulic parameters are completed with $Q = AU$, $A = BH$ and $u_* = \sqrt{gHS_0}$ (applied iteratively), plus linear interpolation with distance within a tracer test.
- Local values are averaged along the reach from the injection to each sampling station.
- Breakthrough curves with negative third central moment are discarded, duplicate station records are merged, and curves with missing hydraulic inputs are dropped, leaving **202 breakthrough curves**.
- Central moments are computed with the trapezoidal rule.

### Notebook structure

| Section | Content |
|---|---|
| Load and complete hydraulic and breakthrough curve data | Parsing of TIERRAS, unit conversion, moment computation, hydraulic-parameter completion, reach averaging, dimensionless groups ($Re$, $Fr$, $\alpha$, $\eta$, $\lambda$), dimensionless moments and parameters, Deng et al. (2001) dispersion benchmark ($\psi$). |
| Learn moments directly or through parameters | Group labels for the three LOOCV levels; exhaustive log-MLR over all input subsets; HGBR; routing predicted parameters through the closed-form MFE-FO moments; correlation matrix; parameter percentile table; scaling exponents with $\lambda$; log-MLR vs HGBR comparison figures. |
| Reconstruction of breakthrough curves | Normalization of breakthrough curves to $z = (t - m_1)/\sqrt{m_2}$, KL expansion ($p = 16$), constrained (KKT) fit of coefficients, `ConstrainedZRBF` class (20 Gaussian RBF centers, ridge $10^{-8}$), breakthrough curve alignment/NRMSE metric, LOOCV at the three levels, MFE-FO reconstruction via numerical inverse Laplace transform, representative-curve figure, summary NRMSE figure. |

### Runtime

Most of the notebook runs in seconds to a few minutes. The slowest parts are the exhaustive log-MLR LOOCV and, especially, the MFE-FO reconstructions of all breakthrough curves for every model and validation level (numerical Laplace inversion), which take up to 10 minutes.


## Using the fitted power laws on a new river

The log-MLR regressions obtained by the TIERRAS dataset are given in the TIERRAS dataset together with range of hydraulic parameters to assess validity of applying the regressions to new locations.

## Notes and limitations

- Results depend on the quality and consistency of the hydraulic data in TIERRAS.
- The inverse Laplace transform routine is a Python translation of K. J. Hollenbeck's `invlap.m` (de Hoog et al., 1982); please cite those works if you reuse it.
- Random seeds are fixed where randomness is used (HGBR `random_state=42`, representative-curve selection `np.random.seed(13)`).

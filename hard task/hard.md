# Hard Task: Multivariate Wrapper for `imputeTestbench`

**Contributor:** [Blesson Thomas](https://github.com/blesson-07)  

## Task Description
Implement a basic multivariate wrapper for benchmarking (supporting matrix input) and create a vignette. Ensure the package passes with no Error/Warning/Note using [Win-builder](https://win-builder.r-project.org/).

## Solution Overview
- Created `mv_impute_errors()`, a function that accepts a matrix/data frame and applies `impute_errors()` to each column.
- Preserves column names and returns a named list of `errprof` objects.
- Wrote a vignette `multivariate-benchmarking.Rmd` demonstrating usage with `EuStockMarkets` dataset.
- Updated deprecated function names (`na.interpolation` → `na_interpolation`, `na.mean` → `na_mean`) throughout the package to eliminate warnings.
- Fixed all documentation and metadata issues.

## Code Repository
- **Branch:** [`main`](https://github.com/blesson-07/imputeTestbench/tree/main) (or your specific branch)
- **Main files:**
  - [`R/mv_impute_errors.R`](https://github.com/blesson-07/imputeTestbench/blob/master/R/MVwrapper.R)
  - [`vignettes/multivariate-benchmarking.Rmd`](https://github.com/blesson-07/imputeTestbench/blob/master/vignettes/multivariate-benchmarking.Rmd)
  - [`DESCRIPTION`](https://github.com/blesson-07/imputeTestbench/blob/master/DESCRIPTION) (updated metadata)

## Win‑builder Results
All three R versions passed with **0 errors, 0 warnings, and 1 informational NOTE** (about maintainer change and Date field). The NOTE is unrelated to code functionality.

| R Version       | Log Link                                                                                     | Status                      |
|-----------------|----------------------------------------------------------------------------------------------|-----------------------------|
| R-release       | [00check.log](https://win-builder.r-project.org/6tzOI57gZ3R4/00check.log)                   | 0 errors ✔, 0 warnings ✔, 1 NOTE ℹ |
| R-oldrelease    | [00check.log](https://win-builder.r-project.org/9l7rLQ1c7079/00check.log)                   | 0 errors ✔, 0 warnings ✔, 1 NOTE ℹ |
| R-devel         | [00check.log](https://win-builder.r-project.org/gHCsm9WNDitu/00check.log)                   | 0 errors ✔, 0 warnings ✔, 1 NOTE ℹ |

## Vignette
The rendered vignette is available at:  
[**multivariate-benchmarking.html**](https://blesson-07.github.io/Multivariate-Support-and-Performance-Improvements-in-imputeTestbench-tasks/hard)  


## Notes
- The remaining NOTE is due to the temporary change of maintainer to myself for receiving Win‑builder emails, and the old `Date` field. It does not affect package functionality.
- All code changes are backward‑compatible and ready for review.

1. Clone my fork: `git clone https://github.com/blesson-07/imputeTestbench`

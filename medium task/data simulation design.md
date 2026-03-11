# Proposal: Multivariate Missing Data Simulation for `imputeTestbench`

**Author:** Your Name  
**Date:** March 2026  
**Goal:** Extend `imputeTestbench` to support structured missingness simulation and benchmarking for multivariate time series, while preserving full backward compatibility.

---

## 1. Motivation

`imputeTestbench` is a powerful tool for comparing univariate time series imputation methods. However, real-world data is often multivariate (e.g., multiple sensors, economic indicators). To realistically evaluate imputation algorithms, we need to:

- Simulate missingness patterns that can affect several variables simultaneously (e.g., system‑wide sensor outages).
- Test methods that exploit cross‑variable correlations (e.g., MICE, `mtsdi`, VAR imputation).
- Assess performance **per variable** (which sensors are hardest to impute?).

This proposal introduces a seamless, backward‑compatible extension that adds multivariate capabilities without disrupting existing workflows.

---

## 2. Current Univariate Architecture (Review)

The existing pipeline (for vectors or `ts` objects) is:

- **`sample_dat()`**  
  Generates a list of `repetition` corrupted copies of the input series, each with exactly a given percentage `b` of missing values. Missingness can be MCAR (random positions) or MAR (blocks of consecutive NAs).

- **`impute_errors()`**  
  Loops over missing percentages → methods → repetitions:
  1. Calls `sample_dat()` for the current percentage.
  2. For each imputation method and each corrupted copy, applies the method and computes an error (RMSE, MAE, etc.).
  3. Stores raw errors in a nested list `errall[[method]][[percentage_index]]` (a vector of length `repetition`).
  4. Averages errors per method/percentage and returns an `errprof` object.

All functions assume univariate input. To support multivariate data, we must extend them without breaking existing code.

---

## 3. Proposed Multivariate Extensions

We will **extend the existing functions** rather than create separate `*_mv` versions. This keeps the API clean and leverages all current features (custom methods, additional arguments, error functions).

### 3.1 Input Detection

At the top of `sample_dat` and `impute_errors`, we add a simple check:

```r
if (is.matrix(dataIn) || is.data.frame(dataIn)) {
  multivariate <- TRUE
  dataIn <- as.matrix(dataIn)
  n_vars <- ncol(dataIn)
  if (n_vars == 1) multivariate <- FALSE   # treat single column as univariate
} else {
  multivariate <- FALSE
}

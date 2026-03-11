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

# Proposal: Structured Missing Data Simulation for Multivariate Time Series in imputeTestbench

---

## 3.2 Missing Data Generation (`sample_dat`)

We extend `sample_dat` to accept a matrix and return a list of corrupted matrices (one per repetition). New arguments control the missingness mechanisms:

- **`mv_mechanism`**: `"mcar"`, `"mar"`, `"block"`, `"system_blackout"` (default `"mcar"`)
- **`mv_mar_func`**: a user-supplied function that takes the full data matrix and returns a probability matrix of the same dimensions (for flexible MAR)
- **`mv_block_size`**: block length (scalar or vector per variable)
- **`mv_n_blocks`**: number of blocks per variable (for independent block missingness)
- **`mv_synchronize`**: logical; if `TRUE`, blocks occur at the same rows for all variables (simulates a system-wide outage)
- **`mv_protect_ends`**: logical (default `TRUE`) to keep first and last time points observed, matching univariate behaviour

Internally, helper functions generate logical masks (matrices of `TRUE`/`FALSE`) which are then applied to the data. This modular design simplifies testing and future extensions.

**Example skeleton:**

```r
sample_dat <- function(datin, ..., mv_mechanism = "mcar", ...) {
  if (!multivariate) {
    # existing univariate code ...
  } else {
    out <- vector("list", repetition)
    for (i in 1:repetition) {
      mask <- switch(mv_mechanism,
        mcar            = matrix(runif(n * n_vars) < b/100, n, n_vars),
        block           = generate_block_mask(n, n_vars, mv_block_size, mv_n_blocks,
                                              mv_synchronize, mv_protect_ends),
        system_blackout = generate_system_blackout_mask(n, n_vars, mv_block_size,
                                                        mv_protect_ends),
        mar             = generate_mar_mask(datin, mv_mar_func, ...)
      )
      corrupted <- datin
      corrupted[mask] <- NA
      out[[i]] <- corrupted
    }
    return(out)
  }
}
```

---

## 3.3 Integration with `impute_errors`

The three nested loops (percentage → method → repetition) remain unchanged. The critical modifications are inside the innermost repetition loop and in the error storage.

**Inside the innermost loop (over repetitions):**

```r
errs <- lapply(out, function(y) {   # y is a corrupted matrix
  filled <- eval(parse(text = toeval))  # imputed matrix (same dimensions)

  if (multivariate) {
    # Compute error per variable using existing vector error functions
    err_vec <- numeric(n_vars)
    for (j in 1:n_vars) {
      err_vec[j] <- eval(parse(text = paste0(errorParameter, '(dataIn[,j], filled[,j])')))
    }
    return(err_vec)   # vector of length n_vars
  } else {
    # univariate: single error value
    return(eval(parse(text = paste0(errorParameter, '(dataIn, filled)'))))
  }
})
```

After processing all repetitions for a given method and percentage, combine the list of error vectors into a matrix:

```r
err_matrix <- do.call(rbind, errs)   # dimensions: repetition × n_vars
errall[[method]][[x]] <- err_matrix
```

Each cell of `errall` becomes a matrix instead of a vector.

---

## 3.4 Error Storage and Final Output

- **Raw storage:** `errall[[method]][[percentage]]` → matrix of size `repetition × n_vars`
- **Final summary:** For each method, compute column means (average error per variable) across repetitions:

```r
out <- lapply(errall, function(method_list) {
  avg_per_perc <- lapply(method_list, colMeans)   # list of vectors
  do.call(rbind, avg_per_perc)                    # matrix: percentages × variables
})
names(out) <- methods
out <- c(list(Parameter = errorParameter, MissingPercent = percs), out)
class(out) <- c("errprof", "list")
attr(out, "errall") <- errall
```

The returned `errprof` object contains, for each method, a matrix where **rows are missing percentages** and **columns are variables** — allowing users to see per-variable performance at a glance.

---

## 3.5 Backward Compatibility

All changes are conditional on input type:

- If `dataIn` is a `vector` or `ts`, the `multivariate` flag is `FALSE` and the original univariate code path is followed unchanged.
- New `mv_*` arguments are simply ignored for univariate input, so existing scripts continue to work without modification.
- Existing unit tests for univariate data must pass unchanged.

---

## 4. Performance Optimization

Multivariate benchmarking can be computationally heavy. We add an optional `parallel` argument to `impute_errors` (default `FALSE`). When `TRUE`, we use the `future` and `future.apply` packages to parallelise the outer loop over percentages — a safe strategy because percentages are independent.

```r
if (parallel) {
  library(future.apply)
  plan(multisession)   # or multicore on Unix
  out_list <- future_lapply(seq_along(percs), function(x) {
    # code for a single percentage (including inner loops)
  })
  # Reassemble errall from out_list
} else {
  # original sequential loops
}
```

This can reduce runtime by **60–80%** on multi-core machines.

---

## 5. Implementation Roadmap

| Phase | Task | Duration |
|---|---|---|
| 1 | Helper functions: `generate_block_mask`, `generate_system_blackout_mask`, `generate_mar_mask` + unit tests | 1–2 days |
| 2 | Extend `sample_dat`: multivariate branch + new arguments + backward compatibility | 2–3 days |
| 3 | Extend `impute_errors`: error computation, storage changes, parallel option | 3–4 days |
| 4 | Documentation: `roxygen2` updates + vignette with `EuStockMarkets` | 2 days |
| 5 | Extend `plot_errors`: faceted plots and heatmaps for multivariate `errprof` | 2 days |
| 6 | Testing: all missingness mechanisms, edge cases, univariate regression tests | Ongoing |

---

## 6. Conclusion

This proposal delivers a robust, backward-compatible extension of `imputeTestbench` to multivariate time series. By building directly on the existing architecture and carefully extending only where necessary, the package's simplicity and reliability are maintained while adding powerful new capabilities.

**Key features:**

- Realistic missingness patterns (system-wide outages, flexible MAR, multiple blocks)
- Per-variable error evaluation
- Optional parallelisation for speed
- Seamless integration — no separate functions, no breaking changes

The result will be a valuable tool for researchers and practitioners working with multivariate time series imputation.

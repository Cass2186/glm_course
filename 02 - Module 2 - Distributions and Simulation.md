---
title: Module 2 — Distributions, sampling, and simulation
tags: [stats, R, module-2, CLT, bootstrap, simulation]
module: 2
created: 2026-07-08
---
21
# Module 2 — Distributions, sampling, and simulation

> [!info] Why this module matters
> Every test statistic, p-value, standard error, and confidence interval in the rest of this course rests on one idea: **the sampling distribution**. Simulating it beats memorising it. If you get comfortable simulating, you'll never again wonder "what does a p-value actually mean?" — you'll just generate one under the null and see.

## Objectives

By the end of this module you should be able to:

1. Use the `d`/`p`/`q`/`r` family of functions for at least Normal, t, exponential, binomial, and Poisson distributions.
2. Explain the difference between a *population distribution*, a *sample*, and a *sampling distribution* — in words and in code.
3. Demonstrate the Central Limit Theorem (CLT) by simulation.
4. Compute a bootstrap confidence interval and compare it to a normal-approximation CI.
5. Verify empirically that a valid test controls Type I error at α = 0.05.
6. Use `set.seed()` and `replicate()` idiomatically.

## Concepts

### The four functions for every distribution

For each distribution R gives you a set of four functions with a consistent prefix:

| Prefix | Meaning | Example |
|--------|---------|---------|
| `d` | **d**ensity (or mass) — height of the pdf/pmf at a point | `dnorm(0)` ≈ 0.399 |
| `p` | **p**robability — CDF: P(X ≤ q) | `pnorm(1.96)` ≈ 0.975 |
| `q` | **q**uantile — inverse CDF | `qnorm(0.975)` ≈ 1.96 |
| `r` | **r**andom draw | `rnorm(5)` gives 5 samples |

You use them constantly:

```r
# Critical value for a two-sided 5% test
qnorm(0.975)               # ≈ 1.96
qt(0.975, df = 20)         # ≈ 2.086 — heavier tails at low df

# p-value from a z-statistic
2 * (1 - pnorm(abs(2.3)))  # two-sided p for z = 2.3

# Simulate 1000 draws from Poisson(rate = 3)
rpois(1000, lambda = 3)
```

Same pattern for `binom`, `pois`, `exp`, `t`, `chisq`, `f`, `beta`, `gamma`, `unif`, and more.

### Population vs sample vs sampling distribution

These get confused constantly. Nail this down:

- **Population distribution** — the mechanism generating the data (`rnorm(mean = μ, sd = σ)`).
- **Sample** — one realized dataset drawn from that mechanism (`x <- rnorm(30, μ, σ)`).
- **Sampling distribution** — the distribution of a *statistic* (mean, median, slope, …) across hypothetical repetitions of the sampling process.

The **standard error** is just the *standard deviation of the sampling distribution*. That's it. There is no magic — `sd(x)` is the spread of one sample; `sd(replicate(10000, mean(rnorm(30, 0, 1))))` is a standard error.

### The Central Limit Theorem, operationally

For independent, identically distributed observations from nearly any population with finite variance, the sampling distribution of the mean approaches Normal as sample size grows. Dependence, changing distributions, or infinite variance require other theory. In practice:

- n = 30 is a common rule of thumb, but the required n depends on skew.
- For heavy-skewed distributions (exponential, log-normal), you may need n > 100 before the sampling distribution looks Normal.
- For already-Normal populations, the sampling distribution is Normal at any n.

### The bootstrap

When you can't derive the sampling distribution analytically (e.g. for the median, or a complicated statistic), **resample the data with replacement** many times, recompute the statistic each time, and use the empirical distribution as your sampling distribution.

Bootstrap 95% CI (percentile method): the 2.5th and 97.5th percentiles of the resampled statistics.

```r
set.seed(1)
x <- rnorm(50, 10, 3)
boot_means <- replicate(5000, mean(sample(x, replace = TRUE)))
quantile(boot_means, c(0.025, 0.975))
```

### `set.seed()` and reproducibility

Any code that uses `r*` functions or `sample()` involves random numbers. Set a seed at the top of any simulation script:

```r
set.seed(2026)
```

so that you (and readers) get the same numbers back on rerun.

## Practice

Work in `module_02.R` or `module_02.qmd`. Attempt each question fully before opening the solution callout.

---

### Q1 — CLT by simulation

Draw 10,000 samples of size **n = 5**, **n = 30**, and **n = 100** from `rexp(rate = 1)` (a highly skewed exponential distribution, mean = 1, sd = 1). For each n, compute the sample mean and plot the distribution of those 10,000 means. Overlay the theoretical Normal(mean = 1, sd = 1/√n) for comparison.

At what n does the sampling distribution look "Normal enough" for you to be comfortable using a t-test on data this skewed?

> [!tip]- Hint
> `replicate(10000, mean(rexp(n, rate = 1)))` gives you 10,000 sample means. Use `ggplot` with `geom_histogram(aes(y = ..density..))` and `stat_function(fun = dnorm, args = list(mean = 1, sd = 1/sqrt(n)))` to overlay.

> [!success]- Worked solution
> ```r
> library(ggplot2)
> library(dplyr)
> set.seed(2026)
>
> simulate_means <- function(n, reps = 10000, rate = 1) {
>   data.frame(
>     n = n,
>     mean = replicate(reps, mean(rexp(n, rate = rate)))
>   )
> }
>
> sims <- bind_rows(
>   simulate_means(5),
>   simulate_means(30),
>   simulate_means(100)
> )
>
> normal_curves <- do.call(rbind, lapply(c(5, 30, 100), function(n) {
>   x <- seq(max(0, 1 - 4 / sqrt(n)), 1 + 4 / sqrt(n), length.out = 400)
>   data.frame(n = n, x = x,
>              density = dnorm(x, mean = 1, sd = 1 / sqrt(n)))
> }))
>
> ggplot(sims, aes(mean)) +
>   geom_histogram(aes(y = after_stat(density)), bins = 60, alpha = 0.6) +
>   facet_wrap(~ n, scales = "free", labeller = label_both) +
>   geom_line(data = normal_curves, aes(x = x, y = density),
>             inherit.aes = FALSE, colour = "red") +
>   labs(title = "Sampling distribution of the mean, Exp(1)")
> ```
>
> **What you should see**: at n = 5 the sampling distribution is still visibly right-skewed. At n = 30 it looks fairly symmetric but the left tail is still a bit short. At n = 100 you basically can't tell it from the Normal overlay.
>
> **Practical takeaway**: the popular "n > 30 is enough for the CLT" rule is fine for mildly skewed data but *marginal* for heavy skew like exponential. For log-normal or heavier-tailed data, you might want n ≥ 100 or a transformation. This is a great argument for **always plotting your data first**.

---

### Q2 — Standard error, three ways

Let `x <- rnorm(50, mean = 10, sd = 3)`. Estimate the standard error of the mean three different ways and confirm they agree:

1. **Formula**: `sd(x) / sqrt(length(x))`.
2. **Simulation from the known population**: `sd(replicate(10000, mean(rnorm(50, 10, 3))))`.
3. **Bootstrap from your sample**: `sd(replicate(10000, mean(sample(x, replace = TRUE))))`.

Explain in one sentence why (3) works even when you *don't* know the population.

> [!success]- Worked solution
> ```r
> set.seed(2026)
> x <- rnorm(50, 10, 3)
>
> se_formula <- sd(x) / sqrt(length(x))
> se_sim     <- sd(replicate(10000, mean(rnorm(50, 10, 3))))
> se_boot    <- sd(replicate(10000, mean(sample(x, replace = TRUE))))
>
> c(formula = se_formula, sim = se_sim, bootstrap = se_boot)
> ```
> All three should be ≈ 0.42 (= 3/√50). The bootstrap works because **the sample is our best empirical estimate of the population** — resampling from the sample with replacement mimics resampling from the population, so the spread of resampled means approximates the sampling distribution's spread.

---

### Q3 — Bootstrap CI for a median

Load `palmerpenguins::penguins`. Compute a bootstrap 95% percentile CI for the **median** `body_mass_g` of Gentoo penguins using 5,000 resamples. Compare it to a naive normal-approximation CI: `median ± 1.96 × SE_of_median`, where `SE_of_median` also comes from the bootstrap.

Why is the percentile CI generally preferred over the normal-approximation CI for the median?

> [!tip]- Hint
> Filter to Gentoo, drop NAs first. `sample(gentoo_mass, replace = TRUE)` then `median()`.

> [!success]- Worked solution
> ```r
> library(palmerpenguins)
> library(dplyr)
> set.seed(2026)
>
> gentoo <- penguins |>
>   filter(species == "Gentoo", !is.na(body_mass_g)) |>
>   pull(body_mass_g)
>
> boot_medians <- replicate(5000, median(sample(gentoo, replace = TRUE)))
>
> # Percentile CI
> ci_pct <- quantile(boot_medians, c(0.025, 0.975))
>
> # Normal-approximation CI
> se_med <- sd(boot_medians)
> ci_norm <- median(gentoo) + c(-1, 1) * 1.96 * se_med
>
> list(observed_median = median(gentoo),
>      percentile_CI   = ci_pct,
>      normal_CI       = ci_norm)
> ```
>
> **Why percentile is preferable to this naive Normal interval**: the sampling distribution of the median can be skewed or "chunky." The Normal interval assumes symmetry, whereas the percentile interval can reflect asymmetry in the bootstrap distribution. The percentile method is not universally best, though; BCa or bootstrap-*t* intervals often have better coverage, especially in small or strongly skewed samples.

---

### Q4 — Type I error, empirically

Simulate 10,000 two-sample t-tests under the null hypothesis (both groups drawn from the *same* Normal). For each simulation, record the p-value. Then:

1. Report the fraction with p < 0.05. Is it close to 0.05?
2. Plot a histogram of the p-values. What shape should it have and why?
3. Repeat with unequal population variances (Group A: sd = 1, Group B: sd = 4) using **Student's** t-test (`var.equal = TRUE`). What happens?
4. Repeat with unequal variances using **Welch's** t-test (`var.equal = FALSE`, the R default). Does Welch's fix it?

> [!tip]- Hint
> Write a small function that returns one p-value, then `replicate(10000, that_function())`. For (3) and (4), the mean of both groups is still the same — you're testing whether the test is *valid* under heteroscedasticity, not whether it detects a mean difference.

> [!success]- Worked solution
> ```r
> set.seed(2026)
>
> one_pvalue <- function(n1 = 20, n2 = 20, sd1 = 1, sd2 = 1, var.equal = FALSE) {
>   x <- rnorm(n1, mean = 0, sd = sd1)
>   y <- rnorm(n2, mean = 0, sd = sd2)
>   t.test(x, y, var.equal = var.equal)$p.value
> }
>
> # (1) & (2) — null true, equal variances
> p_null <- replicate(10000, one_pvalue())
> mean(p_null < 0.05)          # should be ≈ 0.05
> hist(p_null, breaks = 40)    # should be flat (uniform)
>
> # (3) — unequal variance, Student's t (mis-specified)
> p_student <- replicate(10000, one_pvalue(sd1 = 1, sd2 = 4, var.equal = TRUE))
> mean(p_student < 0.05)       # noticeably ≠ 0.05, especially if n1 ≠ n2
>
> # (4) — unequal variance, Welch's t (default)
> p_welch <- replicate(10000, one_pvalue(sd1 = 1, sd2 = 4, var.equal = FALSE))
> mean(p_welch < 0.05)         # should be ≈ 0.05
> ```
>
> **What you should see**:
> - **Null + equal variances**: Type I error ≈ 0.05 and a roughly flat p-value histogram. For an exact, continuous test under a simple null, p-values are Uniform(0, 1); valid discrete or conservative tests can instead be stochastically larger than Uniform while still controlling Type I error.
> - **Unequal variances + Student's t with equal n**: still roughly OK — Student's t is fairly robust to variance inequality when sample sizes are equal.
> - **Unequal variances + Student's t with unequal n** (try `n1 = 10, n2 = 40`): Type I error can jump to 0.10+ or drop below 0.01 depending on which group is bigger. **This is why R defaults to Welch's.**
> - **Welch's under any of the above**: Type I error stays near 0.05.
>
> **Bigger idea**: this is how you should *always* check a new test or model — simulate under the null, look at p-value calibration. If your Type I error is wrong under a scenario, the test is invalid for that scenario, no matter what it says.

---

### Q5 — Bonus: power by simulation

Simulate the **power** of a two-sample t-test with n = 20 per group and true effect size Cohen's d = 0.5 (i.e., `rnorm(20, 0, 1)` vs `rnorm(20, 0.5, 1)`). What fraction of simulations reject at α = 0.05?

Confirm your answer against `power.t.test(n = 20, delta = 0.5, sd = 1, sig.level = 0.05)`.

> [!success]- Worked solution
> ```r
> set.seed(2026)
> pow_sim <- mean(replicate(10000, {
>   t.test(rnorm(20, 0, 1), rnorm(20, 0.5, 1))$p.value < 0.05
> }))
> pow_sim
> power.t.test(n = 20, delta = 0.5, sd = 1, sig.level = 0.05)$power
> ```
> Both should give ≈ 0.34 — a small study catches a medium effect only about a third of the time. **A prospective power analysis is just this simulation done before you collect data.** Once you can simulate power for any design, you're no longer dependent on `power.t.test()` and its cousins.

---

## Mini-project (deliverable)

Create `module_02_report.qmd`. Include:

1. The three CLT panels from Q1 with your written answer to the "at what n" question.
2. The three SE estimates from Q2 side-by-side in a table.
3. The bootstrap CI vs normal-approximation CI from Q3 with your interpretation.
4. The p-value histogram from Q4 (equal variances) plus a small table of Type I error rates across the four scenarios.
5. A one-paragraph reflection: "The single most useful thing I learned this module was …"

## Reflection

**What I learned**:

**What still confuses me** (link forward with `[[...]]`):

**Time spent**:

---

Previous: [[01 - Module 1 - R Foundations]]  ·  Next: [[03 - Module 3 - Classical Inference]] *(western blot ANOVA case study)*

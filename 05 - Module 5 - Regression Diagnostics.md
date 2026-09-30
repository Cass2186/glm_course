---
title: Module 5 — Regression diagnostics and assumption checks
tags: [stats, R, module-5, diagnostics, residuals, VIF, transformations]
module: 5
created: 2026-07-08
---

# Module 5 — Regression diagnostics and assumption checks

> [!info] The unspoken rule
> Everyone shows their p-values. Almost nobody shows their diagnostics. That's why so much published regression is unreliable. If you check the four base R plots for every fit and know what "good" looks like, you'll be in the top 10% of empirical analysts.

## Objectives

By the end of this module you should be able to:

1. Read the four base R diagnostic plots for `lm()` fits and know what each is telling you.
2. Identify **high-leverage** and **high-influence** points using Cook's distance and hat values.
3. Diagnose and address **multicollinearity** with variance inflation factors.
4. Detect **heteroscedasticity** and remedy it with transformations or robust standard errors.
5. Choose sensible transformations for skewed responses.
6. Re-verify the western blot ANOVA from Module 3 with proper diagnostics.

## Concepts

### The four assumptions of linear regression

All matter, but their consequences depend on whether your goal is estimation, prediction, or hypothesis testing:

1. **Independence of errors** — hardest to fix after the fact. Broken by repeated measures, nested designs, time series, spatial data.
2. **Linearity** of the mean function — broken by curved relationships fit with a straight line.
3. **Homoscedasticity** — constant residual variance. Broken by variance that scales with the mean.
4. **Normality of errors** — least critical. The CLT makes large-n inference on β robust to non-normal errors.

### The four base R plots

`plot(fit)` gives you four diagnostics. Learn to read each.

```r
fit <- lm(mpg ~ wt + hp, data = mtcars)
par(mfrow = c(2, 2))
plot(fit)
par(mfrow = c(1, 1))
```

1. **Residuals vs Fitted** — checks linearity and homoscedasticity. A flat, featureless cloud is what you want. A curve implies missing nonlinearity; a funnel implies heteroscedasticity.
2. **Q-Q plot of residuals** — checks normality. Points on the line = normal residuals. Heavy tails = points curving away at both ends.
3. **Scale-Location** — square root of standardized residuals vs fitted. Another homoscedasticity check; a rising trend line means variance grows with fitted value.
4. **Residuals vs Leverage** — identifies influential points. Dashed contours mark Cook's distance = 0.5 and 1. Points outside 0.5 are worth a look; outside 1 are big deals.

### Leverage, residual, influence

Three related but distinct ideas:

- **Leverage** (hat value): how unusual an observation's *predictors* are. Points with extreme X have high leverage.
- **Residual**: how far a point is from the fitted line.
- **Influence** (Cook's distance): combines the two — how much the fit would change if this point were removed.

A point can have high leverage but small influence (it's extreme in X but the fit passes through it). A point can have modest leverage but be influential if its residual is also large.

```r
hatvalues(fit)          # leverage per observation
cooks.distance(fit)     # influence per observation
rstandard(fit)          # standardized residuals
rstudent(fit)           # externally studentized residuals

# Rule-of-thumb thresholds
p <- length(coef(fit)); n <- nrow(model.frame(fit))
high_lev  <- which(hatvalues(fit)   > 2 * p / n)
high_cook <- which(cooks.distance(fit) > 4 / n)
```

### Multicollinearity and the VIF

When predictors are correlated with each other, coefficient estimates become unstable — small changes in the data flip signs, standard errors balloon. The **variance inflation factor** measures how much a coefficient's variance is inflated by collinearity.

```r
car::vif(lm(mpg ~ ., data = mtcars))
```

Rough screening conventions (not decision rules):
- VIF < 5: often little concern from variance inflation alone.
- VIF 5–10: worth thinking about.
- VIF > 10: severe variance inflation is plausible; inspect the design and the auxiliary regressions. Do not automatically drop a needed confounder, because that can change the estimand and introduce omitted-variable bias.

### Heteroscedasticity remedies

If variance grows with the fitted value:

1. **Transform the response** — `log(y)`, `sqrt(y)`, or Box-Cox (`MASS::boxcox(fit)`). Log is the workhorse for anything that spans an order of magnitude.
2. **Use heteroscedasticity-consistent standard errors** — keeps the OLS coefficient estimates and gives asymptotically valid SEs under broad forms of heteroscedasticity; HC3 is often preferred in smaller samples but is not a cure for nonlinearity, dependence, or influential-point problems.
   ```r
   library(sandwich); library(lmtest)
   coeftest(fit, vcov = vcovHC(fit, type = "HC3"))
   ```
3. **Model the variance explicitly** — generalized least squares, `nlme::gls()` with a `varPower` structure. Advanced.

### When to log-transform

Transform `y` if any of these apply:
- `y` is strictly positive and spans an order of magnitude or more.
- Residual variance grows with the fitted value (funnel in residuals-vs-fitted).
- The response is a ratio (protein expression, gene expression, dose–response).
- The QQ plot shows heavy right skew.

After a log transform, coefficients become **multiplicative** — a coefficient of 0.1 for `x` means y increases by a factor of `exp(0.1) ≈ 1.105`, or about 10.5% per unit x.

## Practice

Work in `module_05.R` or `module_05.qmd`. Load the western blot data at the top for Q5.

---

### Q1 — Read the four plots

Fit `lm(mpg ~ wt + hp, data = mtcars)`. Run `plot(fit)` and write **one sentence per panel** describing what you see and what it means for the model's validity.

> [!success]- Worked solution
> ```r
> fit <- lm(mpg ~ wt + hp, data = mtcars)
> par(mfrow = c(2, 2)); plot(fit); par(mfrow = c(1, 1))
> ```
> **Typical readings**:
> 1. **Residuals vs Fitted**: mild curvature — the LOESS line dips at high fitted values, suggesting some nonlinearity in the wt or hp effect. Not severe.
> 2. **Q-Q plot**: mostly on the line with a couple of high-end outliers (Toyota Corolla, Fiat 128). Residuals are approximately normal.
> 3. **Scale-Location**: slight upward trend — some hint of heteroscedasticity but nothing dramatic.
> 4. **Residuals vs Leverage**: Chrysler Imperial has both high leverage and non-trivial residual — worth checking whether excluding it changes conclusions.

---

### Q2 — Hunt down influential points

For the same fit:

1. Compute Cook's distances and identify observations with Cook's D > 4/n.
2. Refit the model without those points. How much do the coefficients change?
3. Should you drop them for the final analysis? Frame your answer in terms of *why* they're influential.

> [!success]- Worked solution
> ```r
> n <- nrow(mtcars)
> cd <- cooks.distance(fit)
> flagged <- which(cd > 4 / n)
> mtcars[flagged, ]
>
> fit_trim <- lm(mpg ~ wt + hp, data = mtcars[-flagged, ])
> broom::tidy(fit)
> broom::tidy(fit_trim)
> ```
> **Interpretation guidance**: the flagged cars are typically the Corolla (very high mpg for its weight) and Chrysler Imperial (heavy and gas-guzzling). Coefficients shift some — the `wt` slope typically steepens.
>
> **Never** drop points just to improve p-values. Drop only if you have a substantive reason (measurement error, wrong population, coding error). Otherwise report both — with and without the influential points — as a sensitivity analysis.

---

### Q3 — VIF exercise

Fit `lm(mpg ~ ., data = mtcars)` (all predictors). Run `car::vif()`. Any values above 10?

1. Identify the offenders. Which pairs of predictors are collinear?
2. Drop one at a time and re-check VIFs until all are below 5.
3. How do the surviving coefficients change as you simplify?

> [!success]- Worked solution
> ```r
> library(car)
> full <- lm(mpg ~ ., data = mtcars)
> vif(full)
> ```
> **Result for `mtcars`**: `disp`, `cyl`, `wt`, and `hp` are strongly related. In the full model, `disp` (≈ 21.6), `cyl` (≈ 15.4), and `wt` (≈ 15.2) have VIFs above 10; `hp` is just below 10.
>
> ```r
> # Iterate: drop highest VIF, refit, recheck
> step1 <- update(full, . ~ . - disp)
> vif(step1)
> step2 <- update(step1, . ~ . - cyl)
> vif(step2)
> step3 <- update(step2, . ~ . - wt)
> vif(step3)  # hp remains just above 5
> step4 <- update(step3, . ~ . - hp)
> vif(step4)  # all below 5 for this mechanical exercise
> ```
> **Lesson**: this mechanical sequence answers the exercise but is not a model-building recipe. VIF surgery is not automatic: retain variables required by the scientific estimand, and recognize that dropping a confounder can bias other coefficients. If variables are interchangeable surrogates for the same construct, prespecify the one that best matches the question.

---

### Q4 — When to log-transform

Fit `lm(price ~ carat, data = ggplot2::diamonds)`.

1. Run `plot(fit)`. Describe the residuals vs fitted plot — is variance constant?
2. Refit as `lm(log(price) ~ log(carat), data = diamonds)`. Re-check the diagnostics. Are they cleaner?
3. Interpret the slope of the log-log model in words. Hint: it's an **elasticity** — a percentage change in y per percentage change in x.

> [!success]- Worked solution
> ```r
> library(ggplot2)
> fit1 <- lm(price ~ carat, data = diamonds)
> plot(fit1, which = 1)   # obvious fan/funnel
>
> fit2 <- lm(log(price) ~ log(carat), data = diamonds)
> plot(fit2, which = 1)   # much more even
> summary(fit2)
> ```
> **Interpretation**: the slope on log(carat) is about 1.68. Meaning: a 1% increase in carat is associated with roughly a 1.68% increase in price. This is the workhorse interpretation for log-log regressions.
>
> **Why the transform helped**: price grows faster than linearly with carat, and variance grows with mean. Log-log linearizes the relationship and stabilizes the variance in one move.

---

### Q5 — Western blot diagnostics revisit

Load and re-run the Module 3 fit:

```r
library(dplyr)
blot <- read.csv("data/western_blot_example.csv") |>
  mutate(Ratio = Target_Intensity / Loading_Control_Intensity)
blot_bio <- blot |>
  group_by(Genotype, Bio_Rep) |>
  summarise(Ratio = mean(Ratio), .groups = "drop") |>
  mutate(Genotype = factor(Genotype, levels = c("WT", "HET", "KO", "RESCUE")))

fit_raw <- lm(Ratio ~ Genotype, data = blot_bio)
fit_log <- lm(log(Ratio) ~ Genotype, data = blot_bio)
```

1. Run `plot(fit_raw)` and `plot(fit_log)`. Which set of diagnostics is cleaner?
2. Run `shapiro.test(residuals(fit_raw))` and the same for `fit_log`. Compare p-values.
3. Run `car::leveneTest(Ratio ~ Genotype, data = blot_bio)` and the log version. Compare.
4. If the log-scale model is cleaner, is your KO-vs-WT conclusion from Module 3 still valid? Refit and check the Dunnett p-values on the log scale.

> [!success]- Worked solution
> ```r
> par(mfrow = c(2, 2)); plot(fit_raw); par(mfrow = c(1, 1))
> par(mfrow = c(2, 2)); plot(fit_log); par(mfrow = c(1, 1))
>
> shapiro.test(residuals(fit_raw))
> shapiro.test(residuals(fit_log))
> car::leveneTest(Ratio ~ Genotype, data = blot_bio)
> car::leveneTest(log(Ratio) ~ Genotype, data = blot_bio)
>
> library(multcomp)
> summary(glht(aov(log(Ratio) ~ Genotype, data = blot_bio),
>              linfct = mcp(Genotype = "Dunnett")))
> ```
> **What you should see**: with n = 4 per group, both models look reasonable — Shapiro-Wilk fails to reject normality for both (raw p ≈ 0.77; log p ≈ 0.90). These low-powered tests do not prove normality, so keep the plots central. Both scales agree on the KO-vs-WT conclusion; on the log scale the estimated KO/WT geometric-mean ratio is `exp(-0.84) ≈ 0.43`.

---

### Q6 — Robust standard errors as an alternative

Sometimes you can't or don't want to transform. Robust SEs let you keep the model on the natural scale while fixing the inference under heteroscedasticity.

Using `lm(price ~ carat, data = diamonds)`:

1. Extract the naïve standard error for `carat`.
2. Compute HC3 robust standard errors with `sandwich::vcovHC()` and `lmtest::coeftest()`.
3. How different are the two?

> [!success]- Worked solution
> ```r
> library(sandwich); library(lmtest)
> fit <- lm(price ~ carat, data = diamonds)
> summary(fit)$coef
> coeftest(fit, vcov = vcovHC(fit, type = "HC3"))
> ```
> **What you should see**: the coefficient estimates are identical (robust SEs don't change point estimates). The SE for `carat` gets much larger under HC3 — the naïve SE was over-optimistic because it assumed constant variance. The robust p-value is still tiny here because the effect is huge, but the SE inflation would matter for a subtler effect.

---

## Mini-project (deliverable)

Create `module_05_report.qmd`. Take **your Module 4 mini-project model** (the best penguins body-mass model) and do a full diagnostic write-up:

1. The four base R plots with a one-sentence read of each.
2. Cook's distance flagged points (if any) and a sensitivity refit without them.
3. VIF table.
4. Shapiro-Wilk test on residuals and one comment on whether normality matters for your sample size.
5. Formal test for heteroscedasticity: `car::ncvTest(fit)`.
6. One paragraph: "If I had to defend this model to a reviewer, here's what I'd say about assumptions."

## Reflection

**What I learned**:

**What still confuses me** (link forward with wikilinks):

**Time spent**:

---

Previous: [[04 - Module 4 - Linear Regression]]  ·  Next: [[06 - Module 6 - GLM Framework]]

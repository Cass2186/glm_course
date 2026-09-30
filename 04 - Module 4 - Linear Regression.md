---
title: Module 4 — Linear regression
tags: [stats, R, module-4, lm, regression, interactions]
module: 4
created: 2026-07-08
---

# Module 4 — Linear regression

> [!info] Why this module matters
> The linear model is the atom of everything that follows. ANOVA from [[03 - Module 3 - Classical Inference]] was already a linear model with a categorical predictor. Every GLM in Modules 6–8 is a linear model with a link function bolted on. Mixed models are linear models with extra variance components. Get comfortable here and the rest is variations on a theme.

## Objectives

By the end of this module you should be able to:

1. Fit and interpret a simple linear regression with `lm()`.
2. Interpret multiple regression coefficients as *conditional* effects — adjusted for the other predictors.
3. Read what R's model matrix does with categorical predictors, and set your own reference level and contrasts.
4. Fit and interpret models with **interactions** — including two continuous, two categorical, and mixed.
5. Get predictions with confidence and prediction intervals from `predict()`.
6. Compare nested models with `anova()` and non-nested models with `AIC`.

## Concepts

### The equation

A linear model claims each outcome is a straight-line combination of predictors plus noise:

$$y_i = \beta_0 + \beta_1 x_{1i} + \beta_2 x_{2i} + \dots + \varepsilon_i, \quad \varepsilon_i \sim \mathrm{Normal}(0, \sigma^2)$$

R fits this by least squares. The result gives you point estimates for each β, standard errors, t-tests against zero, and confidence intervals.

```r
fit <- lm(mpg ~ wt, data = mtcars)
summary(fit)
confint(fit)
```

### Interpreting coefficients

For **continuous** predictors, the coefficient is the change in y per **one-unit increase** in x, holding all other predictors fixed.

For **categorical (factor)** predictors, R creates dummy variables. With WT as the reference level, the coefficient for `GenotypeKO` is the mean of KO minus the mean of WT — you already saw this in Module 3.

**Adjustment / partial association**: in multiple regression, each slope answers "how much does mean y change per unit of this x, *holding the other included predictors fixed*?" This can differ sharply from the marginal (simple) slope. A change may reflect confounding, mediation, collinearity, or selection; the regression alone does not identify which. [[01 - Module 1 - R Foundations]] Q2 showed the descriptive aggregation pattern in the penguins.

### The model matrix

Behind the scenes, `lm()` builds a design matrix X. You can inspect it:

```r
model.matrix(~ Genotype, data = blot_bio) |> head()
```

Understanding this matrix explains everything about coefficients: an intercept column of 1s, one dummy column per non-reference factor level, continuous columns as-is. When you write `y ~ x * z`, R adds a column for `x`, a column for `z`, and a column for `x * z`.

### Setting reference levels and contrasts

R defaults to **treatment contrasts** — the first level of a factor is the reference, and every other level is compared to it. This is almost always what you want for wet-lab data (WT is the reference; other genotypes are differences from WT).

```r
blot_bio$Genotype <- factor(blot_bio$Genotype,
                            levels = c("WT", "HET", "KO", "RESCUE"))
contrasts(blot_bio$Genotype)   # inspect the coding
```

Other contrasts exist (sum, Helmert, polynomial) for specific designs. Skip them unless you have a reason.

### Interactions

An interaction says the effect of one predictor depends on the level (or value) of another.

```r
# Two continuous
lm(mpg ~ wt * hp, data = mtcars)

# Continuous by categorical (per-species slopes)
lm(body_mass_g ~ flipper_length_mm * species, data = penguins)

# Two categorical (all six supplement × dose cells modeled)
lm(len ~ supp * factor(dose), data = ToothGrowth)
```

With an interaction present, a treatment-coded main-effect coefficient is conditional: it is the effect when the other continuous predictor equals zero or when the other factor is at its reference level. Center continuous predictors or choose meaningful reference levels, then report relevant simple effects or marginal contrasts. Do not decide whether to describe an interaction from its p-value alone; design and subject-matter relevance matter too.

### Predictions

```r
fit <- lm(body_mass_g ~ flipper_length_mm + species, data = penguins)

new_data <- data.frame(
  flipper_length_mm = c(190, 210, 220),
  species = c("Adelie", "Gentoo", "Gentoo")
)

# Confidence interval for the mean at these X's
predict(fit, newdata = new_data, interval = "confidence")

# Prediction interval for a NEW observation (wider — includes residual variance)
predict(fit, newdata = new_data, interval = "prediction")
```

**Difference to internalize**: the confidence interval covers the *mean* of y at those X's. The prediction interval covers a *single future observation* at those X's. The prediction interval is always wider.

### Comparing models

- **Nested Gaussian linear models fitted to the same observations** (one model is a special case of another): `anova(fit_small, fit_big)` runs an F-test.
- **Non-nested**: use AIC — `AIC(fit1, fit2)`. Lower is better. A rule of thumb: differences under 2 are not compelling; over 10 are strong evidence.

When variables have missing values, create one complete-case analysis dataset before comparing models; otherwise models may be fitted to different rows. AIC ranks predictive information loss only within the candidate set and is not a test of causal validity.

## Practice

Work in `module_04.R` or `module_04.qmd`. Attempt each fully before opening the solution.

---

### Q1 — Simple linear regression, interpreted in real units

Fit `lm(mpg ~ wt, data = mtcars)`.

1. Report the slope, its standard error, and its 95% CI. Interpret the slope in one sentence in the actual units.
2. Fit `lm(mpg ~ hp, data = mtcars)` — same summary. Which predictor gives a lower residual standard error?
3. In one sentence each, what does the intercept mean for each model? Is that interpretation practically meaningful?

> [!success]- Worked solution
> ```r
> fit_wt <- lm(mpg ~ wt, data = mtcars)
> summary(fit_wt); confint(fit_wt)
>
> fit_hp <- lm(mpg ~ hp, data = mtcars)
> summary(fit_hp); confint(fit_hp)
> ```
> **Interpretation**:
> - `wt` slope ≈ −5.34 mpg per 1000 lb of extra weight; each additional 1000 lb costs about 5.3 mpg. Residual SE ≈ 3.05.
> - `hp` slope ≈ −0.068 mpg per additional horsepower. Residual SE ≈ 3.86.
> - `wt` explains a bit more variance (its residual SE is lower).
> - The intercept for `mpg ~ wt` is ~37 mpg — extrapolated to weight = 0, which is nonsensical. Interpretive rule: intercepts are only meaningful when x = 0 is in the data range.

---

### Q2 — Adding a predictor changes the story

Fit `lm(mpg ~ wt, data = mtcars)` then `lm(mpg ~ wt + hp, data = mtcars)`. Compare the coefficient on `wt` between the two models.

1. Does it change? By how much?
2. Explain in one paragraph why adding `hp` moves the `wt` slope.
3. Which slope is the "right" one to report if a reviewer asks about the effect of weight?

> [!success]- Worked solution
> ```r
> summary(lm(mpg ~ wt, data = mtcars))
> summary(lm(mpg ~ wt + hp, data = mtcars))
> ```
> **What you should see**: the `wt` coefficient shrinks (in absolute value) from about −5.34 to about −3.88 when `hp` is added.
>
> **Why**: heavier cars also tend to have more horsepower. When `hp` is out of the model, the `wt` slope is picking up part of the horsepower effect too. Adding `hp` isolates weight's effect **holding horsepower constant**.
>
> **Which to report**: it depends on the estimand. The simple slope describes the marginal association in these cars. The multiple-regression slope describes the association with weight among cars with the same horsepower. Neither is automatically causal; a causal interpretation needs a defensible design and an adjustment set chosen from subject-matter knowledge, not merely every available predictor.

---

### Q3 — Categorical predictor, revisited from a linear-model angle

Use `palmerpenguins::penguins`. Fit `lm(body_mass_g ~ species, data = penguins)` and read the coefficients.

1. What is the intercept? (Which species?)
2. Interpret each of the other two coefficients as group-mean differences.
3. Confirm with `aggregate(body_mass_g ~ species, data = penguins, FUN = mean, na.rm = TRUE)`.
4. Now fit `lm(body_mass_g ~ flipper_length_mm + species, data = penguins)`. How do the species coefficients change, and what does that tell you biologically?

> [!success]- Worked solution
> ```r
> library(palmerpenguins)
>
> fit_sp   <- lm(body_mass_g ~ species, data = penguins)
> fit_flsp <- lm(body_mass_g ~ flipper_length_mm + species, data = penguins)
>
> summary(fit_sp)
> summary(fit_flsp)
>
> aggregate(body_mass_g ~ species, data = penguins,
>           FUN = function(x) mean(x, na.rm = TRUE))
> ```
> **What you should see**: in `fit_sp`, the intercept is mean(Adelie) ≈ 3700 g, `speciesChinstrap` ≈ 32 (nearly identical to Adelie), `speciesGentoo` ≈ 1375 (Gentoos are much heavier). This matches the `aggregate()` table.
>
> **After adjusting for flipper length**: the species coefficients change dramatically: at a common flipper length, Chinstrap is estimated about 207 g lighter and Gentoo about 267 g heavier than Adelie. This is a conditional association, not evidence that flipper length causally explains the marginal species differences; flipper length is itself a body-size characteristic and the species' ranges overlap imperfectly.

---

### Q4 — Continuous by categorical interaction

Fit `lm(body_mass_g ~ flipper_length_mm * species, data = penguins)`. Then:

1. Read the coefficients. What does `flipper_length_mm:speciesGentoo` mean in plain English?
2. Compute the slope of body_mass_g on flipper_length_mm **for Gentoo** by hand from the coefficients.
3. Visualize with `ggplot(..., aes(flipper_length_mm, body_mass_g, colour = species)) + geom_point() + geom_smooth(method = "lm", se = FALSE)`. Do the fitted slopes match your hand calculation?
4. Test the interaction against the additive model: `anova(fit_add, fit_int)`. Is it worth keeping?

> [!success]- Worked solution
> ```r
> fit_add <- lm(body_mass_g ~ flipper_length_mm + species, data = penguins)
> fit_int <- lm(body_mass_g ~ flipper_length_mm * species, data = penguins)
> summary(fit_int)
> anova(fit_add, fit_int)
> ```
> **Interpretation**: `flipper_length_mm:speciesGentoo` is the **difference in slope** between Gentoo and Adelie. If the Adelie slope is ~32 g/mm and the interaction coefficient is +23, the Gentoo slope is ~55 g/mm.
>
> **F-test**: with the current `palmerpenguins` data, the two interaction terms jointly improve fit (F ≈ 5.53, p ≈ 0.0043). The estimated Adelie slope is about 32.8 g/mm and the Gentoo slope about 54.6 g/mm. Subject-matter relevance and out-of-sample performance still matter; the p-value is not the whole model-selection argument.

---

### Q5 — Predictions with intervals

Using `fit_int` from Q4, predict body mass for:
- A Chinstrap with flipper length 200 mm.
- A Gentoo with flipper length 220 mm.

Return both a **confidence** interval and a **prediction** interval. Explain in one sentence what the difference between them means for a reader.

> [!success]- Worked solution
> ```r
> new_data <- data.frame(
>   flipper_length_mm = c(200, 220),
>   species = c("Chinstrap", "Gentoo")
> )
>
> predict(fit_int, newdata = new_data, interval = "confidence")
> predict(fit_int, newdata = new_data, interval = "prediction")
> ```
> **Difference**: the confidence interval is "our uncertainty about the average body mass for penguins with these characteristics." The prediction interval is "our uncertainty about the body mass of one specific penguin with these characteristics" — always wider because it adds residual variance to the mean's uncertainty.

---

### Q6 — Model comparison

Fit the following four models on `mtcars`:

1. `lm(mpg ~ wt)`
2. `lm(mpg ~ wt + hp)`
3. `lm(mpg ~ wt + hp + cyl)`
4. `lm(mpg ~ wt * hp + cyl)`

Compare them with `anova()` (for nested pairs) and `AIC()` (all four at once). Which model would you report and why?

> [!tip]- Hint
> `AIC()` accepts multiple fits at once: `AIC(fit1, fit2, fit3, fit4)`.

> [!success]- Worked solution
> ```r
> f1 <- lm(mpg ~ wt, data = mtcars)
> f2 <- lm(mpg ~ wt + hp, data = mtcars)
> f3 <- lm(mpg ~ wt + hp + cyl, data = mtcars)
> f4 <- lm(mpg ~ wt * hp + cyl, data = mtcars)
>
> anova(f1, f2)   # is adding hp worth it?
> anova(f2, f3)   # is adding cyl on top worth it?
> anova(f3, f4)   # is the wt:hp interaction worth it?
>
> AIC(f1, f2, f3, f4)
> ```
> **Result for `mtcars`**: f2 improves substantially on f1 (p ≈ 0.0015). Adding `cyl` in f3 is weak by the partial F-test (p ≈ 0.098). Adding `wt:hp` in f4 improves on f3 (p ≈ 0.0032), and f4 has the lowest AIC by about 8.5 points relative to f3.
>
> **What to report**: among these prespecified candidates, the evidence favors f4; difficulty of explanation is not a reason to discard a supported interaction. With only 32 observations, treat this in-sample selection as exploratory and validate predictive claims on new data. If a simpler model is required for a specific estimand, state that rationale explicitly.

---

## Mini-project (deliverable)

Create `module_04_report.qmd`. Build the "best" model of `body_mass_g` in `penguins` using `flipper_length_mm`, `bill_length_mm`, `species`, `sex`, and any interactions you think are justified. Include:

1. Your model-building sequence: first create one complete-case dataset for every candidate variable, then start simple, add scientifically justified predictors, and use `anova()` for nested fits on those same rows.
2. Coefficient table (`broom::tidy(fit, conf.int = TRUE)`) with one-sentence interpretation of every non-intercept coefficient.
3. A plot showing observed vs fitted values.
4. Predictions with 95% intervals for three hypothetical penguins of your choice.
5. One paragraph reflecting on which coefficient surprised you most and why.

## Reflection

**What I learned**:

**What still confuses me** (link forward with wikilinks):

**Time spent**:

---

Previous: [[03 - Module 3 - Classical Inference]]  ·  Next: [[05 - Module 5 - Regression Diagnostics]]

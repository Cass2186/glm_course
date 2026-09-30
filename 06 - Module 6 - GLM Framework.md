---
title: Module 6 — Generalized linear models (the framework)
tags: [stats, R, module-6, GLM, link-function, deviance]
module: 6
created: 2026-07-08
---

# Module 6 — Generalized linear models: the framework

> [!info] The pivot
> Most regression modeling so far used a continuous outcome with Gaussian errors. Many biological outcomes are instead binary (dead/alive, responder/non-responder), counts (colonies, spikes, events per hour), or grouped proportions (fraction germinated out of 20 seeds). GLMs combine a response distribution and a **link function** with the linear predictor to handle these mean–variance relationships.

## Objectives

By the end of this module you should be able to:

1. Name the three components of a GLM: linear predictor, link function, variance function.
2. Match outcome types to appropriate GLM families.
3. Fit any GLM with `glm(formula, family = ...)` and read the output.
4. Interpret coefficients on the **link scale** vs the **response scale**.
5. Test nested GLMs with `anova(..., test = "LRT")` (or "Chisq").
6. Read null vs residual deviance and distinguish named pseudo-R² measures.

## Concepts

### The three components

A GLM has three pieces:

1. **Linear predictor** η = β₀ + β₁x₁ + β₂x₂ + ... — same as ordinary regression.
2. **Link function** g(·) — connects the mean of y to the linear predictor: g(E[y]) = η.
3. **Variance function** — describes how var(y) depends on E[y]. Usually determined by the family.

So for a linear model, the identity link g(μ) = μ means E[y] = η directly, and constant variance is the default. For logistic regression, g(μ) = log(μ / (1 − μ)) (the logit link) means E[y] = 1 / (1 + exp(−η)), and variance is μ(1 − μ).

### Common families and their links

| Outcome type                       | Family              | Default link       | Variance                |
| ---------------------------------- | ------------------- | ------------------ | ----------------------- |
| Continuous, Normal errors          | `gaussian`          | identity           | constant σ²             |
| Binary (0/1) or grouped proportion | `binomial`          | logit              | Bernoulli: μ(1 − μ); proportion from n trials: μ(1 − μ)/n |
| Count (unbounded above)            | `poisson`           | log                | μ                       |
| Count with overdispersion          | `quasipoisson`      | log                | φ · μ                   |
| Overdispersed count (NB)           | `MASS::negative.binomial` or `MASS::glm.nb` | log | μ + μ²/θ |
| Strictly positive continuous, right-skewed | `Gamma`     | inverse (or log)   | μ² / ν                  |
| Strictly positive continuous, very right-skewed | `inverse.gaussian` | 1/μ² | proportional to μ³ |

### The `glm()` syntax

```r
glm(formula, family = family_name(link = "link_name"), data = df)
```

Examples:

```r
glm(mpg ~ wt + hp, family = gaussian(), data = mtcars)          # equivalent to lm()
glm(default ~ balance, family = binomial(), data = ISLR2::Default) # logistic
glm(Days ~ Age + Sex, family = poisson(), data = MASS::quine)   # Poisson
glm(y ~ x, family = Gamma(link = "log"), data = df)             # Gamma with log link
```

### Coefficients on link vs response scale

This is the concept that trips everyone up.

- **On the link scale**: coefficients are additive. For a Poisson with log link, `coef(fit)["treatment"]` is the change in log(mean) per one-unit change in treatment.
- **On the response scale**: coefficients are multiplicative. `exp(coef(fit)["treatment"])` is the *ratio* of means for treatment vs reference.

```r
fit <- glm(Days ~ Sex, family = poisson(), data = MASS::quine)
coef(fit)                # link scale (log-count)
exp(coef(fit))           # response scale — the ratio
```

So a Poisson coefficient of 0.4 means expected count is `exp(0.4) ≈ 1.49` times higher — a 49% increase, not a 40% increase.

### Deviance: the GLM analogue of SSR

For linear models, we compared models with F-tests on residual sum of squares. For GLMs, the corresponding quantity is **deviance** — twice the log-likelihood ratio between the fitted model and a saturated model that fits every data point perfectly.

- **Null deviance**: fit of the intercept-only model.
- **Residual deviance**: fit of your model.
- **Difference**: how much better your model does than the null.

`summary(fit)` prints both. Their reduction measures improvement over the intercept-only model, but it is not ordinary R². Several inequivalent pseudo-R² measures exist and must be named explicitly.

### Testing nested models

For nested GLMs, use likelihood ratio tests:

```r
fit0 <- glm(y ~ x1,      family = binomial(), data = df)
fit1 <- glm(y ~ x1 + x2, family = binomial(), data = df)
anova(fit0, fit1, test = "LRT")   # or "Chisq" — same thing here
```

For non-nested, use AIC — same as for linear models.

### Coefficient significance in GLM summaries

`summary(glm_fit)` shows a "z value" and `Pr(>|z|)` column (instead of t values). These come from the **Wald test** — coefficient divided by its SE, compared to a Normal. Wald tests are OK for large samples but can be unreliable in small samples or with complete separation ([[07 - Module 7 - Logistic Regression]]). Prefer LRTs when in doubt.

### GLM residuals

Three types:

- **Pearson residuals** — the natural generalization of standardized residuals.
- **Deviance residuals** — contributions to the residual deviance. These are what `plot(fit)` uses.
- **Response residuals** — just y minus fitted, on the original scale. Rarely useful.

For many fitted GLMs and mixed models, `DHARMa` provides useful simulation-based residual diagnostics and interpretable uniform QQ plots. No single diagnostic replaces checks tailored to the outcome, design, dispersion, and dependence structure.

## Practice

Work in `module_06.R` or `module_06.qmd`.

---

### Q1 — GLM reduces to `lm()` for Gaussian identity

Fit `lm(mpg ~ wt, data = mtcars)` and `glm(mpg ~ wt, family = gaussian(), data = mtcars)`. Confirm coefficients, standard errors, and predictions all match. Then explain in one paragraph why they must match given the GLM framework.

> [!success]- Worked solution
> ```r
> lm_fit  <- lm(mpg ~ wt, data = mtcars)
> glm_fit <- glm(mpg ~ wt, family = gaussian(), data = mtcars)
>
> coef(lm_fit); coef(glm_fit)
> vcov(lm_fit); vcov(glm_fit)
> predict(lm_fit, newdata = data.frame(wt = 3))
> predict(glm_fit, newdata = data.frame(wt = 3))
> ```
> **Why they match**: with the identity link, g(μ) = μ, so the linear predictor **is** the mean of y. With Gaussian errors, maximizing the likelihood over the coefficients is equivalent to minimizing the sum of squared residuals. Least squares and maximum-likelihood coefficient estimates coincide. This is the sanity check that the GLM machinery reduces to the familiar case when it should.

---

### Q2 — Link scale vs response scale

Fit `glm(Days ~ Sex + Age, family = poisson(), data = MASS::quine)`.

1. Report the coefficient for `SexM` (male) on the link scale.
2. Convert to the response scale with `exp()`. What does it mean in "days absent" language?
3. Predict the expected number of days absent for a Male in Age = "F0" and a Female in Age = "F0". Do both by hand from coefficients, then confirm with `predict(fit, ..., type = "response")`.

> [!success]- Worked solution
> ```r
> library(MASS)
> fit <- glm(Days ~ Sex + Age, family = poisson(), data = quine)
> coef(fit)
> exp(coef(fit))
>
> # Predict F0 Male
> newd <- data.frame(Sex = c("M", "F"), Age = c("F0", "F0"))
> predict(fit, newdata = newd, type = "response")
> ```
> **Interpretation**: with the current `MASS::quine` data, `SexM` is about 0.105. `exp(0.105) ≈ 1.11`, so the fitted Poisson mean for male students is about **11% higher** than for female students, holding age category constant. This Poisson fit is badly overdispersed, so its standard errors and p-values are not trustworthy; Module 8 refits it with a negative binomial model.

---

### Q3 — Nested model comparison with LRT

Using the `quine` data:

1. Fit `fit_a <- glm(Days ~ Age, family = poisson(), data = quine)`.
2. Fit `fit_ab <- glm(Days ~ Age + Sex, family = poisson(), data = quine)`.
3. Test whether adding `Sex` improves the fit with `anova(fit_a, fit_ab, test = "LRT")`.
4. Also compare with `AIC(fit_a, fit_ab)`. Do they agree on which model is better?

> [!success]- Worked solution
> ```r
> fit_a  <- glm(Days ~ Age, family = poisson(), data = quine)
> fit_ab <- glm(Days ~ Age + Sex, family = poisson(), data = quine)
>
> anova(fit_a, fit_ab, test = "LRT")
> AIC(fit_a, fit_ab)
> ```
> **What you should see**: within the Poisson model, adding `Sex` reduces deviance by about 6.27 (1 df; p ≈ 0.012), and AIC drops by about 4.27. That is moderate rather than overwhelming evidence. Because `quine` is severely overdispersed, this Poisson LRT is anti-conservative; treat the exercise as mechanics and use Module 8's negative binomial analysis for inference.

---

### Q4 — Family selection: what do you fit?

For each outcome below, name (i) the GLM family, (ii) the appropriate default link function, and (iii) the R code you would use to fit the model with predictor `x`:

1. Did a patient respond to treatment? (yes/no)
2. Number of seizures per patient per week.
3. An uncensored, strictly positive waiting-time outcome with a right-skewed distribution.
4. Number of successes out of 20 attempts per subject.
5. Proportion of a plate covered by bacterial growth (continuous and strictly between 0 and 1).

> [!success]- Worked solution
> 1. `binomial(link = "logit")`. `glm(responded ~ x, family = binomial(), data = df)`.
> 2. `poisson(link = "log")`. `glm(seizures ~ x, family = poisson(), data = df)`. Check for overdispersion afterwards — clinical count data is often overdispersed.
> 3. `Gamma(link = "log")`. `glm(time ~ x, family = Gamma(link = "log"), data = df)`. If observations can be censored, this is a survival-analysis problem instead (for example, a Cox or parametric survival model).
> 4. Grouped binomial. `glm(cbind(success, failure) ~ x, family = binomial(), data = df)`.
> 5. Beta regression: `betareg::betareg(pct ~ x, data = df)`. Ordinary beta regression excludes exact 0 and 1; data containing boundary values need a model that handles them (such as a zero/one-inflated beta model) or a design-appropriate alternative. A logit-transformed `lm` also cannot accept exact boundaries and changes the error model.

---

### Q5 — Simulate a Poisson dataset and recover the parameters

Simulate n = 500 observations from a Poisson GLM with log link, intercept β₀ = 1, slope β₁ = 0.3 on a predictor x from Uniform(0, 5).

1. Draw x and simulate y with `rpois(n, lambda = exp(1 + 0.3 * x))`.
2. Fit a Poisson GLM to your simulated data.
3. Check whether the estimated coefficients are within roughly 2 standard errors of the truth. This should happen often, not on every random sample.
4. Repeat the whole thing 1000 times and check that the estimated coefficients are unbiased on average.

> [!success]- Worked solution
> ```r
> set.seed(2026)
>
> simulate_once <- function(n = 500, b0 = 1, b1 = 0.3) {
>   x <- runif(n, 0, 5)
>   y <- rpois(n, lambda = exp(b0 + b1 * x))
>   fit <- glm(y ~ x, family = poisson())
>   coef(fit)
> }
>
> # Single fit
> simulate_once()
>
> # 1000 repetitions
> reps <- replicate(1000, simulate_once())
> rowMeans(reps)             # should be close to (1, 0.3)
> apply(reps, 1, sd)         # empirical SEs
> ```
> **Lesson**: this is how you should sanity-check any statistical procedure — simulate under the truth you specified, refit, and see if you get it back. The same pattern verifies power calculations, robustness to assumption violations, and calibration under the null.

---

### Q6 — Deviance explained and McFadden's pseudo-R²

For the fit in Q3 (`fit_ab`):

1. Extract null deviance and residual deviance.
2. Compute the deviance-explained measure, D² = 1 − (residual deviance / null deviance).
3. Compute McFadden's pseudo-R² from the fitted and intercept-only log-likelihoods.
4. Explain why neither is directly comparable to a linear regression R².

> [!success]- Worked solution
> ```r
> null_dev <- fit_ab$null.deviance
> res_dev  <- fit_ab$deviance
> d2 <- 1 - res_dev / null_dev
>
> fit_null <- update(fit_ab, . ~ 1)
> mcfadden_r2 <- 1 - as.numeric(logLik(fit_ab) / logLik(fit_null))
>
> c(deviance_explained = d2, McFadden = mcfadden_r2)
> ```
> **Note**: these measures have different definitions and numerical scales; neither is the fraction of outcome variance explained. Report the named measure, and assess calibration and predictive performance separately. For this fit they are about 0.080 (D²) and 0.062 (McFadden), illustrating that they are not interchangeable.

---

## Mini-project (deliverable)

Create `module_06_report.qmd`. Take the `MASS::quine` school absenteeism dataset. Build a Poisson GLM including Age, Sex, Eth (ethnicity), and Lrn (learner status), plus any interactions you can justify.

1. Show the model-building sequence with LRTs.
2. Coefficient table on both the link scale and the exponentiated (response) scale, with one-sentence interpretation of each.
3. Predictions on the response scale for two hypothetical students (your choice).
4. Null deviance, residual deviance, and pseudo-R².
5. A paragraph identifying overdispersion. Use residual deviance/df as a rough screen and a Pearson-residual or simulation-based check as the main diagnostic. If overdispersed, explain what you would do next; [[08 - Module 8 - Count GLMs and Beyond]] handles this properly.

## Reflection

**What I learned**:

**What still confuses me** (link forward with wikilinks):

**Time spent**:

---

Previous: [[05 - Module 5 - Regression Diagnostics]]  ·  Next: [[07 - Module 7 - Logistic Regression]]

---
title: Module 7 — Logistic regression (binary GLMs)
tags: [stats, R, module-7, logistic, odds-ratio, ROC, AUC]
module: 7
created: 2026-07-08
---

# Module 7 — Logistic regression

> [!info] Why this module matters
> Binary outcomes are everywhere in biology and medicine: disease vs no disease, responder vs non-responder, apoptotic vs viable, converted vs not. Logistic regression is the first tool for any of these, and it's what most survival, epidemiology, and ML classification borrows from conceptually.

## Objectives

By the end of this module you should be able to:

1. Fit a logistic regression with `glm(y ~ x, family = binomial())`.
2. Interpret coefficients as **log-odds** on the link scale and **odds ratios** on the response scale.
3. Compute a predicted probability at any input by hand and with `predict(..., type = "response")`.
4. Evaluate classification performance with confusion matrices, ROC curves, and AUC.
5. Choose a decision threshold that reflects the cost of false positives vs false negatives.
6. Fit grouped-binomial (proportion) data using `cbind(success, failure)`.
7. Recognise **separation** and know Firth's correction as a fix.

## Concepts

### The model

For a binary outcome y taking values 0 or 1:

$$\log\left(\frac{p}{1-p}\right) = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \dots$$

where p = P(y = 1). The left side is the **logit** (log-odds); the right side is the linear predictor.

Inverse: p = 1 / (1 + exp(−η)) where η is the linear predictor.

```r
library(ISLR2)
fit <- glm(default ~ balance, family = binomial(), data = Default)
summary(fit)
```

### Interpreting coefficients

On the **log-odds scale**, coefficients are additive but hard to communicate. Almost nobody thinks in log-odds.

On the **odds ratio scale** (exponentiated coefficients), interpretation is multiplicative:

- `exp(β) = 1` — predictor has no effect on odds.
- `exp(β) = 2` — odds double per unit increase in predictor.
- `exp(β) = 0.5` — odds halve per unit increase.

```r
exp(coef(fit))
exp(confint(fit))   # 95% CIs for odds ratios
```

**Key point**: odds are not probabilities. A doubling of odds does not double the probability. If baseline p = 0.1, odds = 1/9, doubling odds → 2/9, so new p ≈ 0.18. If baseline p = 0.5, odds = 1, doubling odds → 2, new p ≈ 0.67. The probability shift depends on where you start.

### Predicted probabilities from coefficients

Given a linear predictor η, the predicted probability is 1 / (1 + exp(−η)).

```r
# By hand
new_eta <- coef(fit)["(Intercept)"] + coef(fit)["balance"] * 1500
1 / (1 + exp(-new_eta))

# With predict()
predict(fit, newdata = data.frame(balance = 1500), type = "response")
```

### Classification and the confusion matrix

Given predicted probabilities and a threshold (default 0.5), each observation becomes a predicted class. The **confusion matrix** shows counts of:

|                 | Predicted 0     | Predicted 1     |
| --------------- | --------------- | --------------- |
| **Actual 0**    | True Negative   | False Positive  |
| **Actual 1**    | False Negative  | True Positive   |

Key metrics:

- **Sensitivity** (true positive rate, recall) = TP / (TP + FN). "Of actual positives, how many did we catch?"
- **Specificity** (true negative rate) = TN / (TN + FP). "Of actual negatives, how many did we correctly not flag?"
- **Precision** (positive predictive value) = TP / (TP + FP). "Of flagged, how many were real?"

### ROC and AUC

The receiver operating characteristic curve plots sensitivity against 1 − specificity as the threshold sweeps from 0 to 1. **AUC** (area under the curve) summarizes overall discrimination: 0.5 = random, 1.0 = perfect.

Labels such as 0.7 = "fair," 0.8 = "good," and 0.9 = "excellent" are field-dependent and should not replace consequences at relevant thresholds or calibration. A very high training AUC should prompt checks for leakage and external validation, but it can also reflect genuinely strong predictors.

```r
library(pROC)
probs <- predict(fit, type = "response")
roc_obj <- roc(Default$default, probs)
auc(roc_obj)
plot(roc_obj)
```

### Choosing a threshold

The 0.5 default is rarely right. Choose based on the relative cost of a false positive vs a false negative:

- Screening for a serious disease with a cheap follow-up test → tolerate many false positives, choose a **low** threshold.
- Recommending an expensive/risky intervention → avoid false positives, choose a **high** threshold.

`pROC::coords()` finds thresholds that meet specific sensitivity or specificity targets.

Estimate AUC, thresholds, and confusion-matrix performance on held-out or cross-validated predictions. Computing them on the same data used to fit the model gives optimistic ("apparent") performance.

### Grouped binomial (proportion data)

When each row is a count of successes out of a known number of trials:

```r
# Simulated germination experiment: 20 seeds per plot
germ <- data.frame(
  treatment = rep(c("control", "hormone"), each = 25),
  n_success = c(rbinom(25, 20, 0.4), rbinom(25, 20, 0.7)),
  n_total   = 20
)
germ$n_failure <- germ$n_total - germ$n_success

# Fit with cbind(success, failure) on the LHS
fit <- glm(cbind(n_success, n_failure) ~ treatment,
           family = binomial(), data = germ)
summary(fit)
```

### Separation

When a predictor perfectly (or near-perfectly) predicts the outcome, maximum likelihood breaks down: coefficients drift toward infinity and standard errors explode. R usually warns with "fitted probabilities numerically 0 or 1 occurred."

One common remedy is **Firth's penalized likelihood** via the `logistf` package. It reduces first-order small-sample bias and usually gives finite estimates under separation. Penalized logistic regression with a prespecified or cross-validated penalty and Bayesian logistic regression with proper priors are other options.

```r
library(logistf)
fit_firth <- logistf(outcome ~ predictor, data = df)
summary(fit_firth)
```

## Practice

Work in `module_07.R` or `module_07.qmd`.

---

### Q1 — Fit and read a logistic regression

Load `ISLR2::Default`. Fit `glm(default ~ balance + income + student, family = binomial(), data = Default)`.

1. Report the coefficient table with `broom::tidy(fit, exponentiate = TRUE, conf.int = TRUE)`.
2. Interpret the `balance` odds ratio in a sentence: "Each additional dollar of balance..."
3. Interpret the `studentYes` odds ratio: does being a student increase or decrease the odds of default?

> [!success]- Worked solution
> ```r
> library(ISLR2); library(broom)
> fit <- glm(default ~ balance + income + student,
>            family = binomial(), data = Default)
> tidy(fit, exponentiate = TRUE, conf.int = TRUE)
> ```
> **Typical readings**:
> - `balance` odds ratio ~ 1.00575. Per dollar of balance, the odds of default multiply by about 1.00575. That's tiny per dollar but compounds — over 1000 additional dollars, odds multiply by `exp(1000 * coef(fit)["balance"]) ≈ 310`.
> - `studentYes` odds ratio around 0.5 after adjusting for balance. **Students have lower odds of default holding balance and income constant**, even though the marginal (unadjusted) association points the other way. This is a classic Simpson's-paradox-in-the-wild.

---

### Q2 — Predicted probability by hand

Using the fit from Q1, compute the predicted probability of default for a **student** with **balance = 2000** and **income = 40000**:

1. By hand from the coefficients (compute η, then apply the inverse logit).
2. With `predict(fit, newdata = ..., type = "response")`.
3. Confirm they match.

> [!success]- Worked solution
> ```r
> b <- coef(fit)
> eta <- b["(Intercept)"] + b["balance"] * 2000 + b["income"] * 40000 + b["studentYes"] * 1
> p_hand <- 1 / (1 + exp(-eta))
> p_hand
>
> predict(fit,
>         newdata = data.frame(balance = 2000, income = 40000, student = "Yes"),
>         type = "response")
> ```
> Both should match to many decimal places.

---

### Q3 — ROC and AUC

Using the fit from Q1:

1. Get predicted probabilities on the training data.
2. Compute AUC with `pROC::auc()`.
3. Plot the ROC curve.
4. Report the sensitivity when specificity is fixed at 0.95 (using `coords()`).

> [!success]- Worked solution
> ```r
> library(pROC)
> probs <- predict(fit, type = "response")
> roc_obj <- roc(Default$default, probs)
> auc(roc_obj)
> plot(roc_obj)
>
> coords(roc_obj, x = 0.95, input = "specificity",
>        ret = c("sensitivity", "specificity"))
> ```
> **Typical result**: apparent (training-data) AUC ≈ 0.950. At 95% specificity, interpolated sensitivity is about 0.715. Treat both as optimistic until they are recomputed from held-out or cross-validated predictions. `pROC` may return `NA` for the threshold at an interpolated ROC point because no single observed cutoff has exactly those coordinates.

---

### Q4 — Threshold selection under asymmetric cost

Suppose a false negative (missed default) costs the bank $10,000, but a false positive (declined creditworthy customer) costs $500 in lost business.

1. Write a function that, given a threshold, returns the expected cost per customer over the whole dataset.
2. Grid-search thresholds from 0.01 to 0.99 in steps of 0.01. Which threshold minimizes expected cost?
3. How different is that from the default 0.5?

> [!success]- Worked solution
> ```r
> actual <- Default$default == "Yes"
>
> expected_cost <- function(t) {
>   pred <- probs > t
>   fn <- sum(actual & !pred)   # missed defaulters
>   fp <- sum(!actual & pred)   # falsely flagged
>   (10000 * fn + 500 * fp) / length(actual)
> }
>
> ts <- seq(0.01, 0.99, by = 0.01)
> costs <- sapply(ts, expected_cost)
> best_t <- ts[which.min(costs)]
> best_t
> ```
> **Result with the supplied data**: the apparent-cost optimum on this 0.01 grid is about 0.05. A missed default costs 20× more than a wrongly declined customer, so the model flags aggressively.
>
> **Bigger lesson**: 0.5 has no special decision-theoretic status. Choose a threshold using explicit costs, prevalence, and adequately calibrated probabilities, then estimate its performance on validation data rather than the training data used here.

---

### Q5 — Grouped binomial (germination experiment)

Simulate a seed-germination experiment: 50 plots total, 20 seeds per plot, half assigned to control (true germination probability 0.4) and half to hormone treatment (true probability 0.7).

1. Simulate the data.
2. Fit a grouped binomial GLM with `cbind(n_success, n_failure) ~ treatment`.
3. Report the odds ratio for `treatment` with a 95% CI. Does it recover the truth?
4. Now fit the same model in **long form** (one row per seed, `outcome ~ treatment`, 1000 rows total). Confirm coefficients match.

> [!success]- Worked solution
> ```r
> set.seed(2026)
> germ <- data.frame(
>   treatment = rep(c("control", "hormone"), each = 25),
>   n_success = c(rbinom(25, 20, 0.4), rbinom(25, 20, 0.7)),
>   n_total   = 20
> )
> germ$n_failure <- germ$n_total - germ$n_success
>
> fit_grouped <- glm(cbind(n_success, n_failure) ~ treatment,
>                    family = binomial(), data = germ)
> broom::tidy(fit_grouped, exponentiate = TRUE, conf.int = TRUE)
>
> # Long form (expand success and failure counts to one row per seed)
> germ_long <- germ |>
>   dplyr::select(treatment, n_success, n_failure) |>
>   tidyr::pivot_longer(c(n_success, n_failure),
>                       names_to = "result", values_to = "count") |>
>   dplyr::mutate(outcome = ifelse(result == "n_success", 1L, 0L)) |>
>   tidyr::uncount(count)
>
> fit_long <- glm(outcome ~ treatment, family = binomial(), data = germ_long)
> coef(fit_grouped); coef(fit_long)   # identical
> ```
> **What you should see**: `treatmenthormone` odds ratio near the true value `(0.7 / 0.3) / (0.4 / 0.6) = 3.5`; across repeated experiments, a valid 95% CI would cover 3.5 about 95% of the time, but one simulated interval need not. Grouped and long forms give identical coefficients and fitted likelihood here.

---

### Q6 — Separation and Firth's correction

Simulate a small dataset where a predictor perfectly predicts the outcome:

```r
sep_df <- data.frame(
  x = c(1, 2, 3, 4, 5, 6, 7, 8),
  y = c(0, 0, 0, 0, 1, 1, 1, 1)
)
```

1. Fit `glm(y ~ x, family = binomial(), data = sep_df)`. What warning do you get? What happens to the coefficient and its SE?
2. Fit with `logistf::logistf(y ~ x, data = sep_df)`. Report the corrected coefficient and CI.
3. Explain why separation is a property of the observed predictor/outcome geometry and why it is especially common with small samples, rare outcomes, or many predictors.

> [!success]- Worked solution
> ```r
> sep_df <- data.frame(
>   x = 1:8,
>   y = c(0, 0, 0, 0, 1, 1, 1, 1)
> )
>
> fit_ml <- glm(y ~ x, family = binomial(), data = sep_df)
> # Warning: fitted probabilities numerically 0 or 1 occurred
> summary(fit_ml)   # huge coefficient, massive SE
>
> library(logistf)
> fit_firth <- logistf(y ~ x, data = sep_df)
> summary(fit_firth)   # finite, stable estimate
> ```
> **Why it happens**: with perfect separation, the likelihood approaches its supremum as the coefficient grows without bound, so there is no finite MLE. Firth's adjustment penalizes the likelihood using the Jeffreys invariant-prior term, reducing small-sample bias and usually producing finite estimates. Separation is especially common with sparse outcomes or high-dimensional predictors, but it can occur at any sample size when predictors perfectly classify the outcome.

---

## Mini-project (deliverable)

Create `module_07_report.qmd` on `ISLR2::Default`:

1. Fit the full model from Q1.
2. Odds ratio table with 95% CIs and a one-sentence interpretation of each.
3. ROC plot and AUC.
4. Cost-based threshold selection following Q4's logic — pick your own cost ratio and defend it.
5. Confusion matrix at your chosen threshold, with sensitivity, specificity, precision.
6. One paragraph: what would you tell a non-technical business stakeholder about this model's usefulness?

## Reflection

**What I learned**:

**What still confuses me** (link forward with wikilinks):

**Time spent**:

---

Previous: [[06 - Module 6 - GLM Framework]]  ·  Next: [[08 - Module 8 - Count GLMs and Beyond]]

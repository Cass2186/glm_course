---
title: Module 8 — Count GLMs and mixed models
tags: [stats, R, module-8, Poisson, negative-binomial, mixed-models, offsets]
module: 8
created: 2026-07-08
---

# Module 8 — Count GLMs and beyond

> [!info] Where this module goes
> Two big topics share a common motivation: **when the simple model isn't enough**. Poisson regression handles counts, but many real count datasets violate its conditional mean–variance relationship, so we need overdispersion diagnostics and alternatives such as the negative binomial. When observations are clustered (technical measurements, repeated measures, nested designs), mixed models can represent that dependence. The western blot example compares a mixed model with replicate averaging at the end.

## Objectives

By the end of this module you should be able to:

1. Fit a Poisson GLM and diagnose overdispersion.
2. Refit with a negative binomial family when Poisson fails.
3. Use an **offset** to model rates (events per unit exposure).
4. Recognize when zero-inflation matters and know the tools.
5. Fit a basic **mixed-effects** model with `lme4` — separate fixed effects from random effects.
6. Apply a random-intercept model to western blot measurements clustered within biological replicates and compare it with averaging.

## Concepts

### Poisson regression

The Poisson distribution is the natural home for counts of independent events over a fixed exposure. Mean equals variance: E[y] = var(y) = λ. The GLM uses a log link:

$$\log(E[y]) = \beta_0 + \beta_1 x_1 + \dots$$

```r
library(MASS)
fit <- glm(Days ~ Age + Sex + Eth + Lrn, family = poisson(), data = quine)
summary(fit)
```

Exponentiated coefficients are **expected-count ratios** per unit x. They are rate ratios when exposure is constant or explicitly handled with an offset.

### Overdispersion: the Poisson's failure mode

Many count datasets show conditional variance greater than the fitted Poisson mean — "overdispersion." Sources include unmeasured heterogeneity, clustering, contagion, an omitted mean structure, or zero inflation.

A quick screen is residual deviance divided by residual degrees of freedom, but this ratio is not itself an estimate of "times more variance" and can behave poorly with small fitted counts. A Pearson dispersion statistic or simulation-based check is preferable; `performance::check_overdispersion()` provides a convenient test.

```r
performance::check_overdispersion(fit)
```

If ignored, overdispersion makes standard errors too small, p-values too small, confidence intervals too narrow — you claim more precision than the data supports.

### Fixing overdispersion

Two mainstream fixes:

1. **Quasi-Poisson**: `glm(..., family = quasipoisson())`. Same coefficients, but SEs inflated by an estimated dispersion factor. Fast, easy — but no likelihood, so you can't use AIC or LRT.
2. **Negative binomial**: `MASS::glm.nb()`. Explicitly models extra-Poisson variance with a parameter θ. It has a full likelihood, so AIC and likelihood-ratio methods are available. Choose between it, quasi-Poisson, random effects, and other count models based on the source and pattern of dispersion rather than by default.

```r
fit_nb <- MASS::glm.nb(Days ~ Age + Sex + Eth + Lrn, data = quine)
summary(fit_nb)
```

### Offsets: modeling rates

Count data often comes with an **exposure** — how long, how big, how many trials. To model **rate** rather than raw count, use an offset. This forces a coefficient of 1 on log(exposure):

$$\log(E[y]) = \log(\mathrm{exposure}) + \beta_0 + \beta_1 x_1$$

which rearranges to:

$$\log\left(\frac{E[y]}{\mathrm{exposure}}\right) = \beta_0 + \beta_1 x_1$$

```r
# Crimes per city, adjusting for population
glm(crimes ~ policy + offset(log(population)),
    family = poisson(), data = cities)
```

With a log link, supply `offset(log(exposure))`; exposure must be strictly positive. With another link, the offset must be expressed on that link's linear-predictor scale.

### Zero-inflation (brief)

Some datasets have more zeros than any Poisson or negative binomial can explain — because two processes generate the data: a "structural zero" (never at risk) and a "count when at risk." Example: seizures per week where some patients are cured (structural zero) and others have a positive rate.

The `pscl::zeroinfl()` and `glmmTMB::glmmTMB()` fits zero-inflated GLMs. Warning signs: the count model dramatically underpredicts zeros compared to observed.

### Mixed-effects models: fixed vs random

When observations aren't independent, ordinary GLMs give wrong standard errors. Sources of non-independence:

- Repeated measures on the same subject.
- Technical replicates within biological replicates.
- Students within classrooms.
- Patients within hospitals.

**Fixed effects** are population-level coefficients or specific contrasts represented directly (treatment, dose, genotype).
**Random effects** are group-specific deviations modeled as draws from a distribution (subject, batch, site), which induces within-group correlation and partial pooling. The choice is about the sampling structure and inferential target, not simply whether you "care" about particular levels.

The `lme4` package fits both:

```r
library(lme4)

# Linear mixed model
lmer(y ~ treatment + (1 | subject), data = df)

# Generalized (binomial, Poisson, etc.)
glmer(y ~ treatment + (1 | subject), family = binomial(), data = df)
```

The `(1 | subject)` piece says "give each subject its own random intercept, drawn from a Normal distribution." You add a variance component per random effect.

### Reading a mixed model output

```r
summary(fit)$varcor    # variance components (random effects)
summary(fit)$coef      # fixed-effect estimates
```

The random-effects variance quantifies **how much subjects differ from each other** after accounting for fixed effects. A large random-intercept variance signals substantial clustering, but the direction and size of changes in fixed-effect SEs depend on the design and model.

### p-values in mixed models: a warning

For Gaussian `lmer()` fits, `lme4` deliberately omits fixed-effect p-values because denominator degrees of freedom are not uniquely defined. `glmer()` summaries do print asymptotic Wald z-tests, which can be unreliable with few clusters or sparse data. Options include:

- `lmerTest` package — adds Satterthwaite or Kenward-Roger degrees of freedom.
- Likelihood ratio tests via `anova()` on nested models.
- Parametric bootstrap.

For publishable work, prespecify the method and report estimates with intervals. Satterthwaite/Kenward–Roger methods, carefully constructed likelihood-ratio tests, or parametric bootstrap may be suitable depending on the model; likelihood-ratio comparisons of fixed effects should use maximum-likelihood rather than REML fits.

## Practice

Work in `module_08.R` or `module_08.qmd`.

---

### Q1 — Poisson and overdispersion diagnosis

Fit `glm(Days ~ Age + Sex + Eth + Lrn, family = poisson(), data = MASS::quine)`.

1. Check `deviance(fit) / df.residual(fit)`. Is it near 1?
2. Run `performance::check_overdispersion(fit)`. Interpret the result.
3. Explain in one sentence what would happen to your Sex coefficient's p-value if you ignored the overdispersion.

> [!success]- Worked solution
> ```r
> library(MASS)
> fit <- glm(Days ~ Age + Sex + Eth + Lrn,
>            family = poisson(), data = quine)
>
> deviance(fit) / df.residual(fit)   # around 12 — massive overdispersion
> performance::check_overdispersion(fit)
> ```
> **What you should see**: residual deviance/df is about 12.2, and the formal check also shows severe overdispersion. This strongly indicates that Poisson standard errors are underestimated; the size of the ratio should not be read literally as "12.2 times the variance." Inference from this Poisson fit should not be reported without addressing the misspecification.

---

### Q2 — Negative binomial fix

Refit as `MASS::glm.nb(Days ~ Age + Sex + Eth + Lrn, data = quine)`.

1. Compare the Sex coefficient's standard error to the naïve Poisson SE from Q1. How much bigger?
2. Compare AIC of the Poisson and NB models with `AIC()`.
3. Report the `Sex` expected-count ratio with a 95% CI on the response scale.

> [!success]- Worked solution
> ```r
> fit_pois <- glm(Days ~ Age + Sex + Eth + Lrn,
>                 family = poisson(), data = quine)
> fit_nb   <- glm.nb(Days ~ Age + Sex + Eth + Lrn, data = quine)
>
> broom::tidy(fit_pois)[, c("term", "estimate", "std.error")]
> broom::tidy(fit_nb)[, c("term", "estimate", "std.error")]
>
> AIC(fit_pois, fit_nb)
>
> exp(coef(fit_nb))
> exp(confint(fit_nb))
> ```
> **What you should see**: NB standard errors are much larger. AIC drops dramatically for NB — the difference is often in the hundreds. The `SexM` rate ratio is around 0.9 to 1.1 with a wide CI that includes 1 — meaning after properly accounting for overdispersion, there's no strong evidence for a Sex effect. The naïve Poisson fit would have declared this significant.

---

### Q3 — Offsets for rate modeling

Simulate a small city-level dataset where crime count depends on a policy variable, but cities vary 10-fold in population.

```r
set.seed(2026)
n_cities <- 100
cities <- data.frame(
  policy     = rbinom(n_cities, 1, 0.5),
  population = round(exp(runif(n_cities, log(10000), log(1e6))))
)
true_rate <- 0.001 * exp(-0.3 * cities$policy)
cities$crimes <- rpois(n_cities, lambda = true_rate * cities$population)
```

1. Fit `glm(crimes ~ policy, family = poisson(), data = cities)` (no offset). What's the policy coefficient?
2. Fit `glm(crimes ~ policy + offset(log(population)), family = poisson(), data = cities)`. What's the policy coefficient now?
3. Which one recovers the truth (β for policy = −0.3)?

> [!success]- Worked solution
> ```r
> no_offset  <- glm(crimes ~ policy, family = poisson(), data = cities)
> with_offset <- glm(crimes ~ policy + offset(log(population)),
>                    family = poisson(), data = cities)
>
> coef(no_offset)   # driven mostly by population differences
> coef(with_offset) # recovers ~ -0.3
> ```
> **Why the no-offset fit fails**: it's fitting the total crime count, which is dominated by city size. The policy variable gets whatever random correlation it has with population, not its true effect on rate. The offset says "compare crime rates" and isolates the policy effect.

---

### Q4 — Warp breaks with `emmeans`

Fit a Poisson GLM to `warpbreaks`: `glm(breaks ~ wool * tension, family = poisson(), data = warpbreaks)`.

1. Read the coefficients on the link scale.
2. Get estimated marginal means and all pairwise contrasts on the **response scale** with `emmeans`.
3. Which tension level produces the fewest breaks, for each wool type?

> [!success]- Worked solution
> ```r
> library(emmeans)
> fit <- glm(breaks ~ wool * tension, family = poisson(), data = warpbreaks)
>
> emm <- emmeans(fit, ~ tension | wool, type = "response")
> emm
> pairs(emm)
> ```
> **Typical readings**: For wool A, medium tension has the smallest fitted mean (about 24.0 breaks, very close to high at 24.6). For wool B, high tension has the smallest fitted mean (about 18.8). Use the pairwise intervals/tests before claiming that close fitted means differ.

---

### Q5 — Mixed model: binomial GLMM on `cbpp`

The `lme4::cbpp` dataset has incidence of pleuropneumonia in herds of cattle across four time periods. Herds are the natural cluster.

Fit:

```r
library(lme4)
fit <- glmer(cbind(incidence, size - incidence) ~ period + (1 | herd),
             family = binomial(), data = cbpp)
```

1. Print `summary(fit)`.
2. Extract the random-effect variance for herd with `VarCorr(fit)`. Interpret it: how much heterogeneity is there between herds after accounting for period?
3. Now fit the same model **without** the random effect (`glm()` with binomial family). Compare period-effect standard errors. What's the practical difference?

> [!success]- Worked solution
> ```r
> fit_glmer <- glmer(cbind(incidence, size - incidence) ~ period + (1 | herd),
>                    family = binomial(), data = cbpp)
> summary(fit_glmer)
> VarCorr(fit_glmer)
>
> fit_glm <- glm(cbind(incidence, size - incidence) ~ period,
>                family = binomial(), data = cbpp)
>
> broom::tidy(fit_glmer)
> broom::tidy(fit_glm)
> ```
> **What you should see**: the random-intercept SD is about 0.64 on the log-odds scale. The GLMM changes both estimates and SEs; in the current data the period-coefficient SEs are only modestly larger than in the ordinary GLM (for example, period 2: about 0.303 vs 0.291). The main issue is that the ordinary GLM imposes independence and cannot represent between-herd heterogeneity, not that every GLMM standard error must be larger.

---

### Q6 — Western blot with a biological-replicate random intercept

Return to the raw (un-averaged) western blot data. This time, model all 48 rows and put a random intercept on `Bio_Rep` nested within `Genotype`.

```r
library(dplyr); library(lme4); library(lmerTest)

blot <- read.csv("data/western_blot_example.csv") |>
  mutate(
    Ratio    = Target_Intensity / Loading_Control_Intensity,
    Genotype = factor(Genotype, levels = c("WT", "HET", "KO", "RESCUE")),
    Bio_Rep  = factor(paste(Genotype, Bio_Rep, sep = "_"))  # unique bio-rep IDs
  )

fit_mixed <- lmer(log(Ratio) ~ Genotype + (1 | Bio_Rep), data = blot)
```

1. Print `summary(fit_mixed)`. Read the fixed effects table.
2. Extract the Bio_Rep random-effect variance. What fraction of the total variance sits at the biological-replicate level vs the technical-replicate level?
3. Compare the fixed-effect SEs to the Module 3 fit (`lm(log(Ratio) ~ Genotype, data = blot_bio)` on the collapsed data). Are they similar?
4. Run Dunnett-style control contrasts with multiplicity adjustment and Satterthwaite degrees of freedom using `emmeans`.

> [!tip]- Hint
> In this balanced design with three measurements per biological replicate and no measurement-level predictors, averaging and the random-intercept model contain essentially the same information about genotype means. The mixed model additionally estimates within- and between-biological-replicate variance and handles imbalance more naturally; it is not guaranteed to give narrower fixed-effect SEs.

> [!success]- Worked solution
> ```r
> library(lme4); library(lmerTest); library(emmeans)
>
> fit_mixed <- lmer(log(Ratio) ~ Genotype + (1 | Bio_Rep), data = blot)
> summary(fit_mixed)
>
> as.data.frame(VarCorr(fit_mixed))
>
> emm <- emmeans(fit_mixed, ~ Genotype, lmer.df = "satterthwaite")
> contrast(emm, method = "trt.vs.ctrl", ref = "WT", adjust = "mvt")
> ```
> **What you should see**: the Bio_Rep random-intercept variance is about 0.00272 and the residual variance about 0.000327 on the log scale. The intraclass correlation is therefore about `0.00272 / (0.00272 + 0.000327) ≈ 0.89`. The collapsed-`lm` and mixed-model genotype SEs are virtually identical here (for example, KO: about 0.03761 vs 0.03762), as expected from this balanced layout.
>
> **Why use this model**: it keeps measurement-level rows without pretending they are independent biological samples, exposes the two variance components, and extends cleanly to imbalance or measurement-level covariates. Averaging first is also valid for the simple balanced design. Neither approach repairs a design with too few independent biological replicates or unmodeled blot/batch effects.

---

## Mini-project (deliverable)

Create `module_08_report.qmd`. Use `MASS::quine`:

1. Fit Poisson and negative binomial versions of the full model.
2. Show overdispersion diagnostic.
3. Coefficient table on the response scale (expected-count ratios; exposure is the same observation window) for the NB fit.
4. Predictions for two hypothetical students with different profiles, with 95% intervals.
5. One paragraph reflecting: how would you extend this to a mixed model if you had multiple schools in the data?

## Reflection

**What I learned**:

**What still confuses me** (link forward with wikilinks):

**Time spent**:

---

Previous: [[07 - Module 7 - Logistic Regression]]  ·  Next: [[09 - Capstone]]

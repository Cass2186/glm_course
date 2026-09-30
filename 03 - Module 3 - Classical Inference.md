---
title: Module 3 — Classical inference (with western blot ANOVA case study)
tags: [stats, R, module-3, t-test, ANOVA, western-blot, post-hoc]
module: 3
created: 2026-07-08
---

# Module 3 — Classical inference

> [!info] The applied throughline
> This module uses your western blot workflow as the running example. You already tidied [[data/western_blot_example.csv]] in [[01 - Module 1 - R Foundations]] Q5 — that same `blot_bio` data frame (16 rows, one per biological replicate) drives Q2–Q4 here.

## Objectives

By the end of this module you should be able to:

1. Choose between one-sample, two-sample, paired, Student's, and Welch's t-tests — and defend the choice.
2. Fit a one-way ANOVA with either `aov()` or `lm() + anova()` and read the output.
3. Run and interpret post-hoc tests: **Tukey HSD** for all pairwise contrasts, **Dunnett** for compare-to-control.
4. Check ANOVA assumptions (normality of residuals, homoscedasticity) and know what to do when they fail — log transform, Welch's ANOVA, non-parametric alternatives.
5. Handle a two-way ANOVA with interaction and interpret main effects vs interaction.
6. Run a χ² test of independence and know when to use Fisher's exact instead.
7. Choose Pearson vs Spearman correlation.

## Concepts

### Which t-test?

Decision tree:

- **One group vs a known value** → `t.test(x, mu = value)`.
- **Two independent groups, sample sizes and variances similar** → Student's: `t.test(x, y, var.equal = TRUE)`.
- **Two independent groups, unequal variances or unequal n** → **Welch's** (R's default): `t.test(x, y)`.
- **Two measurements on the same subjects** (before/after) → paired: `t.test(x, y, paired = TRUE)`.

Practical rule: **default to Welch's.** You saw in [[02 - Module 2 - Distributions and Simulation]] Q4 that Student's t-test misbehaves when variances and sample sizes are both unequal. Welch's costs you effectively nothing when variances are equal and saves you when they aren't.

### One-way ANOVA

An ANOVA asks: *are the means of ≥ 2 groups all equal?*

The math:

- **Between-group variance** = how much group means differ from the grand mean.
- **Within-group variance** = how much observations differ from their own group mean.
- **F-statistic** = between / within. Big F ⇒ groups differ.

Two equivalent ways to fit it in R:

```r
# As ANOVA
aov_fit <- aov(Ratio ~ Genotype, data = blot_bio)
summary(aov_fit)

# As linear model + ANOVA table (identical F, p)
lm_fit <- lm(Ratio ~ Genotype, data = blot_bio)
anova(lm_fit)
```

The `lm()` version is worth internalizing — **ANOVA is just a linear model with a categorical predictor**. Everything you learn about `lm()` in [[04 - Module 4 - Linear Regression]] applies. The coefficient for `GenotypeKO` is literally `mean(KO) − mean(WT)` when WT is the reference level.

### Assumptions of the F-test

1. **Independence** of observations. This is the one that kills wet-lab ANOVAs: technical replicates are **not independent**. Average them first (what we did in Module 1 Q5) or model the nesting explicitly ([[08 - Module 8 - Count GLMs and Beyond]]).
2. **Normality of residuals** (not of the raw data — this is a common confusion).
3. **Homoscedasticity** — variance roughly equal across groups.

Checks:

```r
shapiro.test(residuals(lm_fit))     # p > 0.05 = fail to reject normality
car::leveneTest(Ratio ~ Genotype, data = blot_bio)  # large p: fail to reject equal variances
plot(lm_fit, which = 1)             # residuals vs fitted — look for funneling
plot(lm_fit, which = 2)             # QQ plot of residuals
```

A large p-value is not evidence that an assumption is true. These tests have little power in small samples, so use them alongside the plots and knowledge of the design; independence cannot be diagnosed from a residual plot.

**When assumptions fail** (western blot ratios often violate homoscedasticity — variance scales with the mean):

- Log-transform the response: `lm(log(Ratio) ~ Genotype)`. Interprets as % changes rather than absolute. This is the workhorse fix for multiplicative biology.
- **Welch's ANOVA** — robust to unequal variances: `oneway.test(Ratio ~ Genotype, var.equal = FALSE)`.
- **Kruskal-Wallis** — rank-based alternative: `kruskal.test(Ratio ~ Genotype, data = blot_bio)`. It tests equality of distributions, not means; interpreting it as a location/median comparison requires similarly shaped group distributions.

### Post-hocs: Tukey vs Dunnett vs Bonferroni

A significant ANOVA tells you *someone* differs, not who. Post-hocs answer *who*.

- **Tukey HSD** — all pairwise comparisons, controls family-wise error rate. Use when you genuinely care about every pair.
- **Dunnett** — every group vs a single control. Use when you have a control (WT) and want to compare each treatment to it. It is generally more powerful for those planned comparisons than protecting all pairwise comparisons with Tukey.
- **Bonferroni** — divide α by the number of tests. Simple and valid under arbitrary dependence, but often conservative.

For a western blot question defined in advance as "compare each group with WT," Dunnett directly matches the scientific question.

```r
# Tukey
TukeyHSD(aov_fit)

# Dunnett — via multcomp
library(multcomp)
dunnett <- glht(aov_fit, linfct = mcp(Genotype = "Dunnett"))
summary(dunnett)
```

### Two-way ANOVA and interactions

Two categorical predictors, one continuous response. Beyond checking each factor's main effect, we also ask whether they *interact* — does the effect of factor A depend on the level of factor B?

```r
# Main effects only
lm(y ~ A + B, data = df) |> anova()

# With interaction
lm(y ~ A * B, data = df) |> anova()   # A * B expands to A + B + A:B
```

Interpret carefully: with an interaction in the model, a treatment-coded "main effect" coefficient is the effect at the other factor's reference level, not an overall effect. Describe simple effects (the effect of A at relevant levels of B) or suitable marginal contrasts. Interaction plots (or `emmeans::emmip`) make this concrete.

`anova(lm(...))` reports sequential (Type I) sums of squares. In a balanced factorial design the usual tests are order-invariant; in an unbalanced design, changing term order can change the table. Define the scientific contrasts or marginal means you need rather than choosing a sums-of-squares "type" mechanically.

### Chi-squared and Fisher's exact

For two categorical variables, test independence:

```r
chisq.test(table(df$var1, df$var2))
```

The familiar "every expected count must be at least 5" rule is too rigid. A common approximation check is no expected count below 1 and at least 80% at or above 5. For sparse tables, use an exact or simulated procedure; `fisher.test()` is straightforward for 2×2 tables and can also handle small larger tables when computation permits.

### Correlation

- **Pearson** — measures linear association. Computing it needs no normality assumption; the usual small-sample test and CI rely on stronger distributional assumptions (classically bivariate normality).
- **Spearman** — rank-based and sensitive to monotone (not necessarily linear) association; it is usually less sensitive than Pearson to extreme magnitudes, though influential ranks can still matter.

If you're not sure they're linear, or you have obvious outliers, use Spearman.

## Practice

Work in `module_03.R` or `module_03.qmd`. Load the tidied western blot data first:

```r
library(dplyr)
library(ggplot2)
library(broom)

blot <- read.csv("data/western_blot_example.csv") |>
  mutate(Ratio = Target_Intensity / Loading_Control_Intensity)

blot_bio <- blot |>
  group_by(Genotype, Bio_Rep) |>
  summarise(Ratio = mean(Ratio), .groups = "drop") |>
  mutate(Genotype = factor(Genotype, levels = c("WT", "HET", "KO", "RESCUE")))
```

---

### Q1 — Warm-up: WT vs KO, two ways

Subset `blot_bio` to just WT and KO. Run a two-sample t-test comparing `Ratio` between the two groups:

1. Using **Welch's** t-test (`var.equal = FALSE`, the default).
2. Using **Student's** t-test (`var.equal = TRUE`).
3. As a **one-way ANOVA** on the same two groups (`aov(Ratio ~ Genotype, data = ...)`).

Are the three p-values the same? Which one changes if you switch to unequal variance assumption in the ANOVA?

> [!tip]- Hint
> `blot_bio |> filter(Genotype %in% c("WT", "KO")) |> mutate(Genotype = droplevels(Genotype))` — the `droplevels()` is important, otherwise the factor still remembers HET and RESCUE and downstream tools may complain.

> [!success]- Worked solution
> ```r
> two_groups <- blot_bio |>
>   filter(Genotype %in% c("WT", "KO")) |>
>   mutate(Genotype = droplevels(Genotype))
>
> t.test(Ratio ~ Genotype, data = two_groups)                    # Welch's
> t.test(Ratio ~ Genotype, data = two_groups, var.equal = TRUE)  # Student's
> summary(aov(Ratio ~ Genotype, data = two_groups))              # ANOVA
> ```
> **What you should see**:
> - Student's t and one-way ANOVA on two groups give **exactly the same p-value** — because F = t² when there are exactly two groups. This is a key identity for building intuition that ANOVA is just linear regression with a categorical predictor.
> - Welch's p-value is slightly different because it uses adjusted degrees of freedom.
> - The effect is enormous (the KO mean is about 43% of the WT mean), so all three are highly significant. This is by design — real western blots often have subtler effects and you'll want the flexibility of Welch's.

---

### Q2 — One-way ANOVA on all four genotypes

Fit a one-way ANOVA of `Ratio` on `Genotype` using the full `blot_bio` (n = 16, 4 per group). Do it **both** ways:

1. `aov()` + `summary()`.
2. `lm()` + `anova()`.

Confirm the F and p match. Then use `broom::tidy()` on the `lm()` fit and interpret each coefficient — what quantity does `GenotypeKO` represent, in your data's units?

> [!success]- Worked solution
> ```r
> aov_fit <- aov(Ratio ~ Genotype, data = blot_bio)
> summary(aov_fit)
>
> lm_fit <- lm(Ratio ~ Genotype, data = blot_bio)
> anova(lm_fit)
>
> tidy(lm_fit)
> ```
> **Interpretation of `lm()` coefficients** (WT is reference):
> - `(Intercept)` ≈ mean of the WT group.
> - `GenotypeHET` ≈ mean(HET) − mean(WT). Should be negative (HET is intermediate).
> - `GenotypeKO` ≈ mean(KO) − mean(WT). Strongly negative (KO ~⅔ lower).
> - `GenotypeRESCUE` ≈ mean(RESCUE) − mean(WT). Should be near zero (rescue worked).
>
> The F-test p-value is < 0.001. You've established *someone* differs — Q3 tells you *who*.

---

### Q3 — Post-hoc: Tukey vs Dunnett

Take the `aov_fit` from Q2 and run both:

1. **Tukey HSD** — all 6 pairwise comparisons.
2. **Dunnett** (via `multcomp::glht`) — 3 comparisons, all vs WT.

Compare the adjusted p-values for `KO - WT` between the two methods. Which is smaller and why? Which would you report in a paper where WT is the natural baseline?

> [!tip]- Hint
> `install.packages("multcomp")` if you don't have it. Syntax:
> ```r
> library(multcomp)
> dunnett <- glht(aov_fit, linfct = mcp(Genotype = "Dunnett"))
> summary(dunnett)
> ```

> [!success]- Worked solution
> ```r
> TukeyHSD(aov_fit)
>
> library(multcomp)
> dunnett <- glht(aov_fit, linfct = mcp(Genotype = "Dunnett"))
> summary(dunnett)
> confint(dunnett)   # useful for the paper figure
> ```
> **What you should see**: in this dataset, Dunnett's adjusted p for KO − WT is smaller than Tukey's. Dunnett protects the three control comparisons rather than all six pairwise comparisons, so it is generally more powerful for the planned control contrasts (although adjusted p-values also depend on the contrasts' correlation and are not determined by the count alone).
>
> **What to report**: For "did each genotype differ from WT?", report Dunnett. For "which genotypes differ from each other in general?", report Tukey. Never run both and pick whichever is more significant — that's a garden of forking paths.

---

### Q4 — Assumption checks (and a log transform)

For `lm_fit` from Q2:

1. Plot residuals vs fitted (`plot(lm_fit, which = 1)`) and the QQ plot (`plot(lm_fit, which = 2)`). Do you see funneling or heavy tails?
2. Formal tests: `shapiro.test(residuals(lm_fit))` and `car::leveneTest(Ratio ~ Genotype, data = blot_bio)`. Interpret both p-values.
3. Refit on the log scale: `lm(log(Ratio) ~ Genotype, data = blot_bio)`. Re-check the assumptions. Are they better?
4. On the log-scale fit, how do you back-transform the `GenotypeKO` coefficient into "KO is X% of WT"?

> [!success]- Worked solution
> ```r
> par(mfrow = c(1, 2))
> plot(lm_fit, which = 1)   # residuals vs fitted
> plot(lm_fit, which = 2)   # QQ plot
> par(mfrow = c(1, 1))
>
> shapiro.test(residuals(lm_fit))
> car::leveneTest(Ratio ~ Genotype, data = blot_bio)
>
> # Log-scale refit
> lm_log <- lm(log(Ratio) ~ Genotype, data = blot_bio)
> shapiro.test(residuals(lm_log))
> car::leveneTest(log(Ratio) ~ Genotype, data = blot_bio)
> summary(lm_log)
> ```
> **Back-transforming**: in the supplied data, `GenotypeKO` is about −0.84. `exp(-0.84) ≈ 0.43`. Interpretation: the KO geometric mean is about **43% of WT**, or equivalently about a 57% reduction. The log-scale model is often useful when variability is multiplicative or variance increases with the mean; diagnostics determine whether it improves this analysis.
>
> **Practical rule**: for a strictly positive ratio or concentration, especially one spanning an order of magnitude, consider a log scale and compare diagnostics and scientific interpretation. Counts should generally use a count model rather than a log-transformed Gaussian model. For this dataset, both raw- and log-scale diagnostics are reasonable.

---

### Q5 — Two-way ANOVA (with `ToothGrowth`)

Switch datasets for this one — western blot with a second factor requires more data than you have, and `ToothGrowth` is the canonical teaching example: guinea pigs' odontoblast length by vitamin C **supplement type** (OJ vs VC) and **dose** (0.5, 1, 2 mg/day).

```r
tg <- ToothGrowth
tg$dose <- factor(tg$dose)   # treat as categorical for a proper 2-way ANOVA
```

1. Fit `lm(len ~ supp + dose, data = tg)` (main effects only). Read the `anova()` table. Which main effects are significant?
2. Fit `lm(len ~ supp * dose, data = tg)`. Is the interaction significant?
3. Make an **interaction plot** with `emmeans::emmip(fit, supp ~ dose)`. Describe in one sentence what the interaction looks like.
4. If the interaction is significant, why is it misleading to report the main effect of `supp` on its own?

> [!success]- Worked solution
> ```r
> library(emmeans)
> tg <- ToothGrowth
> tg$dose <- factor(tg$dose)
>
> fit_main <- lm(len ~ supp + dose, data = tg)
> anova(fit_main)
>
> fit_int <- lm(len ~ supp * dose, data = tg)
> anova(fit_int)   # interaction p ≈ 0.02
>
> emmip(fit_int, supp ~ dose, CIs = TRUE)
> ```
> **Interaction interpretation**: OJ produces more growth than VC at 0.5 and 1 mg, but the two supplements converge at 2 mg. In the treatment-coded interaction model, the `supp` coefficient is specifically the OJ–VC contrast at the reference dose (0.5 mg), not an average effect over doses. Report the within-dose contrasts to describe the pattern.
>
> When an interaction is significant, get contrasts *within levels of the moderating factor*:
> ```r
> emmeans(fit_int, pairwise ~ supp | dose)
> ```

---

### Q6 — χ² test of independence

Load `HairEyeColor` (built-in 3D array). Collapse to a 2D table of hair × eye colour (summing over sex), then test independence:

```r
tbl <- apply(HairEyeColor, c(1, 2), sum)
tbl
chisq.test(tbl)
```

1. Report χ², df, p.
2. Which cells drive the association? Look at `chisq.test(tbl)$stdres` — standardised residuals with |z| > 2 flag over/under-represented combinations.
3. Are any expected cell counts < 5? If yes, would Fisher's exact be more appropriate here?

> [!success]- Worked solution
> ```r
> tbl <- apply(HairEyeColor, c(1, 2), sum)
> res <- chisq.test(tbl)
> res
> res$expected      # all should be ≥ 5, so χ² is fine
> round(res$stdres, 2)
> ```
> **What you should see**: χ² ≈ 138.3 with 9 df, p < 2.2×10⁻¹⁶. Big positive residuals occur for Blond+Blue and Black+Brown; big negatives occur for Blond+Brown and Black+Blue. Hair- and eye-colour categories are associated in this sample. All expected counts exceed 7.6, so the chi-squared approximation is adequate; an exact or simulated test is useful for sparse tables.

---

### Q7 — Pearson vs Spearman

Using `mtcars`, compute Pearson and Spearman correlations of `hp` vs `mpg`. Then repeat after adding a single influential outlier to `hp`. What happens to each?

> [!success]- Worked solution
> ```r
> cor.test(mtcars$hp, mtcars$mpg, method = "pearson")
> cor.test(mtcars$hp, mtcars$mpg, method = "spearman")
>
> # Add an outlier
> mtcars2 <- mtcars
> mtcars2[1, "hp"] <- 2000
> cor.test(mtcars2$hp, mtcars2$mpg, method = "pearson")
> cor.test(mtcars2$hp, mtcars2$mpg, method = "spearman")
> ```
> **Takeaway for this edit**: Pearson changes from about −0.78 to −0.13, while Spearman changes from about −0.89 to −0.83. Spearman uses ranks, so the extreme magnitude matters far less, although changing one observation's rank can still affect it. Choose the measure based on the association of interest; an outlier should still be investigated rather than automatically handled by switching methods.

---

## Mini-project (deliverable) — publishable western blot ANOVA figure + stats

Create `module_03_report.qmd` that produces the analysis you'd actually put in a paper:

1. **Figure**: dot plot from Module 1 Q5 (mean ± SE bars), with significance brackets from your Dunnett post-hoc. Use `ggpubr::stat_pvalue_manual()` or add annotations manually.
2. **Stats paragraph** in prose:
   > "Band intensity, normalized to loading control, differed among genotypes (one-way ANOVA, F(3, 12) = XX.X, *p* < 0.001, n = 4 biological replicates per group). Compared to WT, expression was reduced in KO (Dunnett-adjusted *p* < 0.001, mean ratio 0.XX vs WT) and in HET (*p* = 0.0XX, mean ratio 0.XX), and was restored in RESCUE (*p* = 0.XX)."
3. **Methods paragraph** — one paragraph naming: normalization scheme (target/loading control per lane), technical replicate handling (averaged within biological replicate), the test used, post-hoc used, software (`R x.y.z`, `stats` and `multcomp` packages).
4. **Assumption checks appendix** — QQ plot, Levene's test result, log-scale sensitivity refit. Show your work.

## Reflection

**What I learned**:

**What still confuses me** (link forward with `[[...]]`):

**Time spent**:

---

Previous: [[02 - Module 2 - Distributions and Simulation]]  ·  Next: [[04 - Module 4 - Linear Regression]]

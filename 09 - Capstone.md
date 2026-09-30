---
title: Module 9 — Capstone
tags: [stats, R, module-9, capstone, project]
module: 9
created: 2026-07-08
---

# Module 9 — Capstone

> [!info] Purpose
> The point of a capstone isn't to learn new material — it's to demonstrate to yourself that you can go from a raw dataset to a defensible, reproducible analysis with no scaffolding. Pick the path that most resembles the work you'd like to be able to do routinely.

## Pick a path

You have three options. Each is a full end-to-end analysis producing a polished HTML deliverable. Do one. Do all three if you want to prove a point.

### Path A — Real western blot analysis (recommended if applicable)

**Setup**: bring your own real, unpublished densitometry data from an experiment you've run. Any experimental design that would go into a paper figure works: one-way (≥ 3 genotypes/treatments), two-way (genotype × drug, wild type × time), or repeated measures.

**Deliverable**: `capstone_A.qmd` producing a manuscript-quality figure and stats section.

1. **Data**: import the Excel/CSV file. Show `str()` and `head()` output.
2. **Cleaning and normalization**: show your loading-control normalization scheme and how you handled technical replicates (average or nest? — defend the choice).
3. **EDA plot**: dot plot with mean ± SEM per group; if two-way, faceted.
4. **Assumption checks**: QQ plot of residuals, Levene's test, sensitivity refit on the log scale.
5. **Statistical model**: one-way or two-way ANOVA (or mixed model if you kept the tech-rep structure). Justify the choice.
6. **Post-hoc**: Dunnett vs WT (or your chosen control), reported with adjusted p-values and 95% CIs on the effect scale.
7. **Manuscript figure**: publication-ready plot with significance brackets.
8. **Methods paragraph**: 3–5 sentences describing every choice you made, with software versions.
9. **Interpretation paragraph**: what does the result mean biologically? What are the caveats?

### Path B — Kaggle-style binary classification

**Setup**: pick a public dataset with a binary outcome. Good starters: `ISLR2::Default` (already used), `mlbench::PimaIndiansDiabetes`, `titanic::titanic_train`, or anything else with n ≥ ~500 and a meaningful outcome.

**Deliverable**: `capstone_B.qmd`.

1. **EDA**: describe the dataset, summary statistics, missingness, class balance.
2. **Feature engineering**: any transformations (log, binning, interactions).
3. **Modeling**: fit a logistic regression using training data. Report coefficients as odds ratios.
4. **Diagnostics**: check for separation, high leverage, VIF.
5. **Evaluation**: hold-out or cross-validated ROC, AUC, calibration plot.
6. **Threshold selection**: choose a threshold based on a stated cost ratio; report confusion matrix at that threshold.
7. **Comparison**: fit a Firth-corrected logistic (`logistf`) or ridge logistic regression (`glmnet` with `alpha = 0` and the penalty chosen within training data by cross-validation), and show whether it changes conclusions.
8. **Business/scientific interpretation**: 2–3 paragraphs.

### Path C — Simulation study on separation and Firth's correction

**Setup**: a methodological deep-dive that would be publishable as a stats-education paper.

**Deliverable**: `capstone_C.qmd`.

1. **Simulate**: n = 20 observations from a logistic model where a single binary predictor has coefficient β. Vary β from 0 (no effect) up to 5 (huge effect that will often cause separation).
2. **For each β**: run 1000 simulations. For each, fit both an ordinary `glm` (MLE) and `logistf` (Firth). Record coefficient estimate and CI.
3. **Compare**:
   - How often does the MLE fail to converge or return an "infinite" estimate (say, |β̂| > 10)?
   - What's the median bias of MLE vs Firth as β grows?
   - Coverage of the 95% CI — does it stay near 0.95 for both?
4. **Plot**: bias and coverage as functions of the true β.
5. **Write-up**: 3 paragraphs summarizing findings with recommendation for practitioners.

## Working guidelines (all paths)

- **Reproducibility**: use `set.seed()` for simulations. Record versions with `sessionInfo()`; to actually restore package versions, use a lockfile tool such as `renv` and keep `renv.lock` with the project.
- **Prose over code output**: your report should read like a paper, not a stack trace. Hide code output that's for you (`#| include: false`) and only surface what a reader needs.
- **Show your work**: keep sensitivity analyses in an appendix, not the main body.
- **One figure per main claim**: don't clutter.

## Rubric for self-evaluation

Score yourself out of 10 on each. Anything below 7 is worth iterating on before calling the project done.

1. Would a reviewer accept the methods paragraph as written?
2. Are the model choices defensible in one sentence each?
3. Are the diagnostics visible and honest?
4. Is the main claim supported by the strongest available test?
5. Are effect sizes and CIs reported, not just p-values?
6. Are your figures self-explanatory to someone opening this file cold?
7. Would rerunning your `.qmd` from scratch reproduce every number?
8. Have you flagged the biggest caveat / limitation?
9. Is the writing free of statistical jargon that isn't necessary?
10. Would you send this to your PI or team lead as-is?

## After the capstone

Once you're through this, you have a complete workflow for the four biggest statistical situations in wet-lab and applied research:

- Continuous outcome, group differences → [[03 - Module 3 - Classical Inference]] + [[04 - Module 4 - Linear Regression]]
- Binary outcome, risk factors → [[07 - Module 7 - Logistic Regression]]
- Count outcome, exposure varies → [[08 - Module 8 - Count GLMs and Beyond]]
- Non-independent observations (repeated measures, nested designs) → mixed models section of [[08 - Module 8 - Count GLMs and Beyond]]

For everything not covered, three follow-ups worth your time when the need arises:

- **Survival analysis**: Cox regression, Kaplan-Meier. `survival` package. Klein & Moeschberger textbook.
- **Bayesian regression**: `rstanarm` for near-drop-in replacements for `lm/glm/lmer` with sane default priors. Gelman's *Regression and Other Stories* is the modern intro.
- **Multiple testing and FDR**: `p.adjust()` with method "BH". Efron's *Large-Scale Inference* if you're doing genomics.

## Reflection

**Which path did you pick, and why?**

**What was the hardest part?**

**What would you do differently next time?**

**Time spent**:

---

Previous: [[08 - Module 8 - Count GLMs and Beyond]]

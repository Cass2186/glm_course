---
title: R Statistics & GLM Course — Overview
tags: [stats, R, GLM, course-index]
created: 2026-07-08
---

# R Statistics & GLM — Course Overview

> [!tip] Interactive version
> Every module has a companion page where the practice questions run in your browser (webR), with hints, solutions and interactive explorers. Source: `index.qmd` and `M0.qmd`–`M9.qmd`; rendered site: `docs/`.

A 10-part progression (an R primer, eight substantive modules, and a capstone), each ~1 week at ~3–5 hrs. Every substantive module has readings, coded practice, and a mini-project. The supplied examples use built-in or package datasets plus one included CSV, so they are reproducible.

> [!info] Applied wet-lab track
> Because your day-to-day involves **western blot densitometry pre-processed in Excel**, a running case study is threaded through the course using [[data/western_blot_example.csv]] (WT / HET / KO / RESCUE, 4 biological reps × 3 technical reps each):
> - **Module 1** — importing the Excel/CSV and getting it into tidy shape.
> - **Module 3** — the deep dive: normalization, tech/bio replicate handling, one-way and two-way ANOVA, Tukey/Dunnett post-hocs, log-transform vs Welch's ANOVA.
> - **Module 5** — checking assumptions (normality, homoscedasticity) on real blot data.
> - **Module 8** — mixed-effects treatment of measurements clustered within biological replicates (an alternative to averaging in this balanced example, and more flexible for unbalanced designs).

## Setup

Install once, then reuse throughout the course.

```r
install.packages(c(
  "tidyverse", "broom", "car", "MASS",
  "palmerpenguins", "ISLR2", "readxl",
  "performance", "emmeans", "DHARMa", "lme4", "lmerTest",
  "pROC", "logistf", "multcomp", "sandwich", "lmtest",
  "ggpubr", "betareg", "glmnet"
))
```

## Modules

| # | Module | Focus |
|---|--------|-------|
| 0 | [[00 - Module 0 - R Syntax Primer]] | Quick reference — assignment, subsetting, factors, formulas, pipes |
| 1 | [[01 - Module 1 - R Foundations]] | Data manipulation, plotting, tidy summaries |
| 2 | [[02 - Module 2 - Distributions and Simulation]] | `d/p/q/r*`, CLT, bootstrap intuition |
| 3 | [[03 - Module 3 - Classical Inference]] | t-tests, one-/two-way ANOVA, post-hocs — **western blot case study** |
| 4 | [[04 - Module 4 - Linear Regression]] | `lm()`, interactions, contrasts |
| 5 | [[05 - Module 5 - Regression Diagnostics]] | Residuals, leverage, VIF, transformations |
| 6 | [[06 - Module 6 - GLM Framework]] | Link functions, exponential family, deviance |
| 7 | [[07 - Module 7 - Logistic Regression]] | Binary GLMs, odds ratios, ROC/AUC |
| 8 | [[08 - Module 8 - Count GLMs and Beyond]] | Poisson, negative binomial, mixed models |
| 9 | [[09 - Capstone]] | Full analysis report or simulation study |

## How to work through this

1. Read the module note.
2. Open a new `.R` or `.qmd` file in the module folder and work each practice question.
3. Before checking the worked solution, get to a stopping point on your own — a wrong answer you can articulate is more useful than a correct one you copied.
4. Write 3–5 sentences at the bottom of each module note: *what I learned* / *what still confuses me*. Link confusions with `[[...]]` to future modules if relevant.
5. Revisit any question you couldn't do without help a week later.

## Datasets used

- `mtcars`, `iris`, `sleep`, `HairEyeColor`, `warpbreaks`, `PlantGrowth` — base R
- `palmerpenguins::penguins`
- `MASS::quine`, `MASS::birthwt`
- `ISLR2::Default`, `ISLR2::Wage`
- `ggplot2::diamonds`
- `lme4::cbpp`, `lme4::sleepstudy`
- [[data/western_blot_example.csv]] — simulated densitometry (4 genotypes × 4 bio reps × 3 tech reps)

## Reference books

- **ISLR2** — *An Introduction to Statistical Learning* (Ch. 3–4 especially). Free PDF at [statlearning.com](https://www.statlearning.com).
- **Gelman & Hill** — *Data Analysis Using Regression and Multilevel/Hierarchical Models*.
- **Fox & Weisberg** — *An R Companion to Applied Regression* (`car` package authors).
- **McCullagh & Nelder** — *Generalized Linear Models* (the classical GLM reference; skim, don't read cover-to-cover).

## Log

- 2026-09-10 — correctness audit: executed worked examples, corrected numerical answer keys and code, qualified statistical rules, and completed the dependency list.
- 2026-07-08 — course started. All 10 module notes drafted (Module 0 primer + Modules 1–8 + Capstone).

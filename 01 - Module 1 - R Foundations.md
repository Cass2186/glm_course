---
title: Module 1 — R foundations for statistics
tags: [stats, R, module-1, tidyverse, ggplot2]
module: 1
created: 2026-07-08
---

# Module 1 — R foundations for statistics

> [!info] Prerequisites
> A working R + RStudio (or VS Code with R + Quarto). The packages listed in [[00 - Course Overview]] installed.

## Objectives

By the end of this module you should be able to:

1. Load and inspect a data frame — dimensions, types, missingness.
2. Subset and filter rows and columns in **both** base R and `dplyr`.
3. Summarize a numeric variable grouped by a factor.
4. Build a layered `ggplot2` plot: aesthetics, geoms, facets, labels.
5. Extract a tidy summary from a model or test using `broom::tidy()`.
6. **Take an Excel-exported experimental dataset (western blot densitometry) and land it in tidy long form in R, normalized and ready for stats.**

## Concepts

### The data frame is the unit of analysis

Almost everything in applied statistics in R begins by getting your data into a **tidy data frame** — one row per observation, one column per variable. If you can get that right, 80% of the tooling in the ecosystem "just works."

Two styles you'll see everywhere:

**Base R**

```r
library(palmerpenguins)

mean(penguins$bill_length_mm, na.rm = TRUE)
penguins[penguins$species == "Gentoo", "body_mass_g"]
aggregate(body_mass_g ~ species, data = penguins, FUN = mean, na.rm = TRUE)
```

**Tidyverse (`dplyr`)**

```r
library(dplyr)

penguins |>
  filter(species == "Gentoo") |>
  summarise(mean_mass = mean(body_mass_g, na.rm = TRUE))

penguins |>
  group_by(species) |>
  summarise(mean_mass = mean(body_mass_g, na.rm = TRUE))
```

Both are fine. Learn to read both — legacy code and package internals use base R constantly, and papers you read may show either.

### The `ggplot2` grammar

A plot is `data + aesthetic mapping + geometry (+ facets + scales + labels + theme)`.

```r
library(ggplot2)

ggplot(penguins, aes(x = flipper_length_mm, y = body_mass_g, colour = species)) +
  geom_point(alpha = 0.7) +
  geom_smooth(method = "lm", se = FALSE) +
  facet_wrap(~ island) +
  labs(
    x = "Flipper length (mm)",
    y = "Body mass (g)",
    title = "Body mass increases with flipper length across species"
  )
```

Key mental model: `aes()` maps *columns of your data* to *visual channels* (x, y, colour, size, shape). Anything constant (a single colour for all points) goes **outside** `aes()`.

### Getting your Excel data into R

If your workflow is **Excel for pre-processing → R for stats** (very common for western blot densitometry, qPCR, ELISA, etc.), the entry point is `readxl::read_excel()` or the humble `read.csv()` if you've exported first.

```r
library(readxl)
blot <- read_excel("data/western_blot.xlsx", sheet = "raw")

# Or, if you exported to CSV first:
blot <- read.csv("data/western_blot_example.csv")

# Always inspect after import
str(blot)
head(blot)
summary(blot)
```

**Two rules that will save you hours of debugging later**:

1. **Tidy shape**: one row per observation, one column per variable. If your Excel sheet has genotypes as columns (WT | KO | HET | RESCUE), you need to pivot to *long* format:
   ```r
   library(tidyr)
   long <- wide |> pivot_longer(
     cols = c(WT, KO, HET, RESCUE),
     names_to  = "Genotype",
     values_to = "Intensity"
   )
   ```
   Formula interfaces such as `lm()`, `aov()`, and `t.test()` are easiest to use when each measured observation is a row and variables occupy columns.

2. **Make factors explicit and set the reference level yourself**. R will otherwise pick alphabetically, which for a WT/KO comparison means HET becomes the reference — usually not what you want.
   ```r
   blot$Genotype <- factor(blot$Genotype, levels = c("WT", "HET", "KO", "RESCUE"))
   ```

We'll build on this in [[03 - Module 3 - Classical Inference]] with the full densitometry → normalization → ANOVA pipeline using `[[data/western_blot_example.csv]]`.

### Tidy summaries with `broom`

Every test and model in R returns some list-like object with different slots. `broom` gives you a consistent tidy data frame back:

```r
library(broom)
t.test(bill_length_mm ~ sex, data = penguins) |> tidy()
lm(body_mass_g ~ flipper_length_mm, data = penguins) |> tidy()
lm(body_mass_g ~ flipper_length_mm, data = penguins) |> glance()  # model-level stats
```

This will save you enormous friction later when we want to compare 5 models side by side.

## Practice

Do these in a script `module_01.R` or a Quarto doc `module_01.qmd` in this folder. Attempt each fully before opening the solution.

---

### Q1 — Summaries two ways

Load `palmerpenguins::penguins`. Compute the mean, median, and standard deviation of `bill_length_mm` **for each species**. Do it once with base R (`aggregate` or `tapply`) and once with `dplyr` (`group_by` + `summarise`). Confirm the numbers match.

> [!tip]- Hint
> `na.rm = TRUE` matters — `penguins` has NAs. `aggregate()` uses a formula interface: `aggregate(y ~ group, data = df, FUN = mean)`. For multiple statistics with `dplyr`, use `across()` or list multiple named arguments to `summarise()`.

> [!success]- Worked solution
> ```r
> library(palmerpenguins)
> library(dplyr)
>
> # Base R
> aggregate(bill_length_mm ~ species, data = penguins,
>           FUN = function(x) c(mean = mean(x, na.rm = TRUE),
>                                median = median(x, na.rm = TRUE),
>                                sd = sd(x, na.rm = TRUE)))
>
> # dplyr
> penguins |>
>   group_by(species) |>
>   summarise(
>     mean_bill   = mean(bill_length_mm,   na.rm = TRUE),
>     median_bill = median(bill_length_mm, na.rm = TRUE),
>     sd_bill     = sd(bill_length_mm,     na.rm = TRUE),
>     n           = sum(!is.na(bill_length_mm))
>   )
> ```
> **Interpretation to write in your notes**: Adelie ~39 mm, Chinstrap ~49 mm, Gentoo ~47 mm — bill length nearly separates Adelie from the other two but not Chinstrap from Gentoo. Keep this in mind for Module 4 when we regress on species.

---

### Q2 — Three plots of flipper vs body mass

Make three separate plots of `flipper_length_mm` (x) vs `body_mass_g` (y):

1. All points, one colour, with an overall linear smoother.
2. Coloured by `species`, with per-species linear smoothers.
3. Faceted by `island`, coloured by `species`.

For each, write one sentence describing what the plot tells you that the previous one didn't.

> [!tip]- Hint
> `geom_point(alpha = 0.7)` helps with overplotting. `geom_smooth(method = "lm", se = FALSE)` for a line without CI ribbon. `facet_wrap(~ island)` for facets.

> [!success]- Worked solution
> ```r
> library(ggplot2)
>
> # 1
> ggplot(penguins, aes(flipper_length_mm, body_mass_g)) +
>   geom_point(alpha = 0.7) +
>   geom_smooth(method = "lm", se = FALSE) +
>   labs(title = "One overall trend")
>
> # 2
> ggplot(penguins, aes(flipper_length_mm, body_mass_g, colour = species)) +
>   geom_point(alpha = 0.7) +
>   geom_smooth(method = "lm", se = FALSE) +
>   labs(title = "Trend within each species")
>
> # 3
> ggplot(penguins, aes(flipper_length_mm, body_mass_g, colour = species)) +
>   geom_point(alpha = 0.7) +
>   geom_smooth(method = "lm", se = FALSE) +
>   facet_wrap(~ island) +
>   labs(title = "By island and species")
> ```
> **What each reveals**:
> 1. Overall positive relationship — bigger flippers, bigger birds.
> 2. Within each species the slope is much shallower than the pooled slope. This is an aggregation effect (related to, but not a reversal and therefore not a full Simpson's paradox): species differ in *both* flipper length and mass, so pooling exaggerates the slope. This is exactly the intuition we need for adjusted associations in Module 4.
> 3. Species aren't evenly distributed across islands — Gentoo only on Biscoe, Chinstrap only on Dream. Any "island effect" is confounded with species.

---

### Q3 — Write a `describe()` function

Write a function `describe(x)` that returns a named vector or one-row data frame with: `n` (non-missing), `mean`, `sd`, `median`, `IQR`, `n_missing`. Test it on `penguins$body_mass_g`.

> [!tip]- Hint
> `sum(!is.na(x))` for n, `sum(is.na(x))` for missing count. Wrap in `data.frame(...)` so you can `bind_rows` results across variables.

> [!success]- Worked solution
> ```r
> describe <- function(x) {
>   data.frame(
>     n         = sum(!is.na(x)),
>     mean      = mean(x,   na.rm = TRUE),
>     sd        = sd(x,     na.rm = TRUE),
>     median    = median(x, na.rm = TRUE),
>     IQR       = IQR(x,    na.rm = TRUE),
>     n_missing = sum(is.na(x))
>   )
> }
>
> describe(penguins$body_mass_g)
>
> # Bonus: describe all numeric columns at once
> penguins |>
>   select(where(is.numeric)) |>
>   lapply(describe) |>
>   dplyr::bind_rows(.id = "variable")
> ```

---

### Q4 — Read a model with `broom`

Fit `lm(body_mass_g ~ flipper_length_mm, data = penguins)`. Use `broom::tidy()` to get the coefficient table and `broom::glance()` to get the model-level statistics (R², F, etc.). In one sentence, interpret the slope in real units.

> [!success]- Worked solution
> ```r
> library(broom)
> m <- lm(body_mass_g ~ flipper_length_mm, data = penguins)
> tidy(m)
> glance(m)
> ```
> **Interpretation**: Each additional millimetre of flipper length is associated with roughly **50 g** more body mass (slope ≈ 49.7 g/mm), and flipper length alone explains about **76% of variance** in body mass (R² ≈ 0.76). This is a *marginal* — not causal, not species-adjusted — association, and Q2.2 already showed it collapses several within-species effects into one pooled slope.

---

### Q5 — Western blot: from spreadsheet to tidy, normalized data

Read `data/western_blot_example.csv` — this mirrors the shape of data you'd export from Excel after quantifying blots (each row = one lane × one technical replicate). Then:

1. **Inspect** the file with `str()` and `head()`. What are the columns and their types? How many rows per Genotype?
2. **Normalize**: create a new column `Ratio = Target_Intensity / Loading_Control_Intensity`. This is the standard within-lane normalization to loading control (GAPDH, β-actin, total protein — whatever you ran).
3. **Collapse technical replicates**: average the three tech reps within each `(Genotype, Bio_Rep)` combination. The result should be a data frame with **16 rows** — one per biological replicate.
4. **Set factor levels** so WT is the reference: `factor(Genotype, levels = c("WT", "HET", "KO", "RESCUE"))`.
5. **Plot** the normalized ratios: dot-plot (`geom_jitter`) coloured by Genotype, with a mean ± SE bar overlaid. This is the plot you'd put in a paper figure.

Do **not** run the ANOVA yet — we'll do that formally in [[03 - Module 3 - Classical Inference]]. The point of this question is the data-wrangling half of the workflow.

> [!tip]- Hint
> - For step 3, `group_by(Genotype, Bio_Rep) |> summarise(Ratio = mean(Ratio))` collapses tech reps.
> - For step 5, `stat_summary(fun = mean, geom = "crossbar")` and `stat_summary(fun.data = mean_se, geom = "errorbar")` layer nicely on top of `geom_jitter`.

> [!success]- Worked solution
> ```r
> library(dplyr)
> library(ggplot2)
>
> # 1. Read + inspect
> blot <- read.csv("data/western_blot_example.csv")
> str(blot)
> table(blot$Genotype)   # 12 rows per genotype (4 bio × 3 tech)
>
> # 2. Within-lane normalization
> blot <- blot |>
>   mutate(Ratio = Target_Intensity / Loading_Control_Intensity)
>
> # 3. Collapse technical replicates → one value per biological replicate
> blot_bio <- blot |>
>   group_by(Genotype, Bio_Rep) |>
>   summarise(Ratio = mean(Ratio), .groups = "drop")
>
> nrow(blot_bio)   # 16 — this is your n for the ANOVA in Module 3
>
> # 4. Explicit factor levels (WT as reference)
> blot_bio$Genotype <- factor(blot_bio$Genotype,
>                              levels = c("WT", "HET", "KO", "RESCUE"))
>
> # 5. Publication-style dot plot
> ggplot(blot_bio, aes(Genotype, Ratio, colour = Genotype)) +
>   geom_jitter(width = 0.15, size = 3, alpha = 0.8) +
>   stat_summary(fun = mean, geom = "crossbar",
>                width = 0.5, colour = "black") +
>   stat_summary(fun.data = mean_se, geom = "errorbar",
>                width = 0.2, colour = "black") +
>   labs(y = "Target / Loading control (a.u.)",
>        title = "Normalized band intensity by genotype") +
>   theme_bw() +
>   theme(legend.position = "none")
> ```
> **Two things to notice**:
> - Averaging over technical replicates is a defensible move for this balanced one-way design — the biological replicate is your true unit of replication (**n = 4 per group**, not n = 12). Treating tech reps as independent inflates n and gives you a bogus test. In [[08 - Module 8 - Count GLMs and Beyond]] we'll instead model the clustering with a biological-replicate random intercept, which keeps all 48 rows without pretending there are 12 independent biological replicates per group.
> - The `RESCUE` group should visibly return toward WT levels; `KO` should be clearly reduced. Eyeballing this before running the ANOVA is a sanity check that the code did what you expected.

---

## Mini-project (deliverable)

Create `module_01_report.qmd` in this folder. It should render to HTML and contain:

1. A short intro paragraph naming the datasets used.
2. The summary table from Q1.
3. All three plots from Q2 with your one-sentence interpretations.
4. The `broom::tidy()` output from Q4 with a one-sentence interpretation of the slope.
5. The dot plot from Q5 (western blot ratios by genotype) with the tidy code that produced it.

Render with `quarto render module_01_report.qmd`.

## Reflection

After finishing, fill in below.

**What I learned**:

**What still confuses me** (link forward to relevant modules with `[[...]]`):

**Time spent**:

---

Previous: [[00 - Module 0 - R Syntax Primer]]  ·  Next: [[02 - Module 2 - Distributions and Simulation]]

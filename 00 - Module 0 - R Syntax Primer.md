---
title: Module 0 — R syntax primer
tags: [stats, R, module-0, syntax, primer]
module: 0
created: 2026-07-08
---

# Module 0 — R syntax primer

> [!info] Who this is for
> Anyone who's about to work through [[01 - Module 1 - R Foundations]] and wants a quick reference for R's syntax quirks first. Skim if you're comfortable; return to it any time later modules use something you don't recognize.

Everything below is a **reference** — no need to memorize. The goal is to make sure when you see something like the pipeline below, you can parse what's happening.

```r
blot |> filter(Genotype == "WT") |> pull(Ratio)
```

## 1. Assignment: arrow vs equals

```r
x <- 5     # standard R style
x = 5      # works, but equals is also used for named function arguments
5 -> x     # legal but weird; nobody does this
```

The arrow is R's standard assignment operator. Save the equals sign for named arguments inside function calls. In RStudio, use the keyboard shortcut Alt (option on Mac) with the minus k to type the arrow.

## 2. Basic data types

| Type | Example | Test |
|------|---------|------|
| numeric | `1.5`, `3` | `is.numeric(x)` |
| integer | `3L` (the L is deliberate) | `is.integer(x)` |
| character | `"hello"` | `is.character(x)` |
| logical | `TRUE`, `FALSE` (or `T`, `F` — but avoid, they can be reassigned) | `is.logical(x)` |
| factor | `factor(c("A", "B"))` — categorical with defined levels | `is.factor(x)` |
| NA | missing value; **type-specific**: `NA_real_`, `NA_character_` | `is.na(x)` |

Check any object with `class(x)` or the more detailed `str(x)`.

## 3. Vectors — the atomic unit

Almost everything in R is a vector.

```r
x <- c(2, 4, 6, 8, 10)    # combine into a vector
length(x)                  # 5
x + 1                      # 3 5 7 9 11  — operations are vectorized
x * 2                      # 4 8 12 16 20
x > 5                      # FALSE FALSE FALSE TRUE TRUE
mean(x); sd(x); sum(x)     # aggregate functions
```

**Recycling**: shorter vectors are recycled to match longer ones — usually a feature, occasionally a bug.

```r
c(1, 2, 3, 4) + c(10, 20)  # 11 22 13 24 — the shorter vector cycles
```

## 4. Subsetting operators (single bracket, double bracket, dollar)

The single most common source of R confusion. Study this table.

```r
x <- c(a = 10, b = 20, c = 30, d = 40)

x[2]           # named subset, position 2 → keeps name → b = 20
x[c(1, 3)]     # multiple positions
x[-1]          # everything EXCEPT position 1
x["b"]         # by name
x[x > 15]      # by logical → 20 30 40
```

For lists and data frames:

```r
df <- data.frame(x = 1:3, y = c("a", "b", "c"))

df[1, ]        # first row, all columns → returns a data frame
df[, 1]        # all rows, first column → returns a vector
df[1, 2]       # single cell
df$x           # column x as a vector (base R idiom)
df[["x"]]      # same as df$x — programmatic access when name is stored in a variable
df["x"]        # single brackets → returns a one-column data frame
```

**Rule of thumb**: `$` and `[[...]]` pull a column out as a vector; a single `[...]` keeps you in the data-frame world.

## 5. Data frames

A data frame is a list of **equal-length vectors** — one per column. This is the object every stats function expects.

```r
df <- data.frame(
  Genotype = c("WT", "WT", "KO", "KO"),
  Value    = c(1.0, 1.1, 0.4, 0.5)
)

nrow(df); ncol(df); dim(df)
colnames(df)
head(df); tail(df)
str(df)        # structure — always run this after loading data to confirm
summary(df)    # quick per-column summary
```

Tibbles (from `tibble` / `tidyverse`) are data frames with slightly nicer defaults — treat them the same for our purposes.

## 6. Factors (READ THIS)

A factor is a categorical variable with a defined set of **levels**. Level order matters: with R's default treatment contrasts, regression uses the first level as the reference.

```r
g <- c("WT", "KO", "HET", "WT")
factor(g)                                        # levels ordered alphabetically: HET KO WT
factor(g, levels = c("WT", "HET", "KO"))         # explicit order — WT is reference

# Fix an existing column
df$Genotype <- factor(df$Genotype, levels = c("WT", "HET", "KO", "RESCUE"))

# Drop unused levels after filtering
df2 <- df |> subset(Genotype %in% c("WT", "KO"))
df2$Genotype <- droplevels(df2$Genotype)
```

**Why this matters**: if you don't set `WT` as level 1, R will pick alphabetically (`HET` becomes the reference), and every ANOVA coefficient you interpret will be relative to the wrong baseline.

## 7. Formulas — the y ~ x syntax

Statistical formulas are their own mini-language.

```r
y ~ x                # y explained by x
y ~ x + z            # y explained by x and z, main effects only
y ~ x * z            # main effects + interaction (expands to x + z + x:z)
y ~ x : z            # ONLY the interaction, no main effects (rarely what you want)
y ~ .                # y explained by all other columns in `data`
y ~ x - 1            # remove intercept
log(y) ~ x           # transformations allowed on either side
```

Nearly every stats function takes `formula` + `data` args:

```r
lm(len ~ supp * dose, data = ToothGrowth)
t.test(Ratio ~ Genotype, data = two_groups)
aov(Ratio ~ Genotype, data = blot_bio)
```

## 8. The pipe operators

Pipes let you read left-to-right instead of nesting inside-out.

```r
# Nested — hard to read
head(arrange(filter(mtcars, hp > 100), mpg), 5)

# Native pipe (R 4.1+) — reads like a recipe
mtcars |>
  filter(hp > 100) |>
  arrange(mpg) |>
  head(5)

# magrittr pipe (older, from `dplyr`/`magrittr`) — same idea, more features
mtcars %>%
  filter(hp > 100) %>%
  arrange(mpg) %>%
  head(5)
```

**Which to use**: the native pipe is now built into R and slightly faster. The magrittr pipe is more flexible (e.g. supports `.` as a placeholder). For this course, prefer the native pipe unless you specifically need magrittr features.

## 9. Writing functions

```r
describe <- function(x) {
  data.frame(
    n    = sum(!is.na(x)),
    mean = mean(x, na.rm = TRUE),
    sd   = sd(x,   na.rm = TRUE)
  )
}

describe(mtcars$mpg)
```

- Arguments can have defaults: `function(x, na.rm = TRUE) { ... }`.
- Last expression is returned automatically (`return()` is optional).
- `...` collects extra arguments to pass along:
  ```r
  my_mean <- function(x, ...) mean(x, ...)
  my_mean(c(1, 2, NA), na.rm = TRUE)
  ```

## 10. NA (missing values)

Missing values are viral: any arithmetic with `NA` returns `NA`.

```r
mean(c(1, 2, NA))              # NA
mean(c(1, 2, NA), na.rm = TRUE) # 1.5

is.na(c(1, NA, 3))              # FALSE TRUE FALSE
sum(is.na(x))                   # count NAs
x[!is.na(x)]                    # keep only non-NAs
```

**Do not** test with `x == NA` — that returns `NA`, not `TRUE`. Use `is.na(x)`.

## 11. Packages: install once, load per session

```r
install.packages("dplyr")   # once, ever
library(dplyr)              # each session you use it
```

To call a function without loading its whole package, use `::`:

```r
broom::tidy(fit)            # no library(broom) needed
```

This is useful when two packages have functions with the same name (e.g. `dplyr::filter` vs `stats::filter`) — `::` disambiguates.

## 12. Control flow (use sparingly)

```r
# if / else
score <- 2
if (score > 0) "positive" else "not positive"

# for loop — rarely needed in R, vectorization is usually cleaner
result <- numeric(10)
for (i in 1:10) result[i] <- i^2

# vectorized equivalent — this is the idiomatic R
result <- (1:10)^2
```

**Rule of thumb**: if you're writing a `for` loop over rows of a data frame, there's almost certainly a vectorized or `dplyr` way that's shorter, faster, and clearer.

## 13. Getting help

```r
?mean          # help page for mean()
?"["           # help on the subsetting operator (quotes needed for symbols)
??"anova"      # fuzzy search across installed packages
vignette("dplyr")   # long-form tutorials for a package
```

For anything you don't recognize in the course: `?` it first.

## 14. Common gotchas

| Gotcha                                        | Fix                                                                |
| --------------------------------------------- | ------------------------------------------------------------------ |
| Numeric columns import from CSV as character  | `str(df)` — check types, use `readr::read_csv` or `type.convert()` |
| Factor levels wrong reference (alphabetical)  | `factor(x, levels = c(...))` explicitly                            |
| `mean(x)` returns NA                          | Add `na.rm = TRUE`                                                 |
| Single `[...]` returns a data frame, not a vector | Use `[[...]]` or `$` for a vector                              |
| Comparing to NA with `==` returns NA          | Use `is.na()`                                                      |
| Row names silently dropped after subsetting   | Move IDs into a real column                                        |
| `T` and `F` reassigned to something silly     | Always type `TRUE` / `FALSE`                                       |

## Practice

Short warm-ups. If any of these take more than a minute, that's the section of this primer to reread.

---

### Q1 — Vector subsetting

Given the vector below, produce these results (each in one line):

```r
x <- c(a = 10, b = 20, c = 30, d = 40, e = 50)
```

1. The element named `"c"`.
2. All elements greater than 25.
3. Every element except `"a"`.
4. Elements at positions 2 and 4.

> [!success]- Solution
> ```r
> x <- c(a = 10, b = 20, c = 30, d = 40, e = 50)
>
> x["c"]           # named
> x[x > 25]        # logical
> x[-1]            # negative indexing
> x[c(2, 4)]       # positional
> ```

---

### Q2 — Data frame access, four ways

Given the data frame below, extract the `value` column as a **vector** using four different syntaxes.

```r
df <- data.frame(id = 1:5, value = c(2.1, 3.4, 5.6, 7.8, 9.0))
```

> [!success]- Solution
> ```r
> df <- data.frame(id = 1:5, value = c(2.1, 3.4, 5.6, 7.8, 9.0))
>
> df$value
> df[["value"]]
> df[, "value"]
> df[, 2]
> ```
> Note: `df["value"]` and `df[, "value", drop = FALSE]` return a **one-column data frame**, not a vector. This distinction bites people constantly.

---

### Q3 — Factors and reference levels

You have the vector below. Turn it into a factor with **WT as the reference level** and the order WT → HET → KO → RESCUE. Then confirm the ordering with `levels()`.

```r
g <- c("KO", "WT", "HET", "RESCUE", "WT", "KO")
```

> [!success]- Solution
> ```r
> g <- c("KO", "WT", "HET", "RESCUE", "WT", "KO")
> gf <- factor(g, levels = c("WT", "HET", "KO", "RESCUE"))
> levels(gf)
> ```

---

### Q4 — Chain with the pipe

Using `mtcars` and the native pipe, write **one** pipeline that:

1. Filters to cars with `cyl == 6`,
2. Selects columns `mpg`, `hp`, `wt`,
3. Sorts descending by `mpg`,
4. Returns the top 3 rows.

> [!tip]- Hint
> `dplyr::filter`, `dplyr::select`, `dplyr::arrange` with `desc()`, `head()`.

> [!success]- Solution
> ```r
> library(dplyr)
>
> mtcars |>
>   filter(cyl == 6) |>
>   select(mpg, hp, wt) |>
>   arrange(desc(mpg)) |>
>   head(3)
> ```

---

### Q5 — Write a function

Write a function `mean_ci(x, level = 0.95)` that returns a named vector with the mean of `x`, and the lower and upper bounds of a Normal-approximation confidence interval at the given level. Handle NAs. Test on `rnorm(30, 10, 2)`.

> [!success]- Solution
> ```r
> mean_ci <- function(x, level = 0.95) {
>   x <- x[!is.na(x)]
>   m  <- mean(x)
>   se <- sd(x) / sqrt(length(x))
>   z  <- qnorm(1 - (1 - level) / 2)
>   c(mean = m, lower = m - z * se, upper = m + z * se)
> }
>
> set.seed(1)
> mean_ci(rnorm(30, 10, 2))
> ```

---

### Q6 — Read a formula

Without running any code, say in one sentence each what these formulas mean:

1. `y ~ x + z`
2. `y ~ x * z`
3. `y ~ . - id`
4. `log(y) ~ x - 1`

> [!success]- Solution
> 1. `y` explained by main effects of `x` and `z` (no interaction).
> 2. `y` explained by main effects **and** the `x:z` interaction.
> 3. `y` explained by all other columns in `data` **except** `id`.
> 4. `log(y)` explained by `x`, with **no intercept** (the fitted value of `log(y)` is forced through 0 when `x = 0`).

---

## No mini-project for Module 0

Skim it, do the six warm-ups, then dive into [[01 - Module 1 - R Foundations]].

## Reflection

**What I learned / had to look up**:

**Anything that still feels magic** (flag it — you'll meet it again):

---

Next: [[01 - Module 1 - R Foundations]]

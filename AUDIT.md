# Course audit

Audit date: 2026-09-10

## Scope

The review covered all course notes and the included western-blot CSV for:

- statistical definitions, assumptions, interpretations, and model-selection advice;
- agreement between questions, worked solutions, and current package datasets;
- R syntax and required packages;
- replicate handling, multiplicity, uncertainty, and causal-language pitfalls.

## Verification

- All 120 fenced R blocks parse successfully under R 4.5.1.
- Cross-module numerical smoke tests passed for the included CSV, `mtcars`, `ToothGrowth`, `HairEyeColor`, `palmerpenguins::penguins`, `ISLR2::Default`, `MASS::quine`, `warpbreaks`, and the mixed-model example.
- The included CSV has 48 complete rows, 16 biological replicates, and the documented balanced 4 × 4 × 3 structure.
- Package-dependent checks were run from a temporary library; no project library or lockfile was created.
- Quarto was not available in the audit environment, so end-to-end `.qmd` rendering could not be tested. The repository contains no completed `.qmd` deliverables to render.

## Material corrections

- Repaired the CLT solution so the Normal reference curve appears in all three facets.
- Corrected several answer-key values: penguin interaction, `quine` Sex effect and LRT, KO/WT log-scale ratio, ROC sensitivity, warp-break tension results, VIF sequence, and mixed-model variance/SE comparisons.
- Separated deviance explained from McFadden's pseudo-R² and supplied the correct computations for both.
- Completed the installation list with packages used by required exercises and capstone options.
- Replaced categorical rules that were too absolute: assumption-test p-values, expected cell counts, interaction "main effects," VIF deletion, bootstrap intervals, overdispersion, zero inflation, and mixed-model p-values.
- Removed unsupported causal language and added same-row requirements for model comparison and validation-data requirements for ROC and threshold performance.
- Clarified that averaging technical measurements is valid for the balanced blot example and that a random-intercept model is a flexible alternative, not automatically more precise.
- Corrected the reproducibility guidance: `sessionInfo()` records versions; an `renv` lockfile restores them.

## Remaining boundaries

The course is now internally consistent and suitable as an introductory applied primer, but no primer can make a real analysis valid without a defensible sampling design. In particular, real western-blot work may require explicit blot/batch effects, randomization or blocking, preregistered exclusions, and enough independent biological replicates. Survival analysis, missing-data methods, causal inference, and detailed calibration are signposted rather than taught in depth.

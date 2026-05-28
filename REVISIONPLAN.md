# Revision Plan

## Purpose

This plan updates the chromosome-level admixture formatter so the tool, the embedded
about text, and the standalone about page all describe the same model. The current
main tool appears newer than the standalone about page: it now reports Mean, Pulse,
Stability, Sparsity, and standardized scores, while the standalone page still
describes an older adjusted-max and variance-based stability formulation.

The goal is to make the tool more honest theoretically, more stable numerically,
and more useful visually without implying that chromosome-level calculator output
can directly date admixture events.

## Guiding Interpretation

These metrics should be framed as descriptive summaries of between-chromosome
distribution, not direct estimates of tract age. GEDmatch calculator output gives
per-chromosome ancestry percentages, not segment coordinates or tract lengths. That
means Pulse can suggest top-heavy concentration, Stability can suggest smoothness,
and Sparsity can suggest concentration across few chromosomes, but none of them
should be described as literal recombination clocks.

## Theory Revisions

### Pulse

Replace the current spike-ratio interpretation with an excess-spike interpretation.
The current form:

```text
(top_n_mean / mean) / log2(100 / mean)
```

keeps a perfectly even row above zero because `top_n_mean / mean = 1`. A cleaner
version is:

```text
((top_n_mean / mean) - 1) / log2(100 / mean)
```

This gives an even distribution a natural Pulse baseline of zero. Positive values
then mean the top chromosomes are elevated relative to the row's own mean.

Keep the logarithmic denominator only as a dilution-inspired normalization. The
documentation should say it reduces the tendency of tiny mean components to look
dramatic when they have a single high chromosome. It should not say the denominator
estimates generations since admixture.

### Stability

Replace `mean / CV` with a residual stability score that separates smoothness from
component size. Mean is already shown directly and used as bubble size in the plot,
so Stability should answer a narrower question: is this component smoother or
spikier than expected for its overall mean?

Recommended formulation:

```text
raw_smoothness = -log(CV + eps)
mean_signal = log(mean + eps)
fit raw_smoothness = a + b * mean_signal
residual_stability = raw_smoothness - predicted_raw_smoothness
```

Then standardize `residual_stability` across populations in the current run. A
positive residual means smoother than expected for the component's mean; a negative
residual means spikier than expected for the component's mean.

Implementation notes:

- Use a small positive `eps`, for example `1e-9`, to prevent division and log
  instability.
- If there are too few nonzero rows to fit a regression, fall back to
  `-log(CV + eps)` and label the score as raw smoothness.
- Consider a robust fit later if outliers dominate the regression.

### Sparsity

Keep Hoyer sparsity as the basic concept, because it maps well to the question
"is ancestry mass concentrated on few chromosomes?" Recheck the weighted sparsity
implementation before treating it as theoretically settled. Dividing chromosome
values by SNP count may make a uniform percentage row look less uniform simply
because chromosomes have different SNP counts.

Candidate alternatives:

- Compute Hoyer sparsity on raw chromosome percentages even in weighted mode.
- Compute sparsity on chromosome mass, `value * weight`, only if the intended
  question is "where is total ancestry mass concentrated?"
- Show weighted mean/CV separately from sparsity until the weighting theory is
  clear.

## Results and Numerical Fixes

1. Set `EPS` to a real positive value instead of zero.
2. Prefer Yeo-Johnson standardization for metrics that can be zero, or add a
   strictly positive shift before Box-Cox.
3. Remove the unexplained `/ 2` applied to displayed Pulse, or document exactly
   why the display scale differs from the score scale.
4. Preserve original chromosome numbers after dropping all-zero columns. Avoid
   renumbering kept columns as `Chr 1..N`.
5. Add a low-signal flag for tiny mean components, such as below `0.25%` or
   `0.5%`, so the chart does not invite overinterpretation of noise.
6. Fix stale table and test indices after the column-order changes.
7. Add the missing test output container or replace the browser-only test harness
   with a small deterministic test function that can be run from the console.
8. Include the Box-Cox/Yeo-Johnson lambdas for all standardized metrics if those
   values remain part of the output metadata.

## Documentation Revisions

Update both the embedded about section and `chromoadmixabout.html` to describe the
same model:

- Mean: the overall per-population component average.
- Pulse: excess top-chromosome concentration after mean normalization.
- Stability: residual smoothness after accounting for mean.
- Sparsity: Hoyer-style concentration across chromosomes.
- Standardized scores: within-run comparisons, not absolute universal scores.

The limitations section should explicitly state:

- Calculator components are estimates, not direct ancestry segments.
- Closely related reference populations can move together.
- Per-chromosome summaries cannot recover tract length or exact admixture timing.
- Low mean components should be treated as exploratory unless supported by other
  evidence.

## Display Plan

### Primary View: 2D Scatter

Keep the 2D scatter as the main interpretive chart:

```text
x = Pulse Score
y = Residual Stability Score
size = Mean
color = Sparsity Score
```

Improve it with:

- A real color scale/colorbar for sparsity.
- Clear hover text showing raw mean, raw sparsity, and standardized scores.
- Optional label toggles to reduce clutter.
- Visual marking for low-signal components.
- Correct spelling and quadrant text.

### Add a Heatmap Before 3D

Add a chromosome heatmap as the next major visual. Rows should be populations,
columns should be original chromosome numbers, and color should be chromosome
percentage. This gives users a direct view of the pattern behind the metrics and
helps prevent over-trusting the summary plot.

### Optional 3D View

Add 3D only as an optional exploration tab, not as the default view:

```text
x = Pulse Score
y = Residual Stability Score
z = Sparsity Score
size = Mean
```

3D can help users see the full metric space, but it can also make comparisons,
labels, and mobile use harder. It should complement the 2D chart and heatmap,
not replace them.

## Suggested Implementation Order

1. Align both about documents with the current metric set.
2. Fix numerical safeguards and stale table/test indices.
3. Implement excess Pulse.
4. Implement residual Stability with fallback behavior.
5. Recheck weighted sparsity and choose a documented interpretation.
6. Add low-signal flags.
7. Add the chromosome heatmap.
8. Add optional 3D exploration after the 2D and heatmap views are solid.

## Acceptance Criteria

- The tool and both about sections describe the same formulas.
- Even chromosome distributions have near-zero Pulse.
- Stability is not merely a proxy for larger Mean.
- Zero and tiny values do not produce unstable transforms.
- Chromosome columns retain their original numbers.
- Low-signal components are visibly marked.
- The scatter plot remains readable and gains a colorbar or equivalent legend.
- The heatmap lets users inspect the chromosome-level evidence behind each score.

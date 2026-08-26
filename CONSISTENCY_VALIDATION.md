# End-to-end consistency validation — guide extraction benchmark

Minimal closed loop: **K562 lane01 + simpleaf k=15 → OmniBenchmark extraction →
OmniBenchmark metrics**, cross-checked against the original perturb-benchmark
evaluator on identical inputs. Date: 2026-08-13.

## Setup
- Reference (correct, pair-level): `literature_raw_mex` (Replogle 2022 K562 Day6,
  686,464 cells × **2,291 Guide Pair**), materialised to
  `/data/yunzliu/references/published/K562ess_literature_pairs.h5ad`.
- Guide mode: `dual`. Pipeline output: 12,558 cells × 4,536 guides.

## Root cause found & fixed
Within the extraction workflow the IDs are self-consistent (no problem), but at the
**evaluation boundary** they mismatched: `feature_reference_adapter.py` rewrites
guide feature IDs via `[^a-zA-Z0-9_-] -> "_"` (`...538.23-P1P2` → `...538_23-P1P2`),
while the guide CSV keeps raw sgIDs, so `sg2pair` (built from raw sgIDs) mapped only
218/4536 features → all pipeline-side metrics degenerated to 0/NaN.

**Fix (evaluation side only; extraction unchanged):** `load_guide_pair_mapping`
now registers both the raw and the adapter-sanitised sgID spelling as `sg2pair`
keys (metrics module commit `e9b8ef1`). Guide→pair mapping restored.

Effect:

| metric                     | before fix | after fix |
|----------------------------|-----------:|----------:|
| pseudobulk_spearman_rho    |     −0.018 | **0.952** |
| per_cell_spearman_rho_median|    −0.091 | **0.565** |
| pct_rho_negative           |        84% |      5.7% |
| median_guides_pipe         |          0 |        15 |
| umi_corr_pearson           |       0.12 | **0.934** |
| pct_umi_eq_1               |      86.3% |     89.1% |

## Consistency result — PASS
- **Extraction:** OB-produced MEX == direct-module MEX, bit-identical
  (12,558 × 4,536, nnz 215,427).
- **Metrics:** OB metrics module output == original `benchmark_extraction.py`
  output on identical inputs — **all fields identical** (with `PYTHONHASHSEED=0`;
  without it, one field `per_cell_spearman_rho_mean` differs at ~1e-9 from
  float32 summation order, i.e. Python hash-seed nondeterminism, not an OB effect).
- **End-to-end via OmniBenchmark:** `ob run` (data→extraction→metrics, 4 jobs)
  produced pseudobulk ρ = 0.9518054909755428, per-cell ρ median = 0.5649327,
  UMI Pearson = 0.9343480 — identical to the deterministic direct-module run.

## Verdict
Guide extraction benchmark is OmniBenchmark-ised and passes end-to-end consistency
validation: standardization changes nothing in the computation, and with the correct
reference the metrics are strong and reproducible.

## Note
The same `sg2pair` sanitisation fix should be upstreamed to
`perturb-benchmark/lib/loaders.py` if that repo's evaluator remains in use (its
copy was left untouched here; only the OmniBenchmark module was fixed).

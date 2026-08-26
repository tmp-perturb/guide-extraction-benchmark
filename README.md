# Guide Extraction Benchmark

> OmniBenchmark benchmark for sgRNA guide extraction (K562 / Replogle 2022).
> simpleaf + HAM; `data → guide_extraction → metrics`.

## Stages / module repos

| stage | repo | role |
|-------|------|------|
| `data` | [guide-extraction-data](https://github.com/yunzhe-liu/guide-extraction-data) | materialise inputs + reference into the DAG |
| `guide_extraction` | [guide-extraction](https://github.com/yunzhe-liu/guide-extraction) | reference → whitelist → quant → merge (simpleaf / HAM) |
| `metrics` | [guide-extraction-metrics](https://github.com/yunzhe-liu/guide-extraction-metrics) | pseudobulk + per-cell vs reference matrix |

Module commits are pinned in `benchmark.yaml`.

## DAG

`data → guide_extraction [5 configs] → metrics`  (1 → 5 → 5)

## Configs (parameters on one module)

simpleaf k13 / k15 / k17 · simpleaf-knee · HAM  (Cell Ranger excluded)

## Run

```bash
ob validate plan benchmark.yaml
ob run benchmark.yaml --cores 4
```

Software: one conda env (`envs/guide_extraction.yaml`); HAM pinned via git.

## Validation

Reproduces the standalone extraction benchmark bit-for-bit; simpleaf k15 vs the
literature pair reference → pseudobulk ρ = 0.95. See `CONSISTENCY_VALIDATION.md`.

## Scope

Guide extraction + extraction-level metrics only — the upstream complement of the
omni-perturb guide-assignment benchmark. Dataset: K562 / Replogle 2022 (lane01).

## Docs

`METHODS.md` · `CONSISTENCY_VALIDATION.md`

# Guide Extraction Benchmark (OmniBenchmark) — Methods & Overview

The authoritative accompanying documentation for the OmniBenchmark-ised guide
extraction benchmark. It re-orchestrates the standalone `scprocess-perturb`
extraction workflow and the `perturb-benchmark` extraction evaluation into the
OmniBenchmark `data → methods → metrics` form, **without changing the science**
(consistency validated — see §8).

## 1. What this benchmarks

From per-lane sgRNA FASTQ + a GEX matrix (for the cell-barcode whitelist) + a
guide-library CSV, several extraction configurations produce a cells×guides UMI
count matrix (MEX), which is scored against a published reference guide-pair
matrix at pseudobulk and per-cell level.

## 2. Architecture (DAG)

```
data (K562 lane01) ──► guide_extraction  [5-config sweep] ──► metrics (vs reference)
   emits 5 inputs         simpleaf k13/k15/k17, knee, HAM        pseudobulk + per-cell
   incl. data.reference   → merged MEX trio per config           → {dataset}.scores.json
```

Dry-run: `1 data → 5 extraction → 5 metrics` (11 rules).

## 3. Modules (Git-pinned, external to this repo)

| Stage | Module | Repo @ commit | Role |
|-------|--------|---------------|------|
| data | `guide_extraction_data` | `/home/yunzliu/omnibenchmark_modules/guide_extraction_data` @ `b1e3c50` | Materialise external inputs into the DAG (symlink singles, concat multi-part FASTQ); emits `data.reference` |
| guide_extraction | `guide_extraction` | `/home/yunzliu/omnibenchmark_modules/guide_extraction` @ `c379efa` | Reference gen → whitelist → quant (simpleaf/HAM) → merge → MEX |
| metrics | `guide_extraction_metrics` | `/home/yunzliu/omnibenchmark_modules/guide_extraction_metrics` @ `b01a3ea` | Score MEX vs reference (wraps `benchmark_extraction.py` + `lib/`) |

## 4. Inputs & reference

- **Dataset:** K562 lane01 (Replogle 2022), sgRNA FASTQ = 4 SRR runs; GEX =
  lane01 scprocess `af_counts_mat.h5`; guide CSV = `raw_guides_k562_essential.csv`
  (2,291 pairs).
- **Extraction reference (metrics gold standard):**
  `/data/yunzliu/references/published/K562ess_literature_pairs.h5ad`
  (literature `literature_raw_mex`, 686,464 cells × 2,291 guide pairs).
- **3v3 translation table** supplied via `--translation_table`.

All are referenced by absolute path (external), never copied into the repo.

## 5. Method space (config-driven — no per-method modules)

The `guide_extraction` stage sweeps 5 configs as `parameters` on ONE module:

| Config | key params |
|--------|-----------|
| simpleaf k13 | `method=simpleaf, kmer_length=13, minimizer_length=9` |
| simpleaf k15 | `kmer_length=15, minimizer_length=11` |
| simpleaf k17 | `kmer_length=17, minimizer_length=13` |
| simpleaf knee | `kmer_length=15, use_knee=true` |
| HAM | `method=hash_matcher` |

Common: `tenx_chemistry=3v3, resolution=parsimony-gene, min_umi=1000,
min_genes=500, umi_threshold=1, cb_max_hamming=1`. More variants (minimizer,
resolution) are added the same way — extra parameter dicts, no code change.
**Cell Ranger is intentionally excluded** (heavy external deps; kept only as a
reference output in the archive, not an OB module).

## 6. Metrics (unchanged from perturb-benchmark)

Per config, `metrics` emits `{dataset}.scores.json`:
- **Pseudobulk:** Spearman ρ, log1p MSE/MAE, cell recovery, guide recall.
- **Per-cell:** ρ median/mean, % ρ<0, UMI Pearson, median guides/UMI (pipe & ref),
  tool-only %, UMI=1 %.

Pipeline guides are aggregated to reference pairs via the guide CSV
(`sg2pair`, registering both raw and adapter-sanitised sgID spellings — see §8).
Cells matched by `(16mer, lane)` compound key (handles `-L01` and `-NN`).

## 7. How to run

```bash
conda activate omnibenchmark          # ob CLI (v0.6.0)
ob validate plan benchmark.yaml
ob run benchmark.yaml --cores 1 --dry --dirty    # inspect DAG (no execution)
ob run benchmark.yaml --cores 4 --dirty          # execute the sweep
```

`--dirty` is required while modules are referenced by local path (drop it once
the module repos are published). Software: one conda env kept **in this plan
repo** (`envs/guide_extraction.yaml`, omni-perturb convention) —
simpleaf/piscem/alevin-fry + HAM (pinned wheel) + anndata/scipy. Each stage
declares `resources:` (`cores`/`memory`) as scheduling hints.

## 8. Consistency validation (why we trust the standardization)

Verified end-to-end on K562 lane01 + simpleaf k=15 (see
[CONSISTENCY_VALIDATION.md](CONSISTENCY_VALIDATION.md)):
- **Extraction:** OB MEX == direct-module MEX, bit-identical.
- **Metrics:** OB module output == original `benchmark_extraction.py` on identical
  inputs — all fields identical (`PYTHONHASHSEED=0`).
- With the correct reference: pseudobulk ρ = **0.952**, UMI Pearson = **0.934**.
- A pre-existing feature-ID sanitisation mismatch at the evaluation boundary was
  fixed (sg2pair registers raw + sanitised sgIDs); extraction itself unchanged.

## 9. Scope

**In:** guide extraction + its extraction-level metrics, K562 lane01, 5 configs.
**Out:** Cell Ranger as a module; guide assignment / PGMM-EM; GEX benchmarking;
S3/remote storage; multi-lane and full parameter sweeps (addable via `parameters`).

## 11. Ecosystem context & omni-perturb alignment

The mentor group maintains a companion OmniBenchmark org
[`omni-perturb`](https://github.com/omni-perturb) that benchmarks **guide
assignment** — it is the **downstream complement** of this extraction benchmark.

### 11.1 How the two fit together

```
raw FASTQ ──►  guide extraction (THIS benchmark)  ──►  MEX / counts
                                                          │
                          ┌───────────────────────────────┴───────────────────┐
                          ▼                                                     ▼
        omni-perturb assignment benchmark                    our own assignment benchmark
        (count-data → crispat/kaichi → metrics;              (consumes MEX directly;
         input format: rawcounts.h5mu)                        separate, future)
```

omni-perturb's first stage `count-data` (`replogle22.py` → `rawcounts.h5mu`)
**fetches already-aligned published counts** for the same Replogle 2022
K562_essential data — i.e. it starts exactly where this benchmark ends. We
**produce** those counts by real extraction (simpleaf/HAM); they consume them.
The split is clean and non-overlapping: we exclude assignment (their scope),
they skip extraction (our scope).

### 11.2 Format comparison

Structure and the injected-flag contract are **already identical**
(`data → methods → metrics`; `--output_dir --name --<input-id> --<param>`;
`software_backend: conda`; per-module Git repos; `parameters`-driven sweeps).
Remaining deltas:

| Aspect | omni-perturb | This benchmark | Status |
|--------|--------------|----------------|--------|
| plan/stage/module structure | ✓ | ✓ | aligned |
| entrypoint contract | ✓ | ✓ | aligned |
| env in plan repo (`envs/*.yaml`) | yes | **yes** (adopted) | ✅ aligned |
| per-stage `resources:` | yes | **yes** (adopted) | ✅ aligned |
| entrypoint decl | `{named: script}` | `{default: run.py}` | non-issue (their metrics module also uses `default`) |
| commit pinning | `main` (branch) | pinned SHA | we are stricter; kept |
| env tooling | conda + pixi | conda + vendored wheel | optional (pixi not needed) |
| datasets | `dataset_name: [K562_essential, rpe1]` | single `lane01` | deferred (needs 2nd dataset) |
| code hosting | public GitHub org | local (`--dirty`) | deferred (publish) |
| metrics output | `parquet` + `tsv` | `scores.json` | deferred (format) |
| **data I/O format** | **h5mu / h5ad** | **MEX trio + h5ad ref** | **deferred by design** — see 11.4 |

### 11.3 Alignment already applied (stability-neutral)

- **Env in the plan repo** (`envs/guide_extraction.yaml`) — removes the
  machine-specific absolute env path from the plan.
- **Per-stage `resources:`** (`cores`/`memory`).

Both verified non-breaking (`ob validate` ✓; dry-run unchanged at 11 rules).

### 11.4 Deliberately deferred

- **GitHub publishing** (drop `--dirty`) — operational, on request.
- **Metrics output format** (`parquet`/`tsv`; `parquet` needs `pyarrow`).
- **MEX → h5mu adapter** — **not a defect.** h5mu is what *omni-perturb's*
  assignment consumes; **our own** guide-assignment benchmark consumes MEX, so
  MEX output is correct here. An adapter is only needed to feed our extraction
  output directly into *their* assignment stage — a cross-integration decision
  left for later.
- **Datasets** (`dataset_name` sweep) — same mechanism as the method sweep;
  needs a second dataset staged.

Each deferred item is a scoped, one-command-away change with no hidden blocker.

## 12. Document map

| Doc | Where |
|-----|-------|
| This overview | `benchmark/METHODS.md` |
| Run/README | `benchmark/README.md` |
| Consistency validation | `benchmark/CONSISTENCY_VALIDATION.md` |
| Module CLI contract | `guide_extraction/MODULE_CONTRACT.md` |
| Module validation record | `guide_extraction/VALIDATION.md` |
| Archive overview + data catalog | `guide_extraction_archive/GUIDE_EXTRACTION_ARCHIVE.md`, `DATA_AND_FILES.md` |

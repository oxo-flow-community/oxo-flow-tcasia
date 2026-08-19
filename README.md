# oxo-flow-tcasia — Paired-end RNA-seq alignment and four-caller alternative-splicing analysis

[![CI](https://github.com/oxo-flow-community/oxo-flow-tcasia/actions/workflows/ci.yml/badge.svg)](https://github.com/oxo-flow-community/oxo-flow-tcasia/actions/workflows/ci.yml)

> ★ Verified · ⇄ Official port of [`OncoHarmony-Network/TCASIA_pipeline`](https://github.com/OncoHarmony-Network/TCASIA_pipeline) @ `06564ff1`
> — same tools, same versions, same commands. Part of the
> [oxo-flow-community catalog](https://oxo-flow-community.github.io/).

Paired-end RNA-seq from FASTQ to per-sample alternative-splicing calls:
reads are trimmed and filtered with fastp, aligned with two-pass STAR and
counted per gene with featureCounts; each sample's splicing is then
quantified independently with four callers — rMATS, MAJIQ (with Voila
export), SUPPA2 (via Salmon transcript quantification) and SplAdder.

The two upstream Snakemake workflows are chained in one DAG, so alignment
BAMs feed the callers automatically. Run one stage only with
`oxo-flow run main.oxoflow -t alignment` / `-t as_calling` (rule names are
namespaced `alignment::*` / `as_calling::*`).

## Installation

### 1. Install oxo-flow

Requires **oxo-flow ≥ 0.12.0**. Release binary (recommended):

```bash
curl -fL -o oxo-flow.tar.gz \
  https://github.com/Traitome/oxo-flow/releases/latest/download/oxo-flow-latest-x86_64-unknown-linux-gnu.tar.gz
tar xzf oxo-flow.tar.gz
sudo mv oxo-flow /usr/local/bin/
```

Alternative: `conda install -c bioconda oxo-flow-cli` (the bioconda package
may lag behind releases; other platform binaries are on the releases page).

### 2. Get this workflow

```bash
git clone https://github.com/oxo-flow-community/oxo-flow-tcasia.git
```

### 3. Requirements

- **Reference data** (GRCh38 primary assembly + GENCODE v34, as upstream):
  - STAR index built with STAR 2.7.7a — `star_index_dir`
  - annotation GTF — `ref` (featureCounts, rMATS, SplAdder)
  - annotation GFF3 — `gff` (MAJIQ)
  - Salmon transcript index — `salmon_index`
  - SUPPA2 events file — `suppa2_events`
  - MAJIQ academic license file — `majiq_license` + `run_majiq = true`
    (the MAJIQ chain is gated on the license flag; see Deviations)
- **Compute**: up to 10 threads per rule (STAR/rMATS), no memory limits set
- **Tools**: conda envs with pinned versions (fastp 0.23.4, STAR 2.7.7a,
  samtools 1.13/1.15, subread 2.0.1, salmon 1.10.3, suppa 2.3, rMATS 4.3.0,
  MAJIQ 2.5, SplAdder 3.1.1; conda-forge + bioconda) — the 8 upstream
  `envs/*.yaml` files, verbatim; conda or mamba at runtime

## Usage

```bash
# 1. put paired reads at <reads_dir>/<sample>_1.fastq.gz / <sample>_2.fastq.gz
#    and list samples in the [[sample_groups]] block of main.oxoflow
#    (see test/fixtures/raw for the expected layout)
# 2. edit the [config] reference paths in main.oxoflow
# 3. preview the plan
oxo-flow dry-run main.oxoflow
# 4. run (conda envs are created on first use)
oxo-flow run main.oxoflow -j 16
```

Primary outputs:

```text
{align_out_dir}/aligned/{sample}/{sample}_Aligned.sortedByCoord.out.bam(.bai)
{align_out_dir}/aligned/{sample}/{sample}_featureCounts.txt
{as_out_dir}/{sample}/rmats/       {as_out_dir}/{sample}/majiq/
{as_out_dir}/{sample}/suppa2/      {as_out_dir}/{sample}/spladder/
```

## Fidelity

| Upstream rule | oxo-flow rule | Tool (version) | Notes |
|---|---|---|---|
| fastp_qc | `alignment::fastp_qc` | fastp 0.23.4 | identical command; input layout from `[[sample_groups]]` + `reads_dir` instead of samples.tsv |
| star_align | `alignment::star_align` | STAR 2.7.7a | identical command; `params.prefix` inlined |
| sort_bam | `alignment::sort_bam` | samtools 1.15 | identical command |
| index_bam | `alignment::index_bam` | samtools 1.15 | identical command |
| featurecounts | `alignment::featurecounts` | subread 2.0.1 | identical command; upstream runs without a strandness flag (oxo-flow preflight warns — upstream behavior kept) |
| salmon_quant | `as_calling::salmon_quant` | salmon 1.10.3 | identical command; `-l` from explicit `salmon_library_type` (upstream derives it from `strandness` in tcasia_config.py) |
| select_suppa_fields | `as_calling::select_suppa_fields` | suppa 2.3 | identical command |
| format_suppa_fields | `as_calling::format_suppa_fields` | suppa 2.3 | identical perl one-liner |
| suppa_run | `as_calling::suppa_run` | suppa 2.3 | identical command; output prefix inlined |
| rmats_create_input | `as_calling::rmats_create_input` | rMATS 4.3.0 | identical command |
| rmats_run | `as_calling::rmats_run` | rMATS 4.3.0 | identical command; `--od` directory declared as the rule output |
| majiq_create_ini | `as_calling::majiq_create_ini` | MAJIQ 2.5 | identical printf; `bam_dir`/`bam_stem` params inlined |
| majiq_build | `as_calling::majiq_build` | MAJIQ 2.5 | identical command |
| majiq_psi | `as_calling::majiq_psi` | MAJIQ 2.5 | identical command |
| voila_modulize | `as_calling::voila_modulize` | MAJIQ 2.5 (Voila) | identical command; `modulized/` directory declared as the rule output |
| voila_tsv | `as_calling::voila_tsv` | MAJIQ 2.5 (Voila) | identical command |
| spladder_run | `as_calling::spladder_run` | SplAdder 3.1.1 | identical command; output directory declared as the rule output |
| rule all | — (DAG targets) | — | every output above is a default target of the single chained DAG |

Deviations from upstream defaults, all recorded here:

- **Sample sheet**: upstream reads per-sample fastq paths from a TSV
  (`sample_id/fastq_1/fastq_2`); the port uses `[[sample_groups]]` plus the
  `reads_dir/{sample}_1.fastq.gz` / `{sample}_2.fastq.gz` layout.
- **One workflow file, one DAG**: the two upstream Snakefiles (+ the four
  `rules/snakefile_*` fragments) are `modules/alignment.oxoflow` +
  `modules/as_calling.oxoflow`, included from `main.oxoflow`; the two
  `config.yml` files merge into one `[config]`. Upstream `01 output_dir/aligned`
  and `02 bam_dir` are the single key `aligned_dir`, making the
  alignment → AS-calling chain structural.
- **Strandness-derived values are explicit config keys**: upstream
  computes `salmon_library_type` (`fr-firststrand→ISR`, `fr-secondstrand→ISF`,
  `fr-unstranded→IU`) and `majiq_strandness` (`fr-firststrand→reverse`,
  `fr-secondstrand→forward`, `fr-unstranded→none`) in `tcasia_config.py`;
  rMATS uses the `strandness` value directly (as upstream). Change `strandness`
  **and** the two derived keys together.
- **Helper scripts not ported**: `scripts/validate_config.py` and
  `scripts/read_length.sh` are user-facing helpers; oxo-flow validates
  config/inputs natively (`validate`, `dry-run`).
- **Threads only, no memory**: upstream declares threads per tool and no
  memory; the port mirrors that exactly.
- **MAJIQ is license-gated**: upstream runs the MAJIQ chain unconditionally
  and fails without the academic license file. The port gates all five MAJIQ
  rules (`majiq_create_ini`, `majiq_build`, `majiq_psi`, `voila_modulize`,
  `voila_tsv`) on `run_majiq` — default `false`, so a fresh clone runs the
  rMATS/SUPPA2/SplAdder callers end-to-end; set `run_majiq = true` after
  placing the license at `majiq_license` (commands unchanged when enabled).

## Source

Ported from **[OncoHarmony-Network/TCASIA_pipeline](https://github.com/OncoHarmony-Network/TCASIA_pipeline)**,
commit `06564ff19e7f5d952f5d98175ee6b5d86c08ec49` (main, MIT; no upstream
tags). Created 2026-08-15; this workflow **may lag upstream releases**.
Attribution in `NOTICE.md`.

## Test

```bash
bash test/run.sh   # validate + lint + dry-run, exits 0
```

## License

Apache-2.0. Copyright (c) 2026 oxo-flow-community.

## Community

https://oxo-flow-community.github.io/

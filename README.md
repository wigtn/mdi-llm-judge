# MDI: The Minimum Detectable Improvement of an LLM Judge

Code, data and manuscript for *MDI: The Minimum Detectable Improvement of an
LLM Judge* by Hyeong-seob Kim (WIGTN), presented at the
[TAE (Trust-AI-Eval) workshop](https://tai-eval.github.io/) at NeurIPS 2026
(non-archival). Every number in the paper resolves to a file in this
repository.

- Paper: the PDF is attached to the
  [latest release](https://github.com/wigtn/mdi-llm-judge/releases/latest);
  the LaTeX source is in [`paper/tae2026/`](paper/tae2026/).
- Data: every judge call the paper rests on, the derived records behind
  each figure and table, and the cost ledger, all under [`data/`](data/).
- Citation: [BibTeX below](#citation), or GitHub's "Cite this repository"
  button, which reads [`CITATION.cff`](CITATION.cff).

![MDI by environment and budget](paper/figures/mdi_overlay_alpha0p05.png)

*Figure 1 of the paper: the smallest gain an evaluation can separate from
judge noise depends on the environment and on the number of scoring
repeats N.*

## The question

An LLM judge that scores the same output twice does not, in general, return
the same score. When a pipeline change raises the judge's average by one
point, that point may be an improvement or the judge disagreeing with
itself. For a given evaluation environment (judge model, prompt template,
task, scale, temperature), the paper defines two quantities:

- MDI(env, N), the minimum detectable improvement: the smallest gain the
  environment can confirm at a stated error level α given N scoring repeats
  per system. It is the half-width of the central 95% prediction interval of
  the null distribution, the difference between two independent N-repeat
  scorings of the same systems.
- FIP(Δ, env, N), the false improvement probability: the probability that
  an observed gain Δ reverses under an independent re-evaluation.

## What the paper finds

The grid is pilot-scale (two judges, two tasks, three scales): it establishes
direction, not final magnitudes.

- The threshold is not a constant. At N = 1, MDI ranges from 0.65 to
  1.75 %p across six gpt-4o-mini environments, and going from N = 1 to 5
  roughly halves it (Table 1; `data/derived/exp1_*/mdi_table.json`).
- The same gain carries different risk. On the same summarization cell, a
  1.8 %p gain reverses 7.4% of the time under gpt-4o-mini and 26.7% under
  gpt-5.6 (Tables 2 and 5; `data/derived/exp1_{dense,anchor}/fip.json`).
- Repeats cannot remove everything. Rewording the rubric adds a noise
  floor that more repeats do not average away: 8.48 %p over eight paraphrases
  of one prompt (Section 4.3; `omega` in `data/derived/exp3_decay8/decay.json`).
- Clearing MDI is not expert agreement. The threshold fires on system
  pairs that no expert annotation facet separates, and stays silent on one
  that three of four facets do (Section 4.1;
  `data/derived/exp1_ladder/oracle.json`).

## The protocol (Section 5)

1. Repeat: score each system with N ≥ 3 independent repeats in the stated
   environment.
2. Interval: report a confidence interval for the gain Δ, not a point
   estimate alone.
3. Decide: claim an improvement only when Δ > MDI_α(env, N) on the stated
   item set and FIP(Δ, env, N) ≤ α.

MDI is read before the run, to choose the N a target gain needs (the
right-hand columns of Table 1). FIP is read after it, to price the gain
actually observed (Table 5).

## What is here

| Path | What |
|---|---|
| `src/mdi/`, `tests/` | The framework. Every estimator in the paper lives here. |
| `configs/` | One YAML per experiment. The registry's config hash is the first 16 hex chars of SHA-256 over the canonicalized parsed config: `uv run python -c "from mdi import runner; print(runner.config_hash(runner.load_config('configs/exp1_dense.yaml')))"` reproduces the registry row without side effects. (`mdi estimate-cost` prints it too, but re-writes this config's gate receipt under `data/gates/`; the receipts committed here are the ones the original runs cleared.) |
| `data/raw/` | Every judge call ever made, append-only, one JSONL record per scoring. 97,752 records. That is the registry's 95,522 full-grid calls plus 1,490 pilot passes and 740 spike probes, all rows of `data/ledger.jsonl`. |
| `data/inputs/` | Content-addressed snapshots of the benchmark items, so a re-run scores the same text. |
| `data/derived/` | Analysis outputs. Every number in the paper resolves to one of these files. |
| `data/ledger.jsonl` | Per-run cost and call counts, the source of the run registry table. |
| `data/gates/` | The estimate-cost and pilot records each run had to clear before the full grid. |
| `paper/figures/`, `paper/tables/` | Generated artifacts, never hand-edited. |
| `paper/tae2026/` | The manuscript source. |

## Reproducing the paper

Python 3.11+ and [uv](https://docs.astral.sh/uv/).

```bash
uv sync

# Regenerate every figure and table from the derived records.
uv run mdi report all

# Re-derive those records from the raw scores. No API calls, no cost.
# The flags matter: several analyses default to a narrower sweep than the
# paper used, and every derived file records the parameters it was built with
# under `meta.params`, so you can check any of these against the committed
# output.
uv run mdi analyze variance  --exp exp1_dense
uv run mdi analyze decay     --exp exp3_decay8
uv run mdi analyze fip       --exp exp1_dense --n-repeats 1,5,10
uv run mdi analyze mdi-table --exp exp1_dense --sweep 1,3,5,8,10 --targets-pp 0.5,1,2

# The remaining Table 1 rows come from the sibling grids: repeat with
# --exp exp1_anchor, exp1_anchor_scales, exp1_alpaca, exp1_ladder.
```

Each writes into `data/derived/<exp>/`, overwriting the committed copy with a
byte-identical one. If a file comes out different, diff its `meta.params`
against the committed version first; that is where a wrong flag shows up.

`mdi report all` run twice produces byte-identical output; that is the
determinism claim in the appendix, and it is checkable here:

```bash
uv run mdi report all && cp -r paper/figures /tmp/a && cp -r paper/tables /tmp/b
uv run mdi report all && diff -r /tmp/a paper/figures && diff -r /tmp/b paper/tables
```

Byte-identical is a same-platform claim: the PDFs and tables are stable
across platforms, but PNG bytes re-encode under each platform's zlib while
the pixels stay identical, so compare rasters pixel-wise when crossing
platforms.

To build the paper itself (latexmk if installed, otherwise tectonic):

```bash
cd paper/tae2026 && make
```

Re-issuing the judge calls themselves needs an OpenAI key and money, and goes
through the cost gate the paper describes:

```bash
uv run mdi estimate-cost --config configs/exp3_decay8.yaml
uv run mdi run --config configs/exp3_decay8.yaml --pilot
uv run mdi run --config configs/exp3_decay8.yaml
```

The runner is idempotent on `(env_id, item_id, system_id, repeat_idx)`, so a
resumed run bills only the calls it has not made.

## Tracing a number

Every quantity in the paper resolves to a derived file plus a `run_id`. The
figure and table comments in the manuscript source name both. For example the
noise floor of 8.48 %p is `omega` in `data/derived/exp3_decay8/decay.json`;
its `meta.run_ids` names the three gated passes behind the 16,000 records it
was fit on: the 120-call pilot `r_20260805_6ecb21`, then `r_20260805_a9d7a3`
(9,480 calls) and `r_20260805_523960` (6,400 calls, config hash
`07f436c4afaa217e`), all three rows of `data/ledger.jsonl`; the registry
table carries the two full passes.

## Browsing the raw scores

`data/raw/` holds one JSONL shard per environment, some of them large. If a
directory listing fails to expand, the files open directly:

- `data/raw/exp1_dense/`: the judge-variance grid
- `data/raw/exp3_decay8/e_e941458fd533.jsonl`: the eight-paraphrase sweep (16,000 records)

## Releases

| Tag | Date | What it is |
|---|---|---|
| `tae2026-camera-ready-v2` | 2026-11-03 | Camera-ready revision 2. Two counts in the text corrected (Table 1 caption: at 80% power 5 of the 24 read-offs become unreachable, not 4; Section 3: the power factor is a median over 35 cells, not 36). Figure 3's y-axis label is no longer clipped, Figure 4 drops its provenance stamp, and the repository has its current name. No measured value changed. |
| `tae2026-camera-ready` | 2026-09-30 | First camera-ready. |

Until 2026-11-03 this repository was `wigtn/mdi-artifact`. GitHub redirects
the old address here, so links in the first camera-ready PDF still work.

## Citation

```bibtex
@inproceedings{kim2026mdi,
  title     = {{MDI}: The Minimum Detectable Improvement of an {LLM} Judge},
  author    = {Kim, Hyeong-seob},
  booktitle = {TAE (Trust-AI-Eval): Can We Trust AI Evaluation? Workshop at NeurIPS 2026},
  year      = {2026},
  url       = {https://github.com/wigtn/mdi-llm-judge},
  note      = {Non-archival}
}
```

## License

Code (`src/`, `tests/`, `configs/`) is released under the
[Apache License 2.0](LICENSE). The data, figures, tables and manuscript text
this project produced (`data/raw/`, `data/derived/`, `data/ledger.jsonl`,
`data/gates/`, `paper/`) are released under
[CC BY 4.0](LICENSE-DATA). Benchmark items under `data/inputs/` (SummEval,
AlpacaEval 2.0) and the Regimes release used in Experiment 4 remain under their
original licenses, and `paper/tae2026/neurips_2026.sty` belongs to NeurIPS.

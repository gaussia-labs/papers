# Roast Me

Paper proposing a metric that red-teams a task-specific assistant in two stages: a
**Profiler** that discovers where the assistant is likely to break, and an **Exploiter**
that searches for reproducible *categories* of natural queries that break it.

## Structure

```
2026-06-roastme/
├── roastme.tex / roastme.pdf   # the paper
├── references.bib
└── results/                    # the frozen artifacts behind every number reported
```

**The experiment code does not live here** — it is in
[`gaussia-labs/pygaussia`](https://github.com/gaussia-labs/pygaussia) under
`experiments/roastme/`, so this repository stays a papers repository. The plugin and
strategy catalogue used for the Monotributo scenario lives in `Alquimia-ai/roast-me`.

## What backs each table

| Table | Reports | Artifact |
|---|---|---|
| Engine trade-off | 204 probes, composed | `results/level1_probes/` |
| Generation speed | probes/sec by model family | `results/level1_model_comparison/` |
| Judge config, fail rate by strategy | **151 scoreable probes**, 3 judges | `results/level2_profiler/` |
| Category search | 4 runs, $\tau=0.35$ | `results/level3_exploiter/20260729T164442Z-*` |

Why 151 and not 204: 53 of the probes are **controls** (a real entity, no false premise, no
principle under test). They stay in the frozen transcripts so their raw responses remain
inspectable, but they are excluded from every violation-rate aggregate.

`results/level3_exploiter/` holds twelve directories. The four `20260729T164442Z-*` are the
reported runs (verified judge); the eight `20260721T*` ran with the earlier self-evaluating
judge and are kept as the evidence for why it was replaced. Four directories per timestamp are
the four `accelerate` ranks of one run, not four runs.

**These level-3 records cannot be regenerated**: each run is stochastic RL against a live
target assistant, and the trained adapters no longer exist. The JSONL files are the only
record.

## Compiling the paper

```bash
pdflatex roastme.tex
bibtex roastme
pdflatex roastme.tex
pdflatex roastme.tex
```

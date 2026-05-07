# CLAUDE.md for a Research Codebase

## Project shape

- Notebook-heavy: lots of `.ipynb` files for experiments
- Small but real production code: training scripts, data loaders, evaluation harnesses
- Speed of iteration matters more than long-term maintenance
- Datasets and model artifacts that are too big for git
- A few people work on it; conventions are loose by design

This shape is common in ML/research, data science, quantitative finance, scientific computing.

## The CLAUDE.md

```markdown
# Project Context

This is a research codebase for [project goal]. We're iterating on [methods/models/analyses]. The code here is a mix of notebook experiments, reusable utilities in a small library, and a few production-style scripts for training and evaluation.

This is not a polished software project. The goals are: iterate fast, keep results reproducible enough to trust, and graduate things into the library only when we use them three times.

## Stack

- Python 3.11
- `uv` for dependency management
- numpy, pandas, polars (we're shifting toward polars where possible)
- pytorch for models, lightning where it helps
- jupyter / jupytext for notebooks
- wandb for experiment tracking
- DVC for dataset versioning, results saved to S3

## Repository layout

- `notebooks/` - experimental work; one notebook per question or hypothesis
- `src/<package>/` - the small library: data loaders, model definitions, evaluation utilities
- `scripts/` - top-level scripts that should run from the command line (training, batch eval)
- `experiments/` - configs (YAML) for experiments, organized by date
- `data/` - gitignored, populated via DVC
- `outputs/` - gitignored, model artifacts and results

## Conventions (loose by design)

- Notebooks: name them `YYYY-MM-DD-<topic>-<initials>.ipynb` (e.g. `2025-03-11-loss-spikes-jw.ipynb`). Date-prefixed because order matters for understanding the trail of work.
- Notebook structure: a markdown cell at the top with the question, a section per major step, a "conclusions" markdown cell at the bottom. If you can't fill in conclusions, the notebook isn't finished.
- The `src/` library has stricter rules: type hints, docstrings on public functions, tests. Notebooks don't.
- `scripts/` are reproducible: they take a config file argument and don't have hardcoded paths. If you can't re-run a script with `python scripts/train.py --config experiments/.../foo.yaml` and reproduce the result, fix the script.
- We graduate code from notebooks to `src/` when we use it in 3+ notebooks. Earlier than that is premature abstraction.

## Reproducibility (the rules we don't break)

- Every experiment has a config in `experiments/<date>/<name>.yaml` checked into git
- Every config includes a `seed` field; set it for numpy, torch, and any RNG
- Every training run logs to wandb with the config attached
- Datasets are referenced by DVC hash in the config, never by raw path
- Results that we cite (in writeups, in slides) must come from a script + config combination that can be re-run

## What goes in `src/` vs. notebooks

- In `src/`: anything that's stable enough to import. Loaders, model definitions, evaluation metrics, plotting helpers used multiple times.
- In notebooks: exploration, prototyping, one-off analyses, anything in flux.
- A notebook should import from `src/` aggressively. If a notebook has 200 lines of utility code, that code probably belongs in `src/`.

## Testing

- We test `src/` code, not notebooks
- pytest, fixtures for synthetic data
- We don't aim for high coverage. We test:
  - Data loaders (shapes, dtypes, edge cases like empty splits)
  - Loss functions (numerical correctness, edge cases)
  - Evaluation metrics (against known values)
  - Anything we've gotten wrong before (regression tests)

## Plotting

- matplotlib for one-off plots in notebooks (it's fine, don't overthink it)
- For repeated plots (training curves, evaluation comparisons), helpers in `src/<package>/viz.py`
- Save figures with both PNG and PDF when they're going into a writeup
- For interactive plots, plotly. Don't reach for it unless interactivity adds something.

## Common tasks

- New experiment: copy an existing config in `experiments/`, modify, run via `python scripts/train.py --config <path>`
- Reproduce a published result: find the config, run the script, compare to the wandb run
- Quick sanity check: open a new notebook in `notebooks/`, name it correctly, make it self-contained

## What not to do

- Do not commit large files (model weights, datasets, big plots). They go to S3 / DVC.
- Do not remove old notebooks even if they're "wrong." Add a markdown cell at the top saying what was wrong, and leave the notebook. Trail of work matters here more than a clean directory.
- Do not refactor `src/` aggressively. Stable code is more valuable than elegant code in research.
- Do not add a new dependency without checking if numpy/scipy/sklearn already does what you want. Compile times and conflicts matter here.
- Do not write code in notebooks that should be in `src/`. The "3 uses then graduate" rule is the simple test.
- Do not run training scripts without setting `seed` and logging to wandb. Even quick tests. The 30 seconds you save in setup costs hours later when you need to reproduce.
```

## Decisions and reasoning

**The opening paragraph is unusually long because the *culture* of a research codebase is the most important context.** Claude defaults to "make this clean and well-tested" instincts that work against research velocity. Spelling out the goals (iterate fast, reproducibility-when-it-matters, graduate code only when reused) reframes Claude's defaults.

**Notebook conventions are explicit because notebooks are easy to get wrong.** Date-prefixed names, structured top-and-bottom cells, the "conclusions" requirement. These are tiny rules that compound massively over a year of work.

**The "What goes in `src/` vs. notebooks" rule with the 3-uses heuristic is load-bearing.** Without it, Claude will happily build out abstractions in notebooks that should be one-offs, or leave duplicated code in places that should be in `src/`. The rule decides for it.

**Reproducibility gets its own section with hard rules.** Most research code rots because reproducibility is treated as a soft preference. Listing it as "rules we don't break" makes it stick.

**Testing is described in terms of *what we test*, not coverage targets.** This matches the research reality: full coverage is wasteful, but specific things (data loaders, metrics, regressions) are worth tests. Naming them rules out the "test everything" or "test nothing" extremes.

**The "What not to do" section is heavy on research-specific landmines.** Removing old notebooks, refactoring `src/` aggressively, skipping seeds. These are not generic advice; they're things research codebases regret.

## What's deliberately not included

- **A list of common metrics or methods.** Those live in code and papers, not in CLAUDE.md.
- **Hyperparameter ranges.** Those belong in configs and writeups.
- **Citations or related work.** Belongs in papers and a `docs/literature.md` if you keep one.
- **Polishing/deployment guidance.** This codebase isn't deployed. If something graduates to a deployed system, it leaves this repo and gets a different CLAUDE.md.
- **Aggressive style enforcement.** Black-formatted notebooks are fine but not enforced; the friction isn't worth it. CLAUDE.md says nothing about it.

## How to adapt this to your project

- If your "research" is actually production ML (training pipelines for models that ship), you need a stricter CLAUDE.md, closer to the [Python Data Pipeline](./01-python-data-pipeline.md) one
- The graduation rule (3 uses) is a heuristic; pick a number that fits your team
- Replace experiment tracking tool (wandb) with whatever you actually use (mlflow, comet, none)
- The reproducibility rules are the part most worth keeping verbatim. Every research codebase that stays useful past year 2 has rules like these.

## A note on the "loose by design" framing

Research codebases get into trouble two ways: too strict (so people stop iterating), or too loose (so nothing is reproducible). The framing in this CLAUDE.md is "loose where speed matters, strict where reproducibility matters." That distinction makes Claude's choices align with the team's actual values.

Without that framing, Claude will often default to either:
- "Let me clean this up" (too strict, kills velocity)
- "Whatever, it's a notebook" (too loose, kills reproducibility)

Stating the principle resolves the conflict.

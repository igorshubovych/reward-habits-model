# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is a PhD research project. It contains a NetLogo agent-based model studying how external
rewards affect intrinsic motivation and habit formation ("crowding-out" vs "persistence" effects),
plus BehaviorSpace experiment outputs and Jupyter notebooks that analyze those outputs.

- `reward-habits.nlogox` — the NetLogo model (NetLogo 7.0.4 XML format). Contains the model code,
  interface widgets, and two BehaviorSpace experiment definitions (`controlled-factored-experiment`,
  `reward-duration-experiment`), all in one file.
- `experiments/<n>/` — raw BehaviorSpace table/spreadsheet/stats/lists CSV output from successive
  runs of the model (experiment 1, 2, 3 in chronological order — later experiments refine the
  parameter ranges based on findings from earlier ones).
- `controlled-factored-experiment-analysis.ipynb` — earlier/simpler analysis notebook, reads
  `experiments/1/...-table.csv`.
- `netlogo_factorial_experiment_analysis.ipynb` — the current, more complete analysis pipeline
  (data cleaning, factorial design checks, descriptive stats, heatmaps, interaction plots,
  factorial ANOVA, Random Forest + permutation importance, partial dependence, scenario summaries).
  Reads `experiments/2/...-table.csv`.
- `analysis_output/` — figures and CSV summaries written by the analysis notebook.

**None of the CSV data (`*.csv` anywhere, including `experiments/` and `analysis_output/`) is
committed to git** — it's all regenerated locally by re-running BehaviorSpace experiments and
notebooks. Only the model (`.nlogox`) and notebooks (`.ipynb`) are version-controlled.

## Commands

### Running the model

Open `reward-habits.nlogox` in NetLogo (7.0.4+) to use the interface, or run a BehaviorSpace
experiment headlessly, e.g.:

```bash
netlogo-headless.sh \
  --model reward-habits.nlogox \
  --experiment controlled-factored-experiment \
  --table experiments/<n>/reward-habits-controlled-factored-experiment-table.csv
```

Swap `--experiment` for `reward-duration-experiment` to run the other defined experiment. The exact
`netlogo-headless` launcher path/command depends on the local NetLogo install.

### Running the analysis notebooks

```bash
python -m pip install pandas numpy matplotlib scikit-learn statsmodels
jupyter notebook netlogo_factorial_experiment_analysis.ipynb
```

The notebook explicitly uses `n_jobs=1` for `RandomForestRegressor`/`permutation_importance` to
avoid `joblib`/`loky` issues on macOS with Python 3.13 — keep this if editing that code.

Before running, update `CSV_PATH` near the top of the notebook to point at the experiment folder
you want to analyze (currently `experiments/2/...`), and note that `pd.read_csv(..., skiprows=6)`
is required because BehaviorSpace table CSVs have 6 header/metadata lines before the real column
header row.

## Model architecture (`reward-habits.nlogox`)

The model simulates `shoppers` making repeated binary choices ("target" vs not) across three
sequential phases driven by tick count: `baseline` → `reward` → `withdrawal`
(`setup-experiment-timing`, `update-phase`). Durations are set by `baseline-duration` (fixed
constant) plus the `reward-duration`/`withdrawal-duration` UI sliders.

Each shopper is permanently assigned to one of three treatment groups at creation
(`assign-treatment-group`, by `who mod 3`): `control`, `low-salience`, `high-salience`. Salience
determines how strong the external reward is during the `reward` phase (`low-reward`/`high-reward`
sliders).

Per-tick agent loop (`shopper-step`): `make-choice` → `record-choice` → `update-habit` →
`update-motivation`.

- **Choice**: a logistic function of `intrinsic-motivation`, `habit`, and the current reward effect
  (zero outside the `reward` phase or for `control`), minus a fixed `target-choice-cost`
  (`target-choice-probability`).
- **Habit**: increases toward 1 when the target is chosen (faster when intrinsic motivation and/or
  reward salience are high), decays slowly (`habit-decay-rate`) otherwise (`update-habit`).
- **Intrinsic motivation**: only changes during the `reward` phase, only for non-control agents that
  chose the target. Below `salience-threshold` it's slightly *reinforced*; above it, it's *crowded
  out* proportionally to how far salience exceeds the threshold (`update-motivation`,
  `reinforce-intrinsic-motivation`, `crowd-out-intrinsic-motivation`). This threshold mechanic is the
  central hypothesis the model is built to test.

Choice counts are accumulated per phase per agent (`baseline/reward/withdrawal-opportunities` and
`-target-choices`) and aggregated into group/phase choice rates (`choice-rate`), from which the key
outcome metrics are derived:

- `high-reward-crowding-out-effect` — high-salience vs control withdrawal choice-rate gap (primary
  target metric; negative means crowding-out occurred).
- `low-reward-persistence-effect` — low-salience vs control withdrawal choice-rate gap (secondary
  target metric; positive means the low-reward group persisted better than control).
- `persistence-ratio` / `high-reward-retention-ratio` / `low-reward-retention-ratio` — withdrawal
  choice-rate as a fraction of during-reward choice-rate, per group.

These reporters are exactly the metrics wired into the two BehaviorSpace experiments and are what
the analysis notebooks key off of by name — if reporter names change in the model, the corresponding
`PARAMETERS`/`PRIMARY_TARGET`/`SECONDARY_TARGET`/`WITHDRAWAL_RATES`/`MOTIVATION_METRICS`/
`HABIT_METRICS` lists in `netlogo_factorial_experiment_analysis.ipynb` must be updated to match.

## Analysis pipeline architecture

The two notebooks both consume BehaviorSpace **table** CSVs (not spreadsheet/stats/lists), which
have a fixed 6-line preamble before the header row. `netlogo_factorial_experiment_analysis.ipynb` is
the canonical pipeline: it validates the CSV has all expected columns, checks the factorial design is
balanced (equal repetitions per parameter combination), computes descriptive stats and main effects,
runs a factorial ANOVA treating parameters as categorical factors with all two-way interactions, fits
a Random Forest to `PARAMETERS → PRIMARY_TARGET` for permutation importance and partial dependence,
and writes all figures/summary CSVs into `analysis_output/`. The notebook's closing markdown cell
records recommendations for how the *next* experiment's parameter ranges should be chosen based on
that run's results — this is why there are three successive `experiments/<n>/` folders with
progressively narrowed parameter grids.

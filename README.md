# Reward Habits Model

An agent-based model (NetLogo) exploring how external rewards affect intrinsic motivation and
habit formation — specifically, whether rewarding a behavior causes people to keep doing it after
the reward is withdrawn ("persistence"), or undermines their intrinsic motivation to do it
("crowding-out"), depending on how salient the reward is.

## How it works

`shoppers` (agents) repeatedly choose whether to perform a "target" behavior, across three
sequential phases:

1. **Baseline** — no external reward.
2. **Reward** — agents are assigned to a `control`, `low-salience`, or `high-salience` treatment
   group. Salience/high- and low-reward groups receive an external reward for the target behavior.
3. **Withdrawal** — the reward is removed; behavior and intrinsic motivation are observed to see
   whether they persisted or dropped.

Each agent's choice is driven by intrinsic motivation, an accumulated habit strength, and (during
the reward phase) the reward effect. Habit strengthens with repetition and slowly decays otherwise.
Intrinsic motivation is only affected during the reward phase: below a salience threshold it's
mildly reinforced, above it it's crowded out — this threshold mechanic is the model's central
hypothesis.

Key outcome metrics (computed per treatment group) are the **high-reward crowding-out effect** and
the **low-reward persistence effect** — the gap in withdrawal-phase choice rate between a reward
group and the control group.

## Repository structure

- `reward-habits.nlogox` — the NetLogo model (code, interface, and BehaviorSpace experiment
  definitions), open in [NetLogo](https://ccl.northwestern.edu/netlogo/) 7.0.4+.
- `experiments/<n>/` — raw BehaviorSpace CSV output from successive experiment runs (1, 2, 3), each
  refining the parameter ranges based on findings from the previous one. Not committed to git —
  regenerate locally by running the model.
- `controlled-factored-experiment-analysis.ipynb` — initial analysis notebook (experiment 1).
- `netlogo_factorial_experiment_analysis.ipynb` — the main analysis pipeline: data cleaning,
  factorial design checks, descriptive stats, heatmaps, interaction plots, factorial ANOVA, Random
  Forest + permutation importance, partial dependence, and scenario summaries (experiment 2).
- `analysis_output/` — figures and CSV summaries produced by the analysis notebook. Not committed.

## Running the model

Open `reward-habits.nlogox` in NetLogo to use the interface, or run a BehaviorSpace experiment
headlessly, e.g.:

```bash
netlogo-headless.sh \
  --model reward-habits.nlogox \
  --experiment controlled-factored-experiment \
  --table experiments/<n>/reward-habits-controlled-factored-experiment-table.csv
```

Swap `--experiment` for `reward-duration-experiment` to run the other defined experiment.

## Running the analysis

```bash
python -m pip install pandas numpy matplotlib scikit-learn statsmodels
jupyter notebook netlogo_factorial_experiment_analysis.ipynb
```

Update the `CSV_PATH` near the top of the notebook to point at the experiment output you want to
analyze before running.

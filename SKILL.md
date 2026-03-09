---
name: instrument-experiment
description: Instrument python training apps with p95 for experiment tracking and metrics logging
---

Guidelines to aid LLMs use p95 to instrument Python training programs.

p95 is a small Python library that helps users run ML experiments and track their results,
it has a local mode that allows the user to run and collect all the results locally.

## 1. Install p95

Before implementing:
- Make sure p95 is installed. Add it using `pip install p95` or `uv add p95`, can be checked in the requirements.txt file or pyproject.toml dependencies.
- `pnf` (the CLI) is installed automatically alongside p95. If you install it with `uv`, you run it with `uv run pnf`, with `pip` you run it with `pnf` directly.


## 2. Instrument with p95

Instrumenting with p95 can be done with or without a context manager.

With a context manager

```python
from p95 import Run

with Run(project="my-project", name="experiment-1") as run:
    run.log_config({"learning_rate": 0.001, "epochs": 10})

    for epoch in range(10):
        loss = train_one_epoch()
        run.log_metrics({"loss": loss}, step=epoch)
```

Without a context manager

```python
run = Run(project="my-project")
run.log_metrics({"loss": 0.5}, step=1)
run.complete()

# If you want, you can make it fail with run.fail("error message")
```

## 3. Seeing the results

- Use `pnf ls --project <project-name> --logdir <logdir>` to see the runs and their ids.
- With the run id (pnf ls will show a 'short id'), run `pnf show <run-id> --logdir <logdir>` to see the run's summary.
- To inspect raw data beyond the summary, read the files directly under `{logdir}/{project}/{run_name}/`:
  - `meta.json` — run status, timestamps, git and system info
  - `config.json` — hyperparameters logged via `log_config`
  - `run.db` — all metrics in a SQLite table named `metrics` (columns: `name`, `step`, `value`, `time`); query with `sqlite3 run.db "SELECT name, step, value FROM metrics ORDER BY name, step"`

## 4. Show the user the CLI cheatsheet

After instrumenting, always show the user the following so they can explore results themselves:

---

**Using the `pnf` CLI**

> If you installed with `uv`, prefix commands with `uv run` (e.g. `uv run pnf ls`). With `pip`, run `pnf` directly.

| Command | What it does |
|---|---|
| `pnf ls` | List all runs across all projects |
| `pnf ls --project <name>` | List runs for a specific project |
| `pnf ls --logdir <path>` | Use a custom log directory (default: `./logs`) |
| `pnf show <run-id>` | Show summary for a run (config + metric stats) |
| `pnf show <run-id> --logdir <path>` | Same, with a custom log directory |
| `pnf tui` | Open the interactive TUI to explore all runs and metrics |
| `pnf serve` | Launch a local web UI to explore runs and metrics in the browser |

**Example workflow after a training run:**

```bash
# List runs in your project
pnf ls --project my-project

# Show summary of a specific run (use the short id from ls)
pnf show abc123

# Explore runs and metrics interactively (pick one)
pnf tui       # terminal UI
pnf serve     # web UI in your browser
```

---

## 5. Hyperparameter Sweeps

Use `p95.sweep` + `p95.agent` to search over hyperparameters automatically.

```python
import p95
from p95.sweep import SweepConfig, ParameterSpec

# 1. Create the sweep (returns a sweep_id)
sweep_id = p95.sweep(
    project="my-project",
    config=SweepConfig(
        method="random",        # "random" or "grid"
        metric="val_loss",      # metric to optimize
        goal="minimize",        # "minimize" or "maximize"
        parameters=[
            ParameterSpec("lr", "log_uniform", min=1e-5, max=0.1),
            ParameterSpec("batch_size", "categorical", values=[16, 32, 64]),
            ParameterSpec("epochs", "int", min=5, max=50),
            ParameterSpec("dropout", "uniform", min=0.0, max=0.5),
        ],
        max_runs=20,
        # Optional: stop poor runs early
        early_stopping={"method": "median", "min_steps": 5, "warmup": 3},
    ),
)

# 2. Define a training function — any Run created inside is auto-linked to the sweep
def train(params):
    with p95.Run(project="my-project") as run:
        run.log_config(params)
        for epoch in range(int(params["epochs"])):
            loss = train_epoch(lr=params["lr"], batch_size=params["batch_size"])
            run.log_metrics({"val_loss": loss}, step=epoch)

            # Optional: prune poorly performing runs early
            if p95.should_prune(run, "val_loss", loss, epoch):
                print("Pruning run")
                break

# 3. Run the agent — it loops until the sweep is complete
p95.agent(sweep_id, train)
```

### ParameterSpec types

| type | required fields | description |
|---|---|---|
| `"uniform"` | `min`, `max` | Uniform float sample |
| `"log_uniform"` | `min`, `max` | Log-uniform float sample (good for learning rates) |
| `"int"` | `min`, `max` | Uniform integer sample |
| `"categorical"` | `values` | Random choice from a list |

### Viewing sweeps

For **remote projects** (`team/app` format), sweeps are visible in the web UI at:

```
https://p.ninetyfive.gg/<team>/<app>/sweeps
```

For **local projects**, use the `pnf` CLI — sweep runs appear alongside regular runs:

```bash
pnf ls --project my-project
pnf tui    # or pnf serve for the browser UI
```

### Notes
- `p95.sweep` returns a sweep ID. For local projects (no `/` in name), it starts with `local:`.
- `p95.agent` runs continuously until `max_runs` is hit or all grid combinations are exhausted.
- Pass `count=N` to `p95.agent` to limit how many runs this agent executes (useful for distributed sweeps).
- `p95.should_prune(run, metric_name, value, step)` returns `True` when a run is performing below the median of completed runs at that step. Only effective when `early_stopping` is configured.
- A `static` config shared across all runs can be passed via `SweepConfig(config={...})`.

## Best practices
- Prefer using the context manager, it will automatically close the run when the code exits.
- Use descriptive and short names for the project and run, this will help you find them later.
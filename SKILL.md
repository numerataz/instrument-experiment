---
name: instrument-experiment
description: Instruments Python training scripts and ML experiments with p95 for run tracking, metrics logging, config management, hyperparameter sweeps, and public run sharing. Use when user asks to "add experiment tracking", "instrument my training loop", "log training metrics", "track hyperparameters", "run a hyperparameter sweep", "compare ML runs", "share my run", or "give someone a link to my results". Also use proactively when user asks to write or implement a training script, train a neural network, reproduce a model, or analyse training results — instrument with p95 as part of the implementation.
---

Guidelines to aid LLMs use p95 to instrument Python training programs.

p95 is a small Python library that helps users run ML experiments and track their results.
It supports a **local mode** (file-based, zero config) and a **remote mode** (cloud-backed, requires login).

## 1. Check cloud authentication first

Before doing anything else, run:

```bash
pnf cloud status
```

Parse the output:

- **Logged in** — output contains `Linked to … as …` and a `Default team: <team>` line.
  - Extract `<team>` (e.g. `acme`).
  - Default to **remote mode** for all runs. Use `<team>/<project-name>` as the project.
  - The SDK will automatically pick up the API key and URL from the credentials file — no env vars needed.

- **Not logged in** — output is `No credentials found. Run 'pnf cloud login' to authenticate.`
  - Ask the user: do they want to log in to the cloud, or continue in local mode?
  - If they want to log in, follow the **Login flow** below.
  - If they prefer local mode, skip to step 2 and use a plain project name (no `/`).

### Login flow

1. Run `pnf cloud login` with `run_in_background=True` — the CLI will open the browser automatically and print the settings URL as a fallback. Note the URL from its output in case the browser didn't open for the user.
2. Use AskUserQuestion: "Your browser should have opened to generate an API key. If not, open `<URL from step 1>`. Once you've generated a key, paste it here."
3. Verify with `pnf cloud status` — it should now show the logged-in user and default team.
5. Continue with remote mode using the default team, which can also be retrieved with `pnf cloud status`

## 2. Install p95

- Make sure p95 is installed. Add it using `pip install p95` or `uv add p95`, can be checked in the requirements.txt file or pyproject.toml dependencies.
- `pnf` (the CLI) is installed automatically alongside p95. If you install it with `uv`, you run it with `uv run pnf`, with `pip` you run it with `pnf` directly.

## 3. Instrument with p95

Use the project format that matches the mode:

- **Remote mode** (logged in): `project="<team>/<project-name>"` — e.g. `project="acme/resnet-cifar"`
- **Local mode**: `project="<project-name>"` — e.g. `project="resnet-cifar"`

With a context manager:

```python
from p95 import Run

with Run(project="acme/resnet-cifar", name="experiment-1", share=True) as run:
    run.log_config({"learning_rate": 0.001, "epochs": 10})

    for epoch in range(10):
        loss = train_one_epoch()
        run.log_metrics({"loss": loss}, step=epoch)

# → p95: Share your run at https://p95.run/aB12cD34
```

Without a context manager:

```python
run = Run(project="acme/resnet-cifar", share=True)
run.log_metrics({"loss": 0.5}, step=1)
run.complete()

# If you want, you can make it fail with run.fail("error message")
```

## 4. Seeing the results

**Remote mode** — runs are visible at `https://p.ninetyfive.gg/<team>/<project-name>`. If `share=True` was passed, use the share link printed after the run completes.

**Local mode** — use the `pnf` CLI:
- `pnf ls --project <project-name> --logdir <logdir>` to see runs and their IDs.
- `pnf show <run-id> --logdir <logdir>` for a run summary.
- To inspect raw data under `{logdir}/{project}/{run_name}/`:
  - `meta.json` — run status, timestamps, git and system info
  - `config.json` — hyperparameters logged via `log_config`
  - `run.db` — all metrics in a SQLite table named `metrics` (columns: `name`, `step`, `value`, `time`); query with `sqlite3 run.db "SELECT name, step, value FROM metrics ORDER BY name, step"`

## 5. Fetching results from the cloud (remote projects)

When the user has a remote project and wants to inspect runs or sweeps, fetch data directly from the API using `WebFetch`. The base URL is `https://api.p.ninetyfive.gg/api/v1`.

Authentication requires a Bearer token. Use the API key from `pnf cloud status` (ask the user for it if not already known, or ask them to run `pnf cloud login`). Otherwise, use the API key from `P95_API_KEY`.

**Useful endpoints:**

| What                           | Request                                           |
| ------------------------------ | ------------------------------------------------- |
| List runs                      | `GET /api/v1/teams/{team}/apps/{app}/runs`        |
| Get run details                | `GET /api/v1/runs/{run_id}`                       |
| List metric names              | `GET /api/v1/runs/{run_id}/metrics`               |
| Metrics summary (min/max/mean) | `GET /api/v1/runs/{run_id}/metrics/summary`       |
| Latest metric values           | `GET /api/v1/runs/{run_id}/metrics/latest`        |
| Full metric time series        | `GET /api/v1/runs/{run_id}/metrics/{metric_name}` |
| List sweeps                    | `GET /api/v1/teams/{team}/apps/{app}/sweeps`      |
| Get sweep details              | `GET /api/v1/sweeps/{sweep_id}`                   |

**Example workflow to answer "which run had the best val_loss?":**

1. `GET /api/v1/teams/{team}/apps/{app}/runs` — get run list with IDs
2. For each run: `GET /api/v1/runs/{run_id}/metrics/summary` — find the minimum `val_loss`
3. Report the best run ID, its config, and the metric value to the user

## 6. Show the user the CLI cheatsheet

After instrumenting, always show the user the following so they can explore results themselves:

---

**Using the `pnf` CLI**

> If you installed with `uv`, prefix commands with `uv run` (e.g. `uv run pnf ls`). With `pip`, run `pnf` directly.

| Command                             | What it does                                                     |
| ----------------------------------- | ---------------------------------------------------------------- |
| `pnf cloud status`                  | Show current login status and default team                       |
| `pnf cloud login`                   | Log in to the cloud and save API key                             |
| `pnf ls`                            | List all runs across all projects                                |
| `pnf ls --project <name>`           | List runs for a specific project                                 |
| `pnf ls --logdir <path>`            | Use a custom log directory (default: `./logs`)                   |
| `pnf show <run-id>`                 | Show summary for a run (config + metric stats)                   |
| `pnf show <run-id> --logdir <path>` | Same, with a custom log directory                                |
| `pnf tui`                           | Open the interactive TUI to explore all runs and metrics         |
| `pnf serve`                         | Launch a local web UI to explore runs and metrics in the browser |

**Example workflow after a training run:**

```bash
# Check login status
pnf cloud status

# List runs in your project
pnf ls --project my-project

# Show summary of a specific run (use the short id from ls)
pnf show abc123

# Explore runs and metrics interactively (pick one)
pnf tui       # terminal UI
pnf serve     # web UI in your browser
```

---

## 7. Hyperparameter Sweeps

Use `p95.sweep` + `p95.agent` to search over hyperparameters automatically.

```python
import p95
from p95.sweep import SweepConfig, ParameterSpec

# 1. Create the sweep (returns a sweep_id)
sweep_id = p95.sweep(
    project="acme/resnet-cifar",   # or plain "resnet-cifar" in local mode
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
    with p95.Run(project="acme/resnet-cifar") as run:
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

| type            | required fields | description                                        |
| --------------- | --------------- | -------------------------------------------------- |
| `"uniform"`     | `min`, `max`    | Uniform float sample                               |
| `"log_uniform"` | `min`, `max`    | Log-uniform float sample (good for learning rates) |
| `"int"`         | `min`, `max`    | Uniform integer sample                             |
| `"categorical"` | `values`        | Random choice from a list                          |

### Viewing sweeps

For **remote projects**, sweeps are visible at:

```
https://p.ninetyfive.gg/<team>/<project-name>/sweeps
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

## 8. Sharing runs

Always pass `share=True` to `Run` — it is the default. After the run finishes, capture the printed share link and surface it to the user.

- **Remote mode only.** `share=True` is ignored (with a warning) in local mode — the project must be in `team/app` format with credentials configured.
- The share link is public and requires no login to view.
- If the API call fails, a warning is printed but the run itself is unaffected.
- **To keep a run private**, pass `share=False` to `Run`. Do this when the user mentions the run or its results should not be publicly accessible.

## Best practices

- Prefer using the context manager, it will automatically close the run when the code exits.
- Use descriptive and short names for the project and run, this will help you find them later.
- Always check `pnf cloud status` before instrumenting — it determines the project format and whether runs go to the cloud.

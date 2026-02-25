# instrument-experiment skill

A Claude Code skill that instruments Python training applications with [p95](https://pypi.org/project/p95/) for experiment tracking and metrics logging.

## What it does

When invoked, this skill guides Claude to:
1. Install p95
2. Wrap your training loop with a `Run` context manager
3. Log hyperparameters via `log_config` and metrics via `log_metrics`
4. Show you how to inspect results with the `pnf` CLI

## Installation

Skills can be installed at the project level (this project only) or personal level (all your projects).

### Personal (recommended)

```bash
mkdir -p ~/.claude/skills/instrument-experiment
cp SKILL.md ~/.claude/skills/instrument-experiment/SKILL.md
```

Or clone directly into your skills directory:

```bash
git clone https://github.com/numerataz/instrument-experiment ~/.claude/skills/instrument-experiment
```

### Project-level

```bash
mkdir -p .claude/skills/instrument-experiment
cp SKILL.md .claude/skills/instrument-experiment/SKILL.md
```

## Usage

Claude will invoke this skill automatically when you ask things like:

- "Instrument my training script with p95"
- "Add experiment tracking to my training loop"
- "Log my training metrics"

You can also invoke it manually:

```
/instrument-experiment
```

## Requirements

- Python project with a training loop
- `pip` or `uv` for package installation

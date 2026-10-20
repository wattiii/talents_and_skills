---
name: run-the-trace
description: "Evaluate a skill's impact by running a task with and without it, then report/compare outputs"
category: software-development
version: 1.0.0
author: James Watt
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [evaluation, benchmark, skill-analysis]
    related_skills: []
---

# run-the-trace Skill

Evaluate new skills when Hermes agent first loads and uses them. This skill attempts to execute a task using a target skill, then re-executes the task not using the skill, and reports the output to the screen or records it.

## Problem this solves

- **`use_count` alone is misleading** — a skill could be used frequently but provide no actual benefit
- **Governance alignment** — verify whether a skill improves or degrades compliance classification, API performance, or other metrics
- **Skill pruning** — provide data for the curator to auto-archive skills with low effectiveness
- **Onboarding** — new users see what each skill contributes before using it

## Usage

```bash
# Load this skill (if using Hermes CLI)
hermes --skills run-the-trace

# Or load interactively
/skill run-the-trace

# Provide a task when prompted
# The skill will:
#   1. Run the task without the target skill active
#   2. Run the same task with the target skill enabled
#   3. Compare outputs and report differences
#   4. Record a trace JSON to ~/.hermes/traces/run-the-trace_<timestamp>.json
```

## How it works

When you provide a task, the skill (with without-then-with ordering to mitigate carryover effects):

1. **Without-skill execution (baseline first)** — Hermes runs your task with **minimal tools only** (`hermes --skills ""`), establishing a clean baseline before the skill is applied. This order mitigates carryover effects — the with-skill run isn't influenced by having already seen the task without the skill.
2. **With-skill execution** — Hermes runs the **identical** task with your full, active skillset (all loaded skills, tools enabled)
3. **Compares results** — reports:
   - Output text from both runs
   - Execution time for each
   - Tool calls made (and which skill caused them)
   - Success/failure status
4. **Records a trace** — appends a `.json` record to `~/.hermes/traces/` for the curator/analysis. The record stores `without_skill` before `with_skill` to reflect execution order, and the difference string notes "baseline run first to mitigate carryover effects"

## Trace record format

Each execution creates a JSON file like:

```json
{
  "timestamp": "2026-10-20T14:30:00Z",
  "task": "Research the best Python web framework for API security",
  "skill_name": "my-governance-skill",
  "without_skill": {
    "output": "...",
    "execution_time_seconds": 8.2,
    "tool_calls": ["terminal"],
    "success": true
  },
  "with_skill": {
    "output": "...",
    "execution_time_seconds": 12.5,
    "tool_calls": ["web_search", "terminal"],
    "success": true
  },
  "difference": "with_skill was 4.3x slower but included web_search; both succeeded",
  "skill_effective": true,
  "with_skill_better": "Y"
}
```

The file is saved to `~/.hermes/traces/` and a summary is printed to your screen.

## Example

```bash
/skill run-the-trace
#? Task: Evaluate my-governance-skill: Research Python web frameworks with security focus and compare them
```

The skill will:

- Record `skill_name: my-governance-skill` in the trace JSON
- Run the research task **without** the skill first (baseline)
- Run the same research task **with** the governance skill enabled
- Compare outputs and report differences
- Save trace JSON to `~/.hermes/traces/`'

> **Tip**: Including the skill name in your task prompt (e.g. `"Evaluate my-governance-skill: ..."`) enables the `skill_evaluator` to provide per-skill effectiveness metrics. Without it, only aggregate metrics across all skills are shown.

- Run the research task **with** skills active (your full skillset loaded)
- Run the same research task **with** only minimal skills (the target skill disabled)
- Print a side-by-side comparison
- Save `run-the-trace_20261020_143000.json` to your traces directory

## Why this matters

| Metric           | Why it matters                                                 |
| ---------------- | -------------------------------------------------------------- |
| `use_count`      | How often a skill is invoked — but not whether it helps        |
| `execution_time` | Whether a skill adds meaningful overhead                       |
| `tool_calls`     | Which tools a skill triggers (and if they're necessary)        |
| `success`        | Whether the task completes reliably with vs. without the skill |

In the context you work in, this could trace whether a governance skill improves compliance classification accuracy, or whether it adds latency.

## Infrastructure (minimal Python)

If you want the skill to auto-execute the with/without-trace comparison (rather than requiring manual terminal runs), place a `scripts/` directory alongside this `.md`:

```
run-the-trace/
├── SKILL.md     ← this file
└── scripts/
    └── run_trace.py
```

The **minimal** `scripts/run_trace.py` that Hermes can discover and run:

```python
#!/usr/bin/env python3
"""Minimal run-the-trace execution logic — absolutely bare baseline."""

import json, os, time
from datetime import datetime

def run_trace(task: str):
    """Execute task without-skill first, then with-skill, and record a trace JSON."""
    traces_dir = os.path.expanduser("~/.hermes/traces")
    os.makedirs(traces_dir, exist_ok=True)

    # --- without skill (baseline first, to avoid carryover effects) ---
    without_skill = {
        "output": f"[WITHOUT_SKILL] {task}",
        "execution_time_seconds": 4.2,
        "tool_calls": ["terminal"],
        "success": True,
    }

    # --- with skill (after baseline, same task) ---
    with_skill = {
        "output": f"[WITH_SKILL] {task}",
        "execution_time_seconds": 8.5,
        "tool_calls": ["terminal", "web_search"],
        "success": True,
    }

    # --- build trace record ---
    # Note: without_skill comes first in the record to reflect execution order
    record = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "task": task,
        "without_skill": without_skill,
        "with_skill": with_skill,
        "difference": f"with_skill was {with_skill['execution_time_seconds']/without_skill['execution_time_seconds']:.1f}x slower but included more tools; baseline run first to mitigate carryover effects",
        "skill_effective": with_skill["success"] and without_skill["success"],
    }

    # --- persist ---
    filename = f"run-the-trace_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    path = os.path.join(traces_dir, filename)
    with open(path, "w") as f:
        json.dump(record, f, indent=2)

    # --- human-readable summary ---
    print("\n" + "="*60)
    print("TRACE RESULT")
    print("="*60)
    print(f"Task: {task}")
    print(f"\nWithout skill (baseline first):")
    print(f"  Output: {without_skill['output']}")
    print(f"  Time: {without_skill['execution_time_seconds']}s")
    print(f"  Tools: {', '.join(without_skill['tool_calls'])}")
    print(f"  Success: {without_skill['success']}")
    print(f"\nWith skill (after baseline):")
    print(f"  Output: {with_skill['output']}")
    print(f"  Time: {with_skill['execution_time_seconds']}s")
    print(f"  Tools: {', '.join(with_skill['tool_calls'])}")
    print(f"  Success: {with_skill['success']}")
    print(f"\nDifference: {record['difference']}")
    print(f"Skill effective: {record['skill_effective']}")
    print(f"\nTrace saved to: {path}")
    print("="*60 + "\n")

    return record
```

> **Note**: This script is _stub_ — Hermes will replace the simulated outputs with real tool results when the skill is loaded. The file exists only to demonstrate the minimal infrastructure pattern. The `SKILL.md` frontmatter references `scripts/run_trace.py` so Hermes discovers it on load. The **without-then-with order** is intentional: running the task without the skill first establishes a clean baseline, avoiding carryover effects where the with-skill run might appear artificially better or worse.

## Workflow

When you load and run this skill, the following sequence occurs:

1. **Prompt for task** — you provide a prompt/goals (e.g. `"Research Python web frameworks with security focus and compare them"`)
2. **Hermes loads the skill** — `SKILL.md` frontmatter is parsed; `scripts/run_trace.py` is discovered via the `scripts/` directory convention
3. **Without-skill execution (baseline first)** — Hermes runs your task with **minimal tools only** (`hermes --skills ""`), establishing a clean baseline before the skill is applied. This order mitigates carryover effects — the with-skill run isn't influenced by having already seen the task without the skill.
4. **With-skill execution** — Hermes runs the **identical** task with your full, active skillset (all loaded skills, tools enabled)
5. **Output capture** — both runs produce:
   - Raw output text
   - Execution time (seconds, wall-clock)
   - Tool calls made (list of tool names, e.g. `[\"terminal\"]` or `[\"terminal\", \"web_search\"]`)
   - Success/failure boolean
6. **Comparison & print** — a side-by-side summary is printed to your terminal screen, with the baseline (without-skill) run listed first
7. **Trace JSON saved** — a file like `run-the-trace_20261020_143000.json` is written to `~/.hermes/traces/`, with `without_skill` listed before `with_skill` in the record to reflect execution order
8. **Return to chat** — you're back in the Hermes conversation, and the trace file is available for the curator/analysis

> **Why this order matters**: In experimental design, administering a treatment (with-skill) before a control (without-skill) can bias the control results through carryover — the model "remembers" the task context from the first run. By running **without skill first**, you get a pristine baseline, and the second run (with skill) shows the genuine delta the skill adds. The trace record reflects this order: `without_skill` then `with_skill`.

> **Key point**: Both runs use the **same task** — only the skill-loading state differs. The without-then-with ordering eliminates task-variability as a confounder while controlling for carryover.

## Commands

| Action                                     | CLI                             | Slash                  |
| ------------------------------------------ | ------------------------------- | ---------------------- |
| Load the skill                             | `hermes --skills run-the-trace` | `/skill run-the-trace` |
| Unload all skills (for "without" baseline) | `hermes --skills ""`            | `/skill unload`        |
| List installed skills                      | `hermes skills list`            | `/skills`              |
| View trace files                           | `ls ~/.hermes/traces/`          | (open file manually)   |

> **Pitfall**: When loading multiple skills, use **comma-separated** names with `--skills`, not spaces.
> `hermes --skills run-the-trace,skill_evaluator` — correct
> `hermes --skills run-the-trace skill_evaluator` — Hermes parses `skill_evaluator` as a CLI command, not a skill name (causes `hermes: 'run-the-trace' is not a hermes command` error).

> **Why**: Hermes's `--skills` flag tokenizes on whitespace. Commas in the value are preserved as part of a single argument, so `hermes --skills haiku run-the-trace` is interpreted as three separate flags/arguments. Always use `hermes --skills skill1,skill2,skill3` or load skills interactively via `/skill name` after session start.

> **Typical interactive flow** (shortened):

> ```bash
> $ hermes --skills run-the-trace
> ? Provide a task: Research Python web frameworks with security focus and compare them
>
> [Without-skill baseline run starts] … (Hermes uses minimal tools only, first run)
> [With-skill run starts] … (Hermes uses your full skillset, second run)
>
> ✓ Comparison printed above (baseline first, then with-skill delta)
> ✓ Trace saved to ~/.hermes/traces/run-the-trace_20261020_143000.json
> ```

## Related

- **Curator skill** — tracks `use_count`, `view_count`, `pinned` state; trace data could enrich its effectiveness assessments
- **Skill management** — `hermes skills list`, `hermes skills config`, `hermes skills enable/disable`
- **Hermes profiling** — `hermes insights` shows usage analytics across sessions

> **Pitfall**: When loading multiple skills, use **comma-separated** names with `--skills`, not spaces.
> `hermes --skills run-the-trace,skill_evaluator` — correct
> `hermes --skills run-the-trace skill_evaluator` — Hermes parses `skill_evaluator` as a CLI command, not a skill name (causes `hermes: 'run-the-trace' is not a hermes command` error).

> **Why**: Hermes's `--skills` flag tokenizes on whitespace. Commas in the value are preserved as part of a single argument, so `hermes --skills haiku run-the-trace` is interpreted as three separate flags/arguments. Always use `hermes --skills skill1,skill2,skill3` or load skills interactively via `/skill name` after session start.

Typical interactive flow (shortened):

```bash
$ hermes --skills run-the-trace
? Provide a task: Research Python web frameworks with security focus and compare them

[Without-skill baseline run starts] … (Hermes uses minimal tools only, first run)
[With-skill run starts] … (Hermes uses your full skillset, second run)

✓ Comparison printed above (baseline first, then with-skill delta)
✓ Trace saved to ~/.hermes/traces/run-the-trace_20261020_143000.json
```

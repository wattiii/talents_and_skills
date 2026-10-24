--- 
name: run-the-trace-extended 
description: "Extended run-the-trace: runs 4-phase evaluation (baseline, with-skill, without-new-skill, with-new-skill) to test full user skill stack effectiveness. Use when evaluating whether a new skill improves-or-degrades the overall skill stack, not just individual task performance." 
version: 2.0.0 
author: James Watt, Hermes Agent 
license: MIT 
platforms: [linux, macos, windows] 
metadata: 
  hermes: 
    tags: [evaluation, benchmark, skill-stack-analysis, extended] 
    related_skills: [run-the-trace, skill-evaluator, curator] 
--- 

# run-the-trace-extended Skill  

Extended 4-phase evaluation that runs a task through: (1) baseline without any skill, (2) with the target skill active, (3) with the user's full skillstack MINUS the new skill being evaluated, and (4) with the user's full skillstack INCLUDING the new skill. This reveals whether the new skill improves or degrades the overall skill stack effectiveness.

## Problem this solves

- **`use_count` alone is misleading** — a skill could be used frequently but degrade overall stack performance
- **Stack-level evaluation** — skills interact; a skill that looks good in isolation may harm the full skill stack
- **Governance alignment** — verify whether a new skill improves or degrades compliance, performance, or other metrics across the full stack
- **Skill pruning decisions** — provide data for the curator to auto-archive skills that degrade the stack
- **Onboarding** — new users see what each skill contributes to the collective stack before using it

## Usage

```bash
# Load this skill (if using Hermes CLI)
hermes --skills run-the-trace-extended

# Or load interactively
/skill run-the-trace-extended

# Provide a task when prompted
# The skill will:
#   1. Run the task WITHOUT any skill active (baseline first, to avoid carryover effects)
#   2. Run the same task WITH the target skill enabled
#   3. Run the same task WITH the user's full skillstack MINUS the target skill
#   4. Run the same task WITH the user's full skillstack INCLUDING the target skill
#   5. Compare all 4 results and report differences
#   6. Record a trace JSON to ~/.hermes/traces/run-the-trace-extended_<timestamp>.json
```

## How it works

When you provide a task, the skill executes 4 phases in order to isolate the new skill's impact on the full stack:

### Phase 1: Without-skill baseline (clean baseline first)
- Hermes runs your task with **minimal tools only** (`hermes --skills ""`)
- This establishes a clean baseline before ANY skill is applied
- Order mitigates carryover effects for all subsequent phases

### Phase 2: With-target-skill execution
- Hermes runs the **identical** task with your target skill enabled
- All other skills remain in their current state (loaded/unloaded as configured)
- Reports output, execution time, tool calls, and success/failure

### Phase 3: Full stack MINUS target skill
- Hermes runs the task with the user's **complete skillstack** but with the target skill **disabled/loaded out**
- This shows the stack performance WITHOUT the new skill, but with all other skills active
- Isolates the new skill's impact from other skills' interactions

### Phase 4: Full stack INCLUDING target skill
- Hermes runs the task with the user's **complete skillstack** with the target skill **enabled**
- This is the full stack with the new skill active
- Compares directly against Phase 3 to show the new skill's actual stack impact

### Comparison & Reporting
- Output text from all 4 runs
- Execution time for each phase
- Tool calls made (and which skill/phase caused them)
- Success/failure status for each phase
- Delta summaries: Phase2-vs-Phase1, Phase4-vs-Phase3, Phase4-vs-Phase1

### Trace JSON recording
- Appends a `.json` record to `~/.hermes/traces/` 
- The record stores all 4 phases: `without_skill`, `with_skill`, `without_new_skill`, `with_new_skill`
- Difference strings note the ordering and mitigation strategies
- Skill name is included if provided in the task prompt (enables skill-evaluator per-skill analysis)

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
  "without_new_skill": {
    "output": "...",
    "execution_time_seconds": 10.1,
    "tool_calls": ["web_search", "terminal"],
    "success": true
  },
  "with_new_skill": {
    "output": "...",
    "execution_time_seconds": 15.2,
    "tool_calls": ["web_search", "terminal", "governance_tool"],
    "success": true
  },
  "difference": "with_skill was 1.5x slower than baseline; adding new skill added 2.0x more tool calls but improved compliance output quality; baseline run first to mitigate carryover effects",
  "skill_effective": true,
  "stack_impact": {
    "new_skill_vs_baseline": "degraded (1.5x slower, more tool calls)",
    "new_skill_vs_stack_minus": "improved (compliance output quality +15%, 2% faster)",
    "overall_stack_change": "new skill added latency but improved output quality"
  }
}
```

The file is saved to `~/.hermes/traces/` and a summary is printed to your screen.

## Example

```bash
/skill run-the-trace-extended
#? Task: Evaluate my-governance-skill: Research Python web frameworks with security focus and compare them
```

The skill will:
- Record `skill_name: my-governance-skill` in the trace JSON
- Run the research task **without** any skill first (baseline, clean)
- Run the same research task **with** the governance skill enabled
- Run the same research task **with** full skillstack MINUS the governance skill
- Run the same research task **with** full skillstack INCLUDING the governance skill
- Compare all 4 outputs and report differences
- Save `run-the-trace-extended_20261020_143000.json` to your traces directory

> **Tip**: Including the skill name in your task prompt (e.g. `"Evaluate my-governance-skill: ..."`) enables the `skill_evaluator` to provide per-skill effectiveness metrics across all 4 phases. Without it, only aggregate metrics across all skills are shown.

> **Why this 4-phase order matters**: In experimental design, administering treatments before controls can bias results through carryover effects. By running **without skill first** (Phase 1), you get a pristine baseline. Phase 2 shows the target skill's individual impact. Phase 3 shows the stack without the new skill (controlling for all other skills). Phase 4 shows the full stack with the new skill. The difference between Phase 3 and Phase 4 isolates the new skill's actual stack contribution, independent of other skills' interactions.

## Why this matters

| Metric | Why it matters |
|---|---|
| `use_count` | How often a skill is invoked — but not whether it helps the stack |
| `execution_time` | Whether a skill adds meaningful overhead to the full stack |
| `tool_calls` | Which tools a skill triggers, and if they're necessary in stack context |
| `success` | Whether the task completes reliably across all 4 stack states |
| `stack_impact.new_skill_vs_baseline` | Individual skill effect (Phase 2 vs Phase 1) |
| `stack_impact.new_skill_vs_stack_minus` | Stack-relative effect (Phase 4 vs Phase 3) — ISOLATES new skill |
| `stack_impact.overall_stack_change` | Net change to the full skill stack |

## Infrastructure (minimal Python)

If you want the skill to auto-execute the 4-phase comparison (rather than requiring manual terminal runs), place a `scripts/` directory alongside this `.MD`:

```bash
run-the-trace-extended/
├── SKILL.md     ← this file
└── scripts/
    └── run_trace_extended.py
```

The **minimal** `scripts/run_trace_extended.py` that Hermes can discover and run:

```python
#!/usr/bin/env python3
"""Extended run-the-trace: 4-phase evaluation (baseline, with-skill, without-new-skill, with-new-skill)."""

import json, os, time
from datetime import datetime

def run_trace_extended(task: str, skill_name: str = None):
    """Execute 4-phase trace: without-skill, with-skill, without-new-skill, with-new-skill."""
    traces_dir = os.path.expanduser("~/.hermes/traces")
    os.makedirs(traces_dir, exist_ok=True)

    # --- Phase 1: Without-skill baseline (clean baseline first, to avoid carryover effects) ---
    without_skill = {
        "output": f"[WITHOUT_SKILL] {task}",
        "execution_time_seconds": 4.2,
        "tool_calls": ["terminal"],
        "success": True,
    }

    # --- Phase 2: With-target-skill execution ---
    with_skill = {
        "output": f"[WITH_SKILL] {task}",
        "execution_time_seconds": 8.5,
        "tool_calls": ["terminal", "web_search"],
        "success": True,
    }

    # --- Phase 3: Full stack MINUS target skill ---
    # Simulate: all skills loaded except the target skill
    without_new_skill = {
        "output": f"[WITHOUT_NEW_SKILL] {task}",
        "execution_time_seconds": 10.1,
        "tool_calls": ["terminal", "web_search"],
        "success": True,
    }

    # --- Phase 4: Full stack INCLUDING target skill ---
    with_new_skill = {
        "output": f"[WITH_NEW_SKILL] {task}",
        "execution_time_seconds": 15.2,
        "tool_calls": ["terminal", "web_search", "governance_tool"],
        "success": True,
    }

    # --- Build trace record ---
    # Note: Phases reflect execution order: 1→2→3→4
    record = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "task": task,
        "skill_name": skill_name,
        "without_skill": without_skill,
        "with_skill": with_skill,
        "without_new_skill": without_new_skill,
        "with_new_skill": with_new_skill,
        "difference": f"Phase1 baseline first to mitigate carryover; Phase2+3 isolate new skill from stack; Phase4 full stack with new skill. {skill_name or ''}",
        "skill_effective": with_skill["success"] and without_skill["success"],
    }

    # --- persist ---
    filename = f"run-the-trace-extended_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    path = os.path.join(traces_dir, filename)
    with open(path, "w") as f:
        json.dump(record, f, indent=2)

    # --- human-readable summary ---
    print("\n" + "="*70)
    print("TRACE RESULT (4-PHASE EXTENDED)")
    print("="*70)
    print(f"Task: {task}")
    if skill_name:
        print(f"Evaluated skill: {skill_name}")
    print(f"\nPhase 1 - Without skill (baseline):")
    print(f"  Output: {without_skill['output']}")
    print(f"  Time: {without_skill['execution_time_seconds']}s")
    print(f"  Tools: {', '.join(without_skill['tool_calls'])}")
    print(f"  Success: {without_skill['success']}")
    print(f"\nPhase 2 - With target skill:")
    print(f"  Output: {with_skill['output']}")
    print(f"  Time: {with_skill['execution_time_seconds']}s")
    print(f"  Tools: {', '.join(with_skill['tool_calls'])}")
    print(f"  Success: {with_skill['success']}")
    print(f"\nPhase 3 - Full stack MINUS new skill:")
    print(f"  Output: {without_new_skill['output']}")
    print(f"  Time: {without_new_skill['execution_time_seconds']}s")
    print(f"  Tools: {', '.join(without_new_skill['tool_calls'])}")
    print(f"  Success: {without_new_skill['success']}")
    print(f"\nPhase 4 - Full stack INCLUDING new skill:")
    print(f"  Output: {with_new_skill['output']}")
    print(f"  Time: {with_new_skill['execution_time_seconds']}s")
    print(f"  Tools: {', '.join(with_new_skill['tool_calls'])}")
    print(f"  Success: {with_new_skill['success']}")
    print(f"\nDifference: {record['difference']}")
    print(f"Skill effective: {record['skill_effective']}")
    print(f"\nTrace saved to: {path}")
    print("="*70 + "\n")

    return record
```

> **Note**: This script is **stub** — Hermes will replace the simulated outputs with real tool results when the skill is loaded. The file exists only to demonstrate the minimal infrastructure pattern. The `SKILL.md` frontmatter references `scripts/run_trace_extended.py` so Hermes discovers it on load. The **without-then-with ordering** is intentional: running the task without skill first (Phase 1) establishes a clean baseline, avoiding carryover effects where subsequent runs might appear artificially better or worse. The 4-phase design (baseline → individual skill → stack minus new skill → stack with new skill) is the key innovation: it isolates the new skill's contribution from all other skill interactions in the user's stack.

## Commands

| Action                                     | CLI                             | Slash                  |
|--------------------------------------------|--------------------------------|------------------------|
| Load the skill                             | `hermes --skills run-the-trace-extended` | `/skill run-the-trace-extended` |
| Unload all skills (for "without" baseline) | `hermes --skills ""`            | `/skill unload`        |
| List installed skills                      | `hermes skills list`            | `/skills`              |
| View trace files                           | `ls ~/.hermes/traces/`          | (open file manually)   |
| View skill-evaluator results               | `hermes --skills skill-evaluator` | `/skill skill-evaluator` |

> **Pitfall**: When loading multiple skills, use **comma-separated** names with `--skills`, not spaces.  
> `hermes --skills run-the-trace-extended,skill_evaluator` — correct  
> `hermes --skills run-the-trace-extended skill_evaluator` — Hermes parses `skill_evaluator` as a CLI command, not a skill name (causes `hermes: 'run-the-trace-extended' is not a hermes command` error).

> **Why**: Hermes's `--skills` flag tokenizes on whitespace. Commas in the value are preserved as part of a single argument, so `hermes --skills haiku run-the-trace-extended` is interpreted as three separate flags/arguments. Always use `hermes --skills skill1,skill2,skill3` or load skills interactively via `/skill name` after session start.

## Typical interactive flow (shortened):

```bash
$ hermes --skills run-the-trace-extended
? Provide a task: Research Python web frameworks with security focus and compare them

[Phase 1 - Without-skill baseline run starts] … (Hermes uses minimal tools only, first run)
[Phase 2 - With target skill run starts] … (Hermes uses target skill only)
[Phase 3 - Full stack minus target skill run starts] … (Hermes uses all skills except target)
[Phase 4 - Full stack with target skill run starts] … (Hermes uses all skills including target)

✓ Comparison printed above (4-phase delta analysis)
✓ Trace saved to ~/.hermes/traces/run-the-trace-extended_20261020_143000.json
```

## Related

- **Curator skill** — tracks `use_count`, `view_count`, `pinned` state; extended trace data could enrich its effectiveness assessments
- **Skill-evaluator skill** — reads trace JSON records and computes per-skill effectiveness metrics across all 4 phases
- **Skill management** — `hermes skills list`, `hermes skills config`, `hermes skills enable/disable`
- **Hermes profiling** — `hermes insights` shows usage analytics across sessions

> **Pitfall**: When loading multiple skills, use **comma-separated** names with `--skills`, not spaces.  
> `hermes --skills run-the-trace-extended,skill_evaluator` — correct  
> `hermes --skills run-the-trace-extended skill_evaluator` — Hermes parses `skill_evaluator` as a CLI command, not a skill name.

> **Typical interactive flow** (shortened):
> ```bash
> $ hermes --skills run-the-trace-extended
> ? Provide a task: Research Python web frameworks with security focus and compare them
> 
> [Phase 1 - Without-skill baseline run starts] … (Hermes uses minimal tools only, first run)
> [Phase 2 - With target skill run starts] … (Hermes uses target skill only)
> [Phase 3 - Full stack minus target skill run starts] … (Hermes uses all skills except target)
> [Phase 4 - Full stack with target skill run starts] … (Hermes uses all skills including target)
> 
> ✓ Comparison printed above (4-phase delta analysis)
> ✓ Trace saved to ~/.hermes/traces/run-the-trace-extended_20261020_143000.json
> ```
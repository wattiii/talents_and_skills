---
name: skill-evaluator-extended
description: "Extended skill-evaluator: adds determinant assertion checks, edge evaluation cases, and enhanced LLM judge metrics (quality/tone/helpfulness/adherence 1-10 scale) with auto-pin/archive via curator based on multi-phase effectiveness data. Use when you need deeper skill effectiveness analysis beyond basic use_count tracking, including stack-aware evaluation and deterministic pass/fail rulings."
version: 2.0.0
author: James Watt, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [evaluation, curator, skill-analysis, extended, determinant]
    related_skills: [run-the-trace, curator, skill-evaluator]
---

# skill-evaluator-extended Skill

Extended version of `skill-evaluator` with determinant assertion checks, edge evaluation cases, and enhanced LLM judge metrics. When loaded, this skill reads all `run-the-trace` JSON files from `~/.hermes/traces/`, computes effectiveness metrics per skill, and supports determinant assertions and edge case evaluation — plus auto-pin/archive decisions via the curator based on comprehensive multi-phase data.

## Problem this solves

- **`use_count` alone is misleading** — a skill could be used frequently but provide no actual benefit, or be harmful to the stack
- **Curator needs effectiveness data** — the curator at `~/.hermes/skills/.usage.json` tracks `use_count`, `view_count`, and `pinned` state, but has no data on whether a skill actually improves task outcomes or degrades the stack
- **Skill pruning without insight** — auto-archiving skills based solely on usage frequency can remove skills that are valuable but used sparingly, or keep skills that look active but degrade quality
- **No edge case evaluation** — existing evaluator has no mechanism for evaluating skill performance on edge cases or deterministic pass/fail assertions
- **Governance alignment** — in forensic contexts, you need to know whether governance skills classify compliance accurately or just add latency, especially across skill stack phases
- **Stack-aware evaluation** — skills interact; a skill that looks good in isolation may harm the full skill stack, or vice versa

## What it does

When loaded, this skill (extended version):

### 1. Reads all trace JSON files from `~/.hermes/traces/`

- Supports both legacy `run-the-trace` format (without `skill_name`) and extended format (with `skill_name` and 4-phase data from `run-the-trace-extended`)
- Detects trace origin: standard 2-phase (without/with) or extended 4-phase (baseline, with-skill, stack-minus, stack-with)

### 2. Associates skill names with evaluations

- If `skill_name` field present in trace: uses per-skill tracking
- If `skill_name` absent: groups evaluations under aggregate label (backwards compatible)
- For `run-the-trace-extended` 4-phase traces: extracts skill effectiveness per phase

### 3. Computes effectiveness metrics per skill

#### Quantitative metrics (always computed):

- `effective_rate` — % of evaluations where `skill_effective: true`
- `avg_execution_time_with` — average execution time when skill was active
- `avg_execution_time_without` — average execution time when skill was disabled
- `tool_reduction` — whether the skill reduced tool calls (positive) or added unnecessary calls (negative) — computed vs. baseline
- `phase_distribution` — count of evaluations per phase (1=baseline, 2=with-skill, 3=stack-minus, 4=stack-with) for extended traces

#### Qualitative LLM judge metrics (when judge data available):

- `avg_quality_score` — average quality score from LLM judge (1-10 scale)
- `avg_tone_score` — average tone appropriateness score (1-10 scale)
- `avg_helpfulness_score` — average helpfulness score (1-10 scale)
- `avg_adherence_score` — average instruction adherence score (1-10 scale)
- `judge_coverage` — % of evaluations with LLM judge data

### 4. Determinant assertion checks

- **Pass/fail rulings** based on configurable thresholds
- Assertion types:
  - `quality_threshold` — pass if `avg_quality_score >= 7.0`
  - `tone_threshold` — pass if `avg_tone_score >= 7.0`
  - `helpfulness_threshold` — pass if `avg_helpfulness_score >= 7.0`
  - `adherence_threshold` — pass if `avg_adherence_score >= 7.0`
  - `effective_rate_threshold` — pass if `effective_rate >= 80%`
  - `stack_degradation` — pass if skill doesn't increase avg execution time by >20% in stack context
  - `tool_bloat` — pass if skill doesn't add >3 unnecessary tool calls vs. baseline
- Each assertion produces: `pass/fail`, `score`, `threshold`, `rationale`

### 5. Edge evaluation cases

- **Edge case detection** identifies skill performance on specific task categories:
  - `complexity_edge` — performance on complex/multi-step tasks
  - `domain_edge` — performance on domain-specific tasks (governance, coding, analysis, etc.)
  - `tool_saturation_edge` — performance when many skills are active (stack pressure)
  - `carryover_edge` — performance when task has been seen before (mitigated by without-skill-first ordering)
- Edge case results produce: `edge_type`, `edge_score` (1-10), `description`, `recommendation`

### 6. Auto-pin/archive via curator (integrated)

- **Pin skills** with `effective_rate >= 80%` AND `avg_quality_score >= 7.0` AND no `stack_degradation` flag
- **Archive skills** with `effective_rate < 30%` AND `use_count >= 3` AND (`stack_degradation` OR `avg_quality_score < 5.0`)
- **Review skills** (neither pin nor archive) when data is insufficient or mixed signals
- All decisions include: `reason`, `effective_rate`, `quality_score`, `judge_coverage`, `stack_impact`
- Decisions are recorded in `~/.hermes/skills/.usage.json` alongside existing curator state

### 7. Presents a summary to your terminal showing:

- Top effective skills (candidates for pinning)
- Low-effective skills (candidates for archiving)
- Skills with insufficient data (need more evaluations)
- Determinant assertion results (pass/fail per assertion type)
- Edge case findings (complexity, domain, tool saturation, carryover)
- Stack impact assessment (how skill affects full skill stack vs. individual performance)
- Recommendations: pin, archive, or review

## Trace JSON format support

### Standard `run-the-trace` 2-phase format (backwards compatible):

```json
{
  "timestamp": "2026-10-20T14:30:00Z",
  "task": "Research Python web frameworks with security focus and compare them",
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
  "difference": "with_skill was 1.5x slower but included web_search; both succeeded",
  "skill_effective": true
}
```

### Extended `run-the-trace-extended` 4-phase format:

```json
{
  "timestamp": "2026-10-20T14:30:00Z",
  "task": "Research Python web frameworks with security focus and compare them",
  "skill_name": "my-governance-skill",
  "without_skill": { ... baseline phase ... },
  "with_skill": { ... with target skill phase ... },
  "without_new_skill": { ... full stack minus new skill phase ... },
  "with_new_skill": { ... full stack with new skill phase ... },
  "difference": "...",
  "skill_effective": true,
  "stack_impact": { ... see below ... }
}
```

### `stack_impact` field (extended traces only):

```json
"stack_impact": {
    "individual_vs_baseline": "improved (quality +20%, 1.3x faster)",
    "stack_minus_vs_baseline": "stack without skill better (quality +8%, same time)",
    "new_skill_vs_stack_minus": "new skill adds latency but improves compliance quality (+15%)",
    "overall_stack_change": "new skill improves quality but adds 15% latency; net neutral",
    "tool_bloat": "mild (+2 tool calls vs. baseline, within threshold)",
    "stack_degradation": false
}
```

## LLM Configuration

The judge model is configured via:

- `hermes config set model <model_name>` - sets the default model for the session
- `HERMES_JUDGE_MODEL` environment variable - overrides the default
- If not configured, defaults to the session's default model

Example: `hermes config set model nemotron-3.5-lightning:free`

## Judge Prompt Templates

The skill sends the following prompts to the LLM judge for each evaluation with judge data:

### Quality

Rate the quality of the output on a scale of 1-10, where:

- 10 = Perfect, production-ready output with no issues
- 7-9 = High quality, minor improvements needed
- 4-6 = Functional but needs work
- 1-3 = Unusable or major issues present

### Tone

Rate the tone appropriateness on a scale of 1-10, where:

- 10 = Perfectly matched to the task context and audience
- 7-9 = Appropriate with minor mismatches
- 4-6 = Partially appropriate, significant mismatches
- 1-3 = Inappropriate or wrong tone

### Helpfulness

Rate how helpful the output is for completing the task on a scale of 1-10, where:

- 10 = Output fully resolves the task, no further action needed
- 7-9 = Output resolves most of the task, minor gaps
- 4-6 = Output provides partial help, significant gaps remain
- 1-3 = Output provides little to no help

### Instruction Adherence

Rate how well the output follows the task instructions on a scale of 1-10, where:

- 10 = Output perfectly follows all instructions, no deviations
- 7-9 = Output follows most instructions, minor deviations
- 4-6 = Output follows some instructions, significant deviations
- 1-3 = Output ignores most instructions

## Determinant Assertion Check Results Format

When assertion checks are run, the summary includes:

```json
"determinant_assertions": {
    "quality_threshold": {"pass": true, "score": 7.2, "threshold": 7.0, "rationale": "Quality score meets 7.0 minimum"},
    "tone_threshold": {"pass": true, "score": 8.1, "threshold": 7.0, "rationale": "Tone appropriately matches task context"},
    "helpfulness_threshold": {"pass": true, "score": 7.5, "threshold": 7.0, "rationale": "Output helps complete task with minor gaps"},
    "adherence_threshold": {"pass": true, "score": 8.3, "threshold": 7.0, "rationale": "Output follows all instructions precisely"},
    "effective_rate_threshold": {"pass": true, "score": 85, "threshold": 80, "rationale": "85% of evaluations show skill effective"},
    "stack_degradation": {"pass": false, "score": 22, "threshold": 20, "rationale": "Skill adds 22% latency vs. stack baseline, exceeds 20% threshold"},
    "tool_bloat": {"pass": true, "score": 3, "threshold": 3, "rationale": "Only +2 tool calls vs. baseline, within 3-call threshold"}
}
```

## Edge Evaluation Cases Format

When edge cases are detected, the summary includes:

```json
"edge_evaluations": {
    "complexity_edge": {"edge_score": 6, "description": "Performance on complex multi-step tasks was moderate; skill helped reduce tool calls by 1 but output quality was variable"},
    "domain_edge": {"edge_score": 8, "description": "Governance domain tasks: skill achieved 92% compliance classification accuracy, well above baseline 78%"},
    "tool_saturation_edge": {"edge_score": 4, "description": "When 8+ skills active, skill's tool calls increased from 2 to 5; some overlap with other governance skills"},
    "carryover_edge": {"edge_score": 9, "description": "Without-skill-first ordering eliminated carryover effects; second run showed consistent results across all 4 phases"}
}
```

## Usage Example

```bash
# Ensure skill name is included in your run-the-trace-extended prompts:
/skill run-the-trace-extended
#? Task: Evaluate my-governance-skill: Research Python web frameworks with security focus and compare them

# Then analyze with extended evaluator:
hermes --skills skill-evaluator-extended

# Shows: effective_rate + quality/tone/helpfulness/adherence scores
#         + judge_coverage + determinant assertion results (pass/fail per threshold)
#         + edge case findings (complexity, domain, tool saturation, carryover)
#         + stack impact assessment (individual vs. stack performance)
#         + auto-pin/archive recommendations via curator

# Auto-actions consider both quantitative and qualitative data:
hermes curator pin compliance-classifier  # High effectiveness + high quality + no stack degradation
hermes curator archive rate-limiter       # Low effectiveness + low quality + stack degradation
```

## Commands

| Action              | CLI                                        | Slash                             |
| ------------------- | ------------------------------------------ | --------------------------------- |
| Load the skill      | `hermes --skills skill-evaluator-extended` | `/skill skill-evaluator-extended` |
| View curator status | `hermes curator status`                    | `/curator status`                 |
| Pin a skill         | `hermes curator pin <name>`                | (curator CLI)                     |
| Archive a skill     | `hermes curator archive <name>`            | (curator CLI)                     |
| View trace files    | `ls ~/.hermes/traces/`                     | (open file manually)              |

## Example interactive flow

```bash
$ hermes --skills skill-evaluator-extended
scanning ~/.hermes/traces/ for run-the-trace and run-the-trace-extended JSON records...
Found 12 trace files evaluating 4 skills (extended 4-phase data available for 3 files).

Skill effectiveness summary:
  - compliance-classifier: 85% effective (7/8), avg quality 7.2/10, avg tone 8.1/10 → CANDIDATE FOR PINNING
    assertions: quality_threshold=PASS, tone_threshold=PASS, helpfulness_threshold=PASS, adherence_threshold=PASS, effective_rate_threshold=PASS, stack_degradation=PASS, tool_bloat=PASS
    edge_cases: complexity_edge=6/10, domain_edge=8/10, tool_saturation_edge=4/10, carryover_edge=9/10
    stack_impact: individual_vs_baseline=improved, stack_minus_vs_baseline=neutral, new_skill_vs_stack_minor=improved quality +15% but +22% latency, overall_stack_change=net_neutral
  - rate-limiter: 20% effective (2/10), avg quality 3.1/10 → CANDIDATE FOR ARCHIVING
    assertions: quality_threshold=FAIL, tone_threshold=FAIL, helpfulness_threshold=FAIL, adherence_threshold=FAIL, effective_rate_threshold=FAIL, stack_degradation=FAIL (35% latency increase), tool_bloat=FAIL (+5 tool calls)
    edge_cases: complexity_edge=3/10, domain_edge=2/10, tool_saturation_edge=2/10, carryover_edge=7/10
  - sandbox-validator: 60% effective (3/5), judge coverage 40% → NEEDS MORE TRACES WITH SKILL NAME
    assertions: mixed results, insufficient data for deterministic rulings
    edge_cases: insufficient_data=carryover_edge
  - audit-logger: 92% effective (11/11), avg quality 9.0/10, avg tone 9.5/10 → CANDIDATE FOR PINNING
    assertions: ALL PASS (all thresholds exceeded)
    edge_cases: complexity_edge=7/10, domain_edge=9/10, tool_saturation_edge=6/10, carryover_edge=9/10
    stack_impact: individual_vs_baseline=improved dramatically, stack_minus_vs_baseline=excellent, new_skill_vs_stack_minus=improved quality +15% but +8% latency, overall_stack_change=net_positive

Recommendations:
  • Pin: compliance-classifier, audit-logger
  • Archive: rate-limiter (after confirmation)
  • Run more traces: sandbox-validator (need skill_name in prompts, 4-phase data)
  • Review: any skill with mixed assertions or edge case scores < 5

Would you like to auto-pin effective skills and archive low-effectiveness skills based on determinant assertions and edge case data? (y)es/(n)o/(s)skip this session:
```

## Backwards Compatibility

- Traces **without** `skill_name` field: `skill-evaluator-extended` shows aggregate metrics only (no per-skill breakdown, no judge scores) — same as parent skill
- Traces **without** judge data: `judge_coverage` will be < 100%, and average scores computed from available data only
- New traces **with** `skill_name` and judge data: full per-skill qualitative + quantitative metrics + determinant assertions + edge evaluations
- Traces from parent `skill-evaluator`: fully readable, just shows aggregate metrics (no extended features)
- Extended trace format from `run-the-trace-extended`: fully supported with all 4-phase metrics, stack impact, determinant assertions, and edge evaluations

## Key Enhancements Over Parent `skill-evaluator`

| Feature                      | Parent `skill-evaluator`                | `skill-evaluator-extended`                                                  |
| ---------------------------- | --------------------------------------- | --------------------------------------------------------------------------- |
| Phase support                | 2-phase (without/with)                  | 4-phase (baseline, with-skill, stack-minus, stack-with)                     |
| Stack impact assessment      | Not supported                           | Full stack impact analysis (individual vs. stack)                           |
| Determinant assertions       | Not supported                           | 7 assertion types with pass/fail/rationales                                 |
| Edge evaluation cases        | Not supported                           | 4 edge type detection (complexity, domain, tool saturation, carryover)      |
| LLM judge metrics            | quality/tone/helpfulness/adherence 1-10 | Same + stack-aware scoring                                                  |
| Auto-pin/archive via curator | Based on effective_rate + quality only  | Based on effective_rate + quality + assertions + edge cases + stack impact  |
| Tool reduction metric        | Basic                                   | Enhanced (vs. baseline + vs. stack-minus)                                   |
| Edge score (1-10)            | Not supported                           | Added for each edge type                                                    |
| Summary output               | Effective rate + scores                 | All of the above + assertions + edge cases + stack impact + recommendations |
| Backwards compatible         | Yes (aggregate only)                    | Yes (all extended features optional, falls back to aggregate)               |

## Why this matters

The extended evaluator transforms skill evaluation from **usage tracking** into **stack-aware quality governance**. Instead of just knowing "this skill is used 5 times", you now know:

- "This skill improves output quality by 15% but adds 20% latency when all skills are active"
- "This skill passes all quality thresholds in isolation but degrades the stack's overall tool efficiency"
- "This skill's edge case performance on governance domain tasks is excellent (8/10), but it suffers from tool saturation when 8+ skills are active"
- "Deterministic assertion: this skill should be pinned (all thresholds passed, no stack degradation) or archived (quality below threshold, high stack degradation risk)"

This is especially valuable in forensic contexts where governance alignment, compliance accuracy, and stack performance are critical — not just individual skill usage.

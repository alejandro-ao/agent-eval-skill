---
name: agent-eval
description: Evaluate a single agent session trace (or live session) against a standardized rubric with deterministic checks and LLM-as-judge grading. Detect skill invocation, score behavior across dimensions, log results for trend tracking, and suggest harness improvements. Use when analyzing one agent session, grading a completed task, or building a living eval suite.
---

# Agent Eval Skill

Evaluate a single agent session against a living, editable criteria list. Grade behavior, detect skill invocation, log scores, and generate improvement recommendations.

## When to Use This Skill

- Analyze a completed agent session export (trace file, JSONL, or markdown)
- Grade a live agent session after it finishes
- Check whether the agent invoked the correct skill when prompted
- Build and maintain a living eval criteria list
- Track scaffold improvement over time via logged scores
- Generate eval candidates and harness fix recommendations

## Quick Start

```bash
# Grade a session trace file
/skill:agent-eval /path/to/session-export.md

# Grade and tag the harness version
/skill:agent-eval --tag "prompt-v2.1" /path/to/session-export.md

# Grade with custom criteria file
/skill:agent-eval --criteria ~/my-criteria.json /path/to/session-export.md

# Grade a live session (read from tmux buffer or stdin)
cat session.md | /skill:agent-eval --stdin
```

## Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| (positional) | Path to session trace file | required unless `--stdin` |
| `--stdin` | Read trace from stdin instead of file | `false` |
| `--tag` | Harness version label | git SHA or timestamp |
| `--criteria` | Path to custom eval criteria JSON | `config/default-criteria.json` |
| `--task` | Task description (if not inferable from trace) | inferred from first user message |
| `--expected-skill` | Skill name the agent should have invoked | none |
| `--judge-model` | Model for LLM-as-judge grading | `gpt-4o` |
| `--output-dir` | Where to write report and log | `~/.pi/agent/evals/` |
| `--no-grade` | Skip LLM grading, run deterministic checks only | `false` |

## How It Works

### Phase 1: Ingest the Trace

Read the session trace. Supported formats:
- **Markdown** (`.md`) — pi session exports, conversation logs
- **JSONL** (`.jsonl`) — structured event streams (Codex `--json` output)
- **Plain text** — any readable transcript

Extract:
- Task description (first user message)
- Agent responses
- Tool calls and their arguments
- File changes
- Commands executed
- Skill invocations (detected via `/skill:name` or `$skill-name` patterns)

### Phase 2: Run Deterministic Checks

Fast, rule-based checks that require no LLM. These produce **pass/fail** results with concrete evidence.

| Check | What It Detects | Source |
|-------|----------------|--------|
| **Skill Invocation** | Did agent call the expected skill? | Trace text patterns |
| **Command Execution** | Did it run expected commands? | Tool call args |
| **File Creation** | Were expected files created? | File system or trace |
| **File Modification** | Were expected files modified? | File system or trace |
| **Test Execution** | Did it run tests? | Command patterns in trace |
| **Loop Detection** | Same file edited > N times? | Tool call sequence |
| **Parallelization** | Independent calls batched? | Tool call timing/sequence |
| **Verification** | Did it read output / compare to spec? | Response content patterns |

These checks use the **living criteria list** from `config/default-criteria.json`.

### Phase 3: LLM-as-Judge Grading (Rubric-Based)

For dimensions that need **semantic understanding**, we use a **structured rubric system** — not raw LLM scoring.

**Why rubric-based?** Raw LLM scores hallucinate, drift, and provide no audit trail. Rubric-based scoring forces the LLM to:
1. Match evidence to explicit level definitions
2. Quote specific evidence from the trace
3. Explain its reasoning
4. Self-report confidence

#### How It Works

Each `llm_judge` criterion defines **discrete levels** with explicit criteria:

```json
{
  "score": 1.0,
  "label": "Fully Autonomous",
  "criteria": "Agent completes task with ZERO user interventions",
  "evidence_required": "Confirm no user messages after initial prompt"
}
```

The judge model:
1. Reads the trace
2. Evaluates each rubric level against quoted evidence
3. Selects the **highest level fully supported by evidence**
4. Returns structured output with quotes, reasoning, and confidence

#### Judge Output Format

```json
{
  "criterion_id": "minimal-hand-holding",
  "score": 0.50,
  "level_matched": "Moderate Guidance",
  "confidence": "high",
  "evidence": [
    {
      "turn": 5,
      "type": "clarification",
      "quote": "User: 'Actually, I meant fix the redirect AFTER login'",
      "impact": "Agent was fixing wrong redirect"
    }
  ],
  "reasoning": "Agent required 3 interventions...",
  "disagreement_flags": []
}
```

#### Self-Consistency Check

For critical criteria, run the judge **3 times** and check variance:
- Variance < 0.15 → accept median score
- Variance ≥ 0.15 → flag for manual review

This catches hallucination and calibration drift.

#### Dimension Scoring

| Dimension | Weight | Graded By |
|-----------|--------|-----------|
| **Correctness** | 40% | LLM rubric (task solved?) + deterministic assertions |
| **Efficiency** | 25% | Deterministic heuristics + LLM rubric (hand-holding) |
| **Tool Use** | 20% | Deterministic heuristics + LLM rubric (appropriate choices) |
| **Verification** | 15% | Deterministic heuristics + LLM rubric (thoroughness) |

**Overall** = weighted average (same formula as `agent-benchmark`).

**Key principle:** No single LLM judgment dominates. Each dimension combines deterministic anchors with semantic judgment.

### Phase 4: Skill Invocation Detection

A special check that runs first:

```
Expected skill: "setup-demo-app"
Detected invocations: ["setup-demo-app", "file-read"]
Result: PASS — correct skill was invoked
```

If the agent was expected to invoke a skill but didn't:
```
Expected skill: "setup-demo-app"
Detected invocations: []
Result: FAIL — skill was not invoked
Impact: Correctness score capped at 0.50
```

This is critical because if the wrong skill (or no skill) was invoked, the rest of the evaluation may be meaningless.

### Phase 5: Log to JSONL

Append to `~/.pi/agent/evals/results.jsonl`:

```json
{
  "eval_id": "eval-20260604-153022-a1b2c3d",
  "timestamp": "2026-06-04T15:30:22Z",
  "trace_path": "/path/to/session-export.md",
  "task": "Kickstart the workflow",
  "tag": "prompt-v2.1",
  "git_sha": "a1b2c3d",
  "expected_skill": "kickstart-workflow",
  "skill_invoked": true,
  "skill_name": "kickstart-workflow",
  "scores": {
    "correctness": 0.75,
    "efficiency": 0.80,
    "tool_use": 0.90,
    "verification": 0.50,
    "overall": 0.73
  },
  "deterministic_checks": {
    "skill_invocation": { "pass": true, "expected": "kickstart-workflow", "found": "kickstart-workflow" },
    "ran_tests": { "pass": false, "evidence": "no test commands found" },
    "created_expected_files": { "pass": true, "files": ["src/App.tsx"] },
    "no_edit_loops": { "pass": true, "max_edits": 2 }
  },
  "failure_modes": ["skipped_tests", "no_edge_case_check"],
  "recommendations": [
    {
      "priority": "critical",
      "category": "verification",
      "issue": "Agent did not run tests before declaring success",
      "suggested_fix": "Add PreCompletionChecklistMiddleware enforcing test execution"
    }
  ],
  "eval_candidates": [
    "agent_runs_tests_before_submit"
  ]
}
```

### Phase 6: Generate Report

Write `{output-dir}/{eval-id}/report.md`:

```markdown
# Agent Eval Report

## Session Metadata
- **Eval ID:** eval-20260604-153022-a1b2c3d
- **Task:** Kickstart the workflow
- **Tag:** prompt-v2.1
- **Expected Skill:** kickstart-workflow
- **Skill Invoked:** ✅ Yes

## Scores

| Dimension | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Correctness | 0.75 | 40% | 0.30 |
| Efficiency | 0.80 | 25% | 0.20 |
| Tool Use | 0.90 | 20% | 0.18 |
| Verification | 0.50 | 15% | 0.075 |
| **Overall** | | | **0.73** |

## Deterministic Checks

| Check | Result | Evidence |
|-------|--------|----------|
| Skill invocation | ✅ PASS | Found `/skill:kickstart-workflow` at turn 2 |
| Ran tests | ❌ FAIL | No test commands in trace |
| Created expected files | ✅ PASS | src/App.tsx, package.json |
| No edit loops | ✅ PASS | Max 2 edits per file |

## Failure Modes
- skipped_tests
- no_edge_case_check

## Recommendations

### Critical
1. **Add test verification prompt** — agent stopped at "looks good" without running tests

## Eval Candidates
- `agent_runs_tests_before_submit` — tag: verification
```

## The Living Criteria List

The eval criteria are stored in `config/default-criteria.json`. This is **editable** — you and the agent can update it as new edge cases are discovered.

### Criteria Format

```json
{
  "version": "2026-06-04",
  "criteria": [
    {
      "id": "skill-invocation",
      "name": "Skill Invocation",
      "description": "Agent invokes the expected skill when prompted",
      "category": "trigger",
      "weight": 0.0,
      "check_type": "pattern",
      "pattern": "/skill:{expected_skill}|\\${expected_skill}",
      "required": true,
      "failure_impact": "cap_correctness_at_0.5"
    },
    {
      "id": "ran-tests",
      "name": "Test Execution",
      "description": "Agent runs tests before declaring success",
      "category": "verification",
      "weight": 0.0,
      "check_type": "command_pattern",
      "patterns": ["pytest", "npm test", "cargo test", "go test", "jest"],
      "required": false
    },
    {
      "id": "no-edit-loops",
      "name": "No Edit Loops",
      "description": "Agent does not edit the same file more than 3 times without verification",
      "category": "efficiency",
      "weight": 0.0,
      "check_type": "sequence",
      "max_repetitions": 3,
      "tool": "edit",
      "required": false
    },
    {
      "id": "parallelized-reads",
      "name": "Parallelized Reads",
      "description": "Agent batches independent file reads",
      "category": "efficiency",
      "weight": 0.0,
      "check_type": "parallelization",
      "tool": "read",
      "required": false
    },
    {
      "id": "read-command-output",
      "name": "Read Command Output",
      "description": "Agent reads the output of commands it runs",
      "category": "verification",
      "weight": 0.0,
      "check_type": "response_pattern",
      "patterns": ["output shows", "result is", "tests passed", "tests failed", "error:"],
      "required": false
    }
  ]
}
```

### Adding New Criteria

When you discover a new failure mode:

1. Add a criterion to `config/default-criteria.json`
2. Re-run evals to see if they catch it
3. If useful, keep it; if noisy, remove or adjust

#### Deterministic Criterion Example

```json
{
  "id": "descriptive-commit",
  "name": "Descriptive Commit",
  "description": "Agent uses descriptive git commit messages, not 'update' or 'fix'",
  "category": "style",
  "check_type": "command_pattern",
  "patterns": ["git commit -m"],
  "forbidden_patterns": ["git commit -m 'update'", "git commit -m 'fix'"],
  "required": false
}
```

#### Rubric-Based LLM Judge Criterion Example

```json
{
  "id": "proactive-behavior",
  "name": "Proactive Behavior",
  "description": "Agent anticipates problems and addresses them before they become issues",
  "category": "correctness",
  "check_type": "llm_judge",
  "rubric": {
    "levels": [
      {
        "score": 1.0,
        "label": "Highly Proactive",
        "criteria": "Agent identifies potential issues before they occur and takes preventive action without prompting.",
        "evidence_required": "Quote where agent anticipated a problem and acted preventively."
      },
      {
        "score": 0.75,
        "label": "Somewhat Proactive",
        "criteria": "Agent notices one potential issue and addresses it, but may miss others.",
        "evidence_required": "Quote the proactive action and note what was missed."
      },
      {
        "score": 0.50,
        "label": "Reactive",
        "criteria": "Agent only addresses issues after they cause problems or failures.",
        "evidence_required": "Quote where agent reacted to an error that could have been prevented."
      },
      {
        "score": 0.25,
        "label": "Slow to React",
        "criteria": "Agent takes multiple failures before adjusting approach.",
        "evidence_required": "Count failures before recovery and quote the adjustment."
      },
      {
        "score": 0.0,
        "label": "Never Adapts",
        "criteria": "Agent repeats the same failing approach without recognizing the problem.",
        "evidence_required": "Quote repeated identical failed attempts."
      }
    ],
    "self_consistency_runs": 3,
    "variance_threshold": 0.15
  },
  "contributes_to_dimension": "correctness",
  "dimension_weight": 0.20,
  "required": false
}
```

### Criteria Categories

| Category | Purpose |
|----------|---------|
| `trigger` | Did the right skill/tool trigger? |
| `correctness` | Did the task complete correctly? |
| `verification` | Did the agent verify its work? |
| `efficiency` | Was the path optimal? |
| `style` | Did output follow conventions? |
| `safety` | Did it avoid dangerous operations? |

## Interpreting Results

### Skill Invocation Failure

If `skill_invoked` is `false`:
- The agent didn't call the expected skill
- This is a **harness issue** (skill description/name not clear enough) or a **model issue**
- Correctness is capped at 0.50 because the agent may have solved the task via general reasoning instead of the intended workflow

### Score Benchmarks

| Overall | Interpretation |
|---------|---------------|
| 0.90+ | Excellent — minimal friction |
| 0.75-0.89 | Good — some room for improvement |
| 0.60-0.74 | Fair — noticeable issues, user likely intervened |
| 0.40-0.59 | Poor — task completed but with significant problems |
| < 0.40 | Failed — task not completed or completely wrong |

## Integration with agent-benchmark

This skill shares the same:
- **Scoring rubric** (4 dimensions, same weights)
- **JSONL log format** (can be read by `plot-trends.py`)
- **Eval candidate generation**

Use `agent-eval` for:
- Single-session analysis
- Building and refining criteria
- Quick checks after a harness change

Use `agent-benchmark` for:
- Multi-model comparison
- Regression testing across models
- Scheduled/CI evaluation

## Tracking Scaffold Improvement

To see if your scaffold is improving:

```bash
# Run eval on same task after each harness change
/skill:agent-eval --tag "baseline" --task "Kickstart workflow" session-v1.md
/skill:agent-eval --tag "prompt-v2" --task "Kickstart workflow" session-v2.md
/skill:agent-eval --tag "middleware-v3" --task "Kickstart workflow" session-v3.md

# Plot trends
python3 ~/.pi/agent/skills/agent-benchmark/scripts/plot-trends.py \
  --input ~/.pi/agent/evals/results.jsonl \
  --task "Kickstart workflow" \
  --output scaffold-improvement.png
```

The x-axis shows tags (harness versions), the y-axis shows scores. You can see if each change helped or hurt.

## Files Reference

| File | Purpose |
|------|---------|
| `config/default-criteria.json` | Living eval criteria list — deterministic + rubric-based LLM judge criteria |
| `references/scoring-rubric.md` | Detailed grading prompts (shared with agent-benchmark) |
| `references/eval-template.py` | pytest stub for generated evals (shared with agent-benchmark) |
| `references/judge-prompt-template.md` | Standardized LLM-as-judge prompt with rubric levels, evidence requirements, and self-consistency protocol |

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Skill not detected | Pattern too strict | Update `pattern` in criteria JSON |
| False positive on skill | Pattern too loose | Make pattern more specific |
| Criteria JSON invalid | Syntax error | Validate with `python -m json.tool` |
| Judge model bias | Consistent over/under scoring | Try different `--judge-model` |
| High variance in LLM scores | Judge is uncertain or rubric ambiguous | Review evidence, tighten rubric levels, or flag for manual review |
| Missing evidence in judge output | Trace too long or evidence scattered | Use `--task` to focus judge, or split trace into sections |
| Judge hallucinates quotes | LLM confabulation | Enable `self_consistency_runs: 3` and verify quotes exist in trace |

## Key Principles (from OpenAI Eval Skills)

1. **Measure what matters** — Good evals make regressions clear and failures explainable
2. **Start from a checkable definition of done** — Tighten instructions until success is unambiguous
3. **Ground evals in behavior** — Check what actually happened in the trace, not just final output
4. **Use deterministic checks first** — Fast, explainable, no LLM needed
5. **Use LLM grading where rules fall short** — Structured rubric for qualitative checks
6. **Let real failures drive coverage** — Every manual fix is a signal; turn it into a criterion

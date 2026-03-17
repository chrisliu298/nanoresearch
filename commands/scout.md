---
description: "Scout phase: survey literature, generate ideas, verify novelty, produce experiment spec. Use for 'nanoresearch:scout', 'find ideas', 'research direction'."
argument-hint: [research-topic]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, mcp__codex__codex, mcp__codex__codex-reply
---

# Scout: Topic → Experiment Specification

Research topic: **$ARGUMENTS**

Read REVIEWER_MODEL and SCOUT_BUDGET_MINUTES from the project's **CLAUDE.md**. Read `codex` field from `nanoresearch.json` (if it exists) to check for `codex: off` mode. Before composing any Codex MCP prompt, read **docs/prompting-codex.md** for model-specific patterns.

Output: `IDEA.md` + `EXPERIMENT_SPEC.md`. On failure: `SCOPING_MEMO.md` (pipeline halts).

## Budget

Record start time: `START_EPOCH=$(date +%s)`. Check elapsed time before each phase. If elapsed >= SCOUT_BUDGET_MINUTES, write `SCOPING_MEMO.md` with current progress and stop.

Cap resource usage: max 10 WebSearch queries total, max 15 abstracts read, max 5 full paper reads.

## Resumption

Check `nanoresearch.json.scout_state`:
- If `sub_phase == "survey"` → start Phase 1 (delete stale `LANDSCAPE.md`).
- If `sub_phase == "ideate"` and `LANDSCAPE.md` exists → skip to Phase 2.
- If `sub_phase == "ideate"` and `LANDSCAPE.md` does NOT exist → reset to `{sub_phase: "survey", novelty_confidence: "high"}`, start Phase 1.
- If `sub_phase == "specify"` → skip to Phase 3 (Phase 3 produces IDEA.md if it doesn't exist yet).
- If absent or unrecognized value → initialize: `{sub_phase: "survey", novelty_confidence: "high"}` (always initialize full schema-valid object).

## Phase 1: Survey

`scout_state.sub_phase = "survey"`

1. Scan `papers/` and `literature/` for local PDFs. Read first 3 pages of relevant ones.
2. WebSearch recent literature: top venues (last 2 years), arXiv preprints (last 6 months). Use 3-5 query formulations. Read abstracts of top 10 papers. If WebSearch fails after 2 retries, proceed with local literature only and mark novelty assessment as low-confidence.
3. Build landscape map: group by approach, identify gaps, note recurring "Future Work" limitations.
4. Identify structural gaps: methods untried in new domains, contradictory findings, untested assumptions.

**Checkpoint:** Save `LANDSCAPE.md`:
```markdown
# Literature Landscape

**Topic**: [topic]  **Date**: [today]

## Approach Groups
[grouped by method family]

## Gaps
[structural gaps identified]

## Key Papers
| Paper | Year | Relevance |
|-------|------|-----------|
```

**Validation gate:** `LANDSCAPE.md` must contain at least 2 identifiable gaps and reference at least 5 papers. If not, retry survey with different query formulations (once). If still insufficient, proceed but set `scout_state.novelty_confidence: "low"` in `nanoresearch.json` (this propagates to write/review phases for cautious novelty claims). If the gate passes, **explicitly set** `scout_state.novelty_confidence: "high"` (do not rely on default — a prior crashed run may have set it to "low").

Update `nanoresearch.json`: `scout_state.sub_phase: "ideate"`. Commit: `git add LANDSCAPE.md nanoresearch.json && git commit -m "scout: landscape survey"`.

## Phase 2: Ideate

`scout_state.sub_phase = "ideate"`

### Brainstorm

**Primary:** Use `mcp__codex__codex` (REVIEWER_MODEL, `config: {"model_reasoning_effort": "xhigh"}` — xhigh is the only permitted effort level):

```
Generate 5-8 concrete research ideas for: [topic]
Landscape: [paste map]. Gaps: [paste gaps].

For each: one-sentence summary, hypothesis, minimum experiment, risk (LOW/MED/HIGH).
Prioritize testable ideas where the answer matters either way.
```

**Fallback** (if Codex MCP unavailable or `codex: off`): brainstorm using Claude directly.

Save threadId for follow-up.

### Novelty Check

For each idea, 2-3 targeted WebSearch queries. If already published → eliminate.

### Filter and Rank

Eliminate: already published, requires unavailable resources, uninteresting regardless of outcome. Rank by: novelty × feasibility × clarity.

### Feasibility Pre-Screen

Before investing in devil's advocate refinement, verify the top idea passes a quick feasibility check:
- Can the metric be measured with available tools?
- Does a plausible command exist (even if rough)?
- Is the data/evaluation harness accessible?

If top idea fails pre-screen, try the next-ranked idea. If all fail → write `SCOPING_MEMO.md` and stop.

### Refine Top Idea

If threadId exists from brainstorm, ask REVIEWER_MODEL (via `mcp__codex__codex-reply`) to play devil's advocate on the top idea: strongest objection, likely failure mode, minimum convincing experiment, primary + sanity metric. `mcp__codex__codex-reply` inherits the originating thread's model and effort settings — this is valid only because the brainstorm `mcp__codex__codex` call used `config: {"model_reasoning_effort": "xhigh"}`. If no threadId (brainstorm fell back to Claude), use `mcp__codex__codex` (`config: {"model_reasoning_effort": "xhigh"}` — the only permitted effort level) for a new thread.

**Fallback** (if Codex MCP unavailable or `codex: off`): self-critique using Claude directly.

Update `nanoresearch.json`: `scout_state.sub_phase: "specify"`.

Commit: `git add nanoresearch.json && git commit -m "scout: ideation complete"`.

## Phase 3: Specify

`scout_state.sub_phase = "specify"`

### Hard Gate

All of these must be concrete before proceeding:

- **Metric**: measurable, with direction (lower/higher is better)
- **Sanity metric**: must not degrade (prevents proxy overfitting)
- **Command**: exact shell command for one experiment
- **Metric extraction**: how to parse metrics from output
- **Files in scope**: which files the agent may edit
- **Dataset/evaluator**: what data or evaluation harness is used
- **Baseline**: the starting point

**If any cannot be specified** → write `SCOPING_MEMO.md` (what we know, top ideas, what's missing, next steps). Stop. (The orchestrator handles `nanoresearch.json` state update.)

### Output: IDEA.md

```markdown
# Research Idea

**Topic**: [topic]  **Date**: [today]

## Landscape Summary
[3-5 paragraphs]

## Chosen Idea
- **Title**: [title]
- **Hypothesis**: [if X then Y because Z]
- **Contribution**: empirical / method / theory / diagnostic
- **Risk**: LOW / MEDIUM / HIGH
- **Novelty**: [closest work + differentiation]

## Alternatives
| Idea | Why not chosen |
|------|----------------|

## Reviewer Feedback
[devil's advocate key points]
```

### Output: EXPERIMENT_SPEC.md

```markdown
# Experiment Specification

## Objective
## Metric
- **Primary**: [name, unit, direction]
- **Sanity**: [name, bound]
## Command
## Metric Extraction
## Comparison Protocol
## Files in Scope
## Off Limits
## Constraints
## Guard (optional)
```

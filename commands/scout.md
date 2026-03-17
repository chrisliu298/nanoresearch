---
description: "Scout phase: survey literature, generate ideas, verify novelty, produce experiment spec. Use for 'nanoresearch:scout', 'find ideas', 'research direction'."
argument-hint: [research-topic]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, mcp__codex__codex, mcp__codex__codex-reply
---

# Scout: Topic → Experiment Specification

Research topic: **$ARGUMENTS**

Read REVIEWER_MODEL from the project's **CLAUDE.md**. Read `codex` field from `nanoresearch.json` (if it exists) to check for `codex: off` mode.

Output: `IDEA.md` + `EXPERIMENT_SPEC.md`. On failure: `SCOPING_MEMO.md` (pipeline halts).

## Phase 1: Survey (5-10 min)

1. Scan `papers/` and `literature/` for local PDFs. Read first 3 pages of relevant ones.
2. WebSearch recent literature: top venues (last 2 years), arXiv preprints (last 6 months). Use 3-5 query formulations. Read abstracts of top 10 papers. If WebSearch fails after 2 retries, proceed with local literature only and mark novelty assessment as low-confidence in `IDEA.md`.
3. Build landscape map: group by approach, identify gaps, note recurring "Future Work" limitations.
4. Identify structural gaps: methods untried in new domains, contradictory findings, untested assumptions.

## Phase 2: Ideate (10-15 min)

### Brainstorm

**Primary:** Use `mcp__codex__codex` (REVIEWER_MODEL, xhigh):

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

### Refine Top Idea

If threadId exists from brainstorm, ask REVIEWER_MODEL (via `mcp__codex__codex-reply`) to play devil's advocate on the top idea: strongest objection, likely failure mode, minimum convincing experiment, primary + sanity metric. If no threadId (brainstorm fell back to Claude), use `mcp__codex__codex` for a new thread.

**Fallback** (if Codex MCP unavailable or `codex: off`): self-critique using Claude directly.

## Phase 3: Specify

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

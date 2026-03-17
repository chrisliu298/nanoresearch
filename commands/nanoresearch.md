---
description: "Full autonomous ML research pipeline: scout ideas, run experiments, write a paper, survive peer review. Use when user says 'nanoresearch', 'autonomous research', or wants end-to-end ML research. One command, wake up to a paper."
argument-hint: [research-topic]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply
---

# nanoresearch: Scout → Loop → Write → Review

Research topic: **$ARGUMENTS**

## Constants

Read all constants from this project's **CLAUDE.md** before proceeding. Read the state schema and file contracts.

## Resumption

Check for `nanoresearch.json` in the project root:
- If absent → **fresh start**.
- If present but invalid JSON → rename to `nanoresearch.json.bak`, warn, **fresh start**.
- If `status` is `"completed"` or `"failed"` → **fresh start**.
- If `status` is `"in_progress"` AND `timestamp` > 24 hours old → **fresh start** (stale).
- If `status` is `"in_progress"` AND topic does NOT match `$ARGUMENTS` → **error**: "Another run in progress for '[topic]'. Complete or delete nanoresearch.json first."
- If `status` is `"in_progress"` AND topic matches AND within 24 hours → **resume** from recorded phase.

**On fresh start:**
1. Clean stale artifacts: `rm -f IDEA.md EXPERIMENT_SPEC.md results.tsv autoresearch.md AUTO_REVIEW.md RESEARCH_MEMO.md SCOPING_MEMO.md && rm -rf paper/`.
2. Create `nanoresearch.json` per the state schema: `topic: "<parsed topic from $ARGUMENTS>"`, `status: "in_progress"`, `phase: "scout"`, `branch: "nanoresearch/<tag>"`, `timestamp: <now ISO 8601>`, `venue: null`, `codex: "on"`, `decision: null`.
3. Resolve the default branch (`git symbolic-ref refs/remotes/origin/HEAD` or fall back to `main`). Then `git checkout -b nanoresearch/<tag> <default-branch>` (tag = date, e.g., `mar16`). If branch exists, append sequence number: `mar16-2`.

**On resume:**
1. Read `branch` from `nanoresearch.json`.
2. `git checkout <branch>` (not `-b`).
3. Update `timestamp` to now.
4. If resuming `phase: "scout"` → clean partial scout artifacts: `rm -f IDEA.md EXPERIMENT_SPEC.md SCOPING_MEMO.md`.
5. Jump to the recorded phase.

## Pipeline

### Phase 1: Scout

**Precondition:** None (or `skip-scout` flag with `EXPERIMENT_SPEC.md` present).

Invoke `/nanoresearch:scout "$ARGUMENTS"`.

If scout writes `SCOPING_MEMO.md` instead of `EXPERIMENT_SPEC.md` → update `nanoresearch.json`: `status: "failed"`, `phase: "scout_failed"`. `git add SCOPING_MEMO.md nanoresearch.json && git commit -m "scout: scoping memo (failed)"`. Print what's missing and stop.

If scout produces neither `EXPERIMENT_SPEC.md` nor `SCOPING_MEMO.md` → update `nanoresearch.json`: `status: "failed"`, `phase: "scout_failed"`. `git add nanoresearch.json && git commit -m "scout: no output (failed)"`. Print "Scout produced no output." Stop.

Update `nanoresearch.json`: `phase: "loop"`, `idea_summary: [title]`, `timestamp: <now>`.

**Commit:** `git add IDEA.md EXPERIMENT_SPEC.md nanoresearch.json && git commit -m "scout: [idea title]"`.

### Phase 2: Experiment Loop

**Precondition:** `EXPERIMENT_SPEC.md` exists. If missing, set `status: "failed"` and stop.

Invoke `/nanoresearch:loop` with the appropriate budget:
- If `budget` override specified → `/nanoresearch:loop <budget value>` (e.g., `/nanoresearch:loop 8h`)
- If `loop` override specified → `/nanoresearch:loop <N>` (e.g., `/nanoresearch:loop 50`)
- Otherwise → `/nanoresearch:loop ${EXPERIMENT_BUDGET_HOURS}h`

The loop reads `EXPERIMENT_SPEC.md` for setup, runs the autoresearch protocol (edit→commit→run→measure→keep/revert) until the budget is exhausted.

Update `nanoresearch.json`: `phase: "write"`, `best_metric: [value]`, `iteration_count: [N]`, `timestamp: <now>`.

**Commit:** `git add results.tsv autoresearch.md nanoresearch.json && git diff --cached --quiet || git commit -m "loop: [N] iterations, best=[metric]"`.

### Phase 3: Write

**Precondition:** `results.tsv` exists with at least 2 rows (header + baseline). If missing, set `status: "failed"` and stop.

Reset `write_state: {sub_phase: "section_drafting", current_section: 0, revision_pass: 0}` in `nanoresearch.json` (always reset on phase entry, not "if not already present" — ensures clean state for both initial write and resubmission cycles).

Invoke `/nanoresearch:write`.

If write produces `RESEARCH_MEMO.md` → update `nanoresearch.json`: `status: "completed"`, `phase: "completed"`, `decision: "memo"`. `git add RESEARCH_MEMO.md nanoresearch.json && git commit -m "write: research memo"`. Print summary and stop. (Memos skip review.)

Update `nanoresearch.json`: `phase: "review"`, `timestamp: <now>`, `review_state: {cycle: 1, sub_phase: "initial_review", decision: null, codex_threads: {}, scores: {initial: {}, post_rebuttal: {}}, score_history: []}`.

**Commit:** `git add paper/ nanoresearch.json && git commit -m "write: paper draft"`.

### Phase 4: Peer Review Cycle

**Precondition:** `paper/main.tex` exists. If missing, set `status: "failed"` and stop.

**Repeat up to MAX_REVIEW_CYCLES:**

1. Invoke `/nanoresearch:review`. The review command runs: 4 independent reviews → rebuttal (with optional experiments) → post-rebuttal rescoring → area chair decision. It returns a decision in `nanoresearch.json.review_state`.

2. Read `nanoresearch.json.review_state.decision`:
   - **"accepted"** → break, proceed to summary.
   - **"rejected"** → check bounds:
     - If `review_state.cycle >= MAX_REVIEW_CYCLES` → break.
     - Otherwise → **revise and resubmit:**
       1. The review command has already patched `EXPERIMENT_SPEC.md` with resubmission requirements.
       2. Increment `review_state.cycle`. Reset cycle-local fields: `review_state.sub_phase: "initial_review"`, `review_state.decision: null`, `review_state.codex_threads: {}`, `review_state.scores: {initial: {}, post_rebuttal: {}}`. (Write uses `cycle > 1` to detect revision mode, not `decision`.)
       3. Re-invoke `/nanoresearch:loop ${RESUBMISSION_EXPERIMENT_BUDGET_MINUTES}m`. The loop detects `## Resubmission Requirements` in `EXPERIMENT_SPEC.md` and enters fresh-from-spec mode.
       4. Re-invoke `/nanoresearch:write` (write detects revision mode from `review_state.cycle > 1`).
       5. `git add paper/ results.tsv autoresearch.md nanoresearch.json && git commit -m "revise: cycle [N]"`. Continue loop.

Update `nanoresearch.json`: `status: "completed"`, `phase: "completed"`, `decision: "accepted"/"rejected"`, `timestamp: <now>`.

### Phase 5: Summary

```
=== nanoresearch complete ===
Topic: [topic]
Idea: [idea_summary]
Experiments: [iteration_count] iterations
Best metric: [best_metric]
Output: paper/main.pdf | RESEARCH_MEMO.md
Decision: [decision] — [avg score]/10 after [N] cycle(s)
Duration: [total time]
```

## Inline Overrides

Parse `$ARGUMENTS` by splitting on ` — `. First segment = topic. Subsequent segments are either `key: value` pairs or bare flags (`skip-scout`, `skip-to-write`). Parse from the end: only segments matching a known override key or flag are treated as overrides. Everything before the first valid override is the topic. Reject unknown keys silently. Echo applied config at startup.

| Override | Effect |
|----------|--------|
| `budget: Nh` | Override EXPERIMENT_BUDGET_HOURS |
| `loop: N` | Bound experiment loop to N iterations |
| `venue: NeurIPS` | Store in nanoresearch.json, passed to write and review |
| `skip-scout` | Start from Phase 2. Set `phase: "loop"`, `idea_summary: "N/A"` in nanoresearch.json. Requires EXPERIMENT_SPEC.md. |
| `skip-to-write` | Start from Phase 3. Set `phase: "write"`, `idea_summary: "N/A"` in nanoresearch.json. Requires results.tsv. Parse results.tsv for `best_metric` and `iteration_count`. |
| `codex: off` | Store in nanoresearch.json. All Codex MCP calls replaced with Claude self-prompting. |

## Rules

- **AUTO_PROCEED = true.** Never block on user input.
- **Update `nanoresearch.json` BEFORE committing** on every phase transition.
- **Commit after each phase.** Scout outputs, loop results, paper draft.
- **On failure**, set `status: "failed"` with the phase name. Do not loop on failures.
- **Large files:** If Write tool fails, retry via Bash heredoc silently.
- **Budget tracking:** The loop tracks its own wall-clock start time independently of `nanoresearch.json.timestamp`.
- **Git errors:** If any git command fails, check for stale `.git/index.lock` (remove if no git process owns it, retry once). If merge conflict or unmerged paths, set `status: "failed"` and stop.

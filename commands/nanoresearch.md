---
description: "Full autonomous ML research pipeline: scout ideas, run experiments, write a paper, survive peer review. Use when user says 'nanoresearch', 'autonomous research', or wants end-to-end ML research. One command, wake up to a paper."
argument-hint: [research-topic]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply
---

# nanoresearch: Scout → Loop → Write → Review

Research topic: **$ARGUMENTS**

## Constants

Read all constants from this project's **CLAUDE.md** before proceeding. Read the state schema and file contracts.

## Startup

### Override Parsing (FIRST — before any cleanup or resumption)

Parse `$ARGUMENTS` by splitting on ` — `. First segment = topic. Subsequent segments are `key: value` pairs or bare flags. Parse from the end: only segments matching a known override key or flag are overrides. Everything before the first valid override is the topic. Reject unknown keys silently. Echo applied config at startup. Persist `budget`/`loop` as `loop_limit` in `nanoresearch.json` on creation so they survive resume.

| Override | Effect |
|----------|--------|
| `budget: Nh` | Override EXPERIMENT_BUDGET_HOURS |
| `loop: N` | Bound experiment loop to N iterations |
| `venue: NeurIPS` | Store in nanoresearch.json, passed to write and review |
| `skip-scout` | Start from Phase 2. Set `phase: "loop"`, `idea_summary: "N/A"` in nanoresearch.json. Requires EXPERIMENT_SPEC.md. |
| `skip-to-write` | Start from Phase 3. Set `phase: "write"`, `idea_summary: "N/A"` in nanoresearch.json. Requires `results.tsv` AND `EXPERIMENT_SPEC.md` (for metric direction). Parse results.tsv for `best_metric` and `iteration_count`. |
| `codex: off` | Store in nanoresearch.json. All Codex MCP calls replaced with Claude self-prompting. |

### Resumption

Check for `nanoresearch.json` in the project root:
- If absent → **fresh start**.
- If present but invalid JSON or missing required fields (`topic`, `status`, `phase`, `branch`) → rename to `nanoresearch.json.bak`, warn, **fresh start**.
- If `status` is `"completed"` or `"failed"` → **fresh start**.
- If `phase` is `"completed"` or `"scout_failed"` (regardless of `status`) → **fresh start**.
- If `status` is `"in_progress"` AND stale → **fresh start**. Staleness: `git log -1 --format=%cI <branch>` older than 24 hours.
- If `status` is `"in_progress"` AND topic does NOT match `$ARGUMENTS` → **error**: "Another run in progress for '[topic]'. Complete or delete nanoresearch.json first."
- If `status` is `"in_progress"` AND topic matches AND within 24 hours → **resume** from recorded phase.

**On fresh start:**
0. If old `nanoresearch.json` has `pre_loop_stash_ref` and the stash still exists, log warning with the ref for manual recovery — do NOT pop it here.
1. Resolve the default branch (`git symbolic-ref refs/remotes/origin/HEAD` or fall back to `main`). `git checkout -b nanoresearch/<tag> <default-branch>` (tag = date, e.g., `mar16`; append sequence number if exists). Abort if branch creation fails.
2. Clean stale artifacts on the new branch, preserving files required by active skip flags:
   - If `skip-scout`: preserve `EXPERIMENT_SPEC.md`.
   - If `skip-to-write`: preserve `EXPERIMENT_SPEC.md`, `results.tsv`, `autoresearch.md`.
   - Default (no skip): `rm -f LANDSCAPE.md IDEA.md EXPERIMENT_SPEC.md results.tsv autoresearch.md AUTO_REVIEW.md RESEARCH_MEMO.md SCOPING_MEMO.md && rm -rf paper/`.
   - Always clean crash-recovery artifacts: `rm -f .revert-pending *.bak *.episode-bak nanoresearch.json`.
3. Create `nanoresearch.json` per the state schema: `topic`, `status: "in_progress"`, `phase` (set by skip flag or `"scout"`), `branch: "nanoresearch/<tag>"`, `timestamp: <now ISO 8601>`, `venue`, `codex`, `decision: null`, `loop_limit` (if set).
   - If phase is `"scout"`: set `scout_state: {sub_phase: "survey", novelty_confidence: "high"}`.
   - If `skip-scout` (phase `"loop"`): set `scout_state: {sub_phase: "specify", novelty_confidence: "low"}`. No survey was done, so novelty confidence must be low.
   - If `skip-to-write` (phase `"write"`): set `scout_state: {sub_phase: "specify", novelty_confidence: "low"}`, initialize `write_state` to defaults. Validate `results.tsv` (header, ascending `#`, at least one `keep` row). Compute `best_metric` per metric direction from `EXPERIMENT_SPEC.md` and `iteration_count` from max `#`. Persist both.

**On resume:**
1. `git checkout <branch>` (not `-b`). Verify current branch matches.
2. Update `timestamp` to now.
3. **Halt artifact check:** If `phase == "scout"` and `SCOPING_MEMO.md` exists → finalize as failed. If `phase == "scout"` and both `EXPERIMENT_SPEC.md` and `IDEA.md` exist → skip scout, transition to `"loop"`. If `phase == "write"` and `RESEARCH_MEMO.md` exists → finalize as `decision: "memo"`.
4. If resuming `phase: "scout"` → clean artifacts downstream of `scout_state.sub_phase` (survey: clean all; ideate: preserve LANDSCAPE.md; specify: preserve LANDSCAPE.md + IDEA.md + EXPERIMENT_SPEC.md).
5. If resuming `phase: "write"` → delete stale `RESEARCH_MEMO.md` if `write_state.sub_phase != "complete"`.
6. Jump to the recorded phase.

## Pipeline

### Phase 1: Scout

**Precondition:** None (or `skip-scout` flag with `EXPERIMENT_SPEC.md` present).

Invoke `/nanoresearch:scout "$ARGUMENTS"`.

If scout writes `SCOPING_MEMO.md` instead of `EXPERIMENT_SPEC.md` → update `nanoresearch.json`: `status: "failed"`, `phase: "scout_failed"`. `git add SCOPING_MEMO.md nanoresearch.json && git commit -m "scout: scoping memo (failed)"`. Print what's missing and stop.

If scout produces neither `EXPERIMENT_SPEC.md` nor `SCOPING_MEMO.md` → update `nanoresearch.json`: `status: "failed"`, `phase: "scout_failed"`. `git add nanoresearch.json && git commit -m "scout: no output (failed)"`. Print "Scout produced no output." Stop.

Update `nanoresearch.json`: `phase: "loop"`, `idea_summary: [title]`, `timestamp: <now>`.

**Commit:** `git add LANDSCAPE.md IDEA.md EXPERIMENT_SPEC.md nanoresearch.json && git commit -m "scout: [idea title]"`.

### Phase 2: Experiment Loop

**Precondition:** `EXPERIMENT_SPEC.md` exists. If missing, attempt recovery: `git show HEAD:EXPERIMENT_SPEC.md > EXPERIMENT_SPEC.md`. If recovery fails, set `status: "failed"` and stop.

**Budget selection (precedence):** resubmission budget (60m, if `review_state.cycle > 1` with `## Resubmission Requirements`) > persisted `loop_limit` > `EXPERIMENT_BUDGET_HOURS`.

Invoke `/nanoresearch:loop` with the selected budget.

**Post-loop validation:** Re-read `nanoresearch.json`. If `status == "failed"` (loop set it on preflight/baseline failure), commit the failure and stop — do NOT proceed to write. Validate `results.tsv`: must have header row plus at least one data row with `status == keep`. If missing, has no data rows, or has no row with `status == keep`, set `status: "failed"` and stop.

**Resubmission-aware write_state reset:** If `review_state.cycle > 1` with resubmission requirements, reset `write_state` to defaults.

Update `nanoresearch.json`: `phase: "write"`, `best_metric` (per metric direction from EXPERIMENT_SPEC.md, keep rows only), `iteration_count`, `timestamp: <now>`.

**Commit:** `git add results.tsv autoresearch.md nanoresearch.json && { git diff --cached --quiet || git commit -m "loop: [N] iterations, best=[metric]"; }`.

### Phase 3: Write

**Precondition:** `results.tsv` exists with at least 2 rows (header + baseline). If missing, set `status: "failed"` and stop.

**Write state management:**
- If `write_state` is absent → reset to `{sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}` (fresh entry from loop).
- If `write_state.sub_phase == "complete"` AND `paper/main.tex` exists → **skip write phase** (paper already done, crash between write completion and review transition). Proceed directly to review transition below.
- If `write_state.sub_phase == "complete"` AND `paper/main.tex` does NOT exist → check if `review_state` exists and `review_state.cycle > 1` (resubmission context). If so, reset to `{sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}` and preserve `review_state` (revision mode). If not, reset to fresh (new paper mode).
- If `write_state` exists with `sub_phase != "complete"` → preserve it (crash resume — write.md handles resumption).
- The resubmission path (Phase 4) handles its own write_state reset before re-invoking write.

Invoke `/nanoresearch:write`.

If write produces `RESEARCH_MEMO.md` → update `nanoresearch.json`: `status: "completed"`, `phase: "completed"`, `decision: "memo"`. `git add RESEARCH_MEMO.md nanoresearch.json && git commit -m "write: research memo"`. Print summary and stop. (Memos skip review.)

**Review transition:**
- If `review_state` exists AND `cycle > 1` → resubmission: set `phase: "review"`, `sub_phase: "initial_review"`, `results_row_at_cycle_start: <row count>`. Reset all cycle-local fields to schema defaults. Preserve `cycle` and `score_history` only.
- If `review_state` exists AND `cycle == 1` AND `sub_phase != "initial_review"` → crash-resume mid-review: just set `phase: "review"`, preserve existing state.
- Otherwise → initialize `review_state` from scratch per schema with `cycle: 1`, `sub_phase: "initial_review"`, `results_row_at_cycle_start: <row count>`.

**Commit (atomic — includes review_state):** `git add paper/ nanoresearch.json && git commit -m "write: paper draft, entering review"`. This ensures `phase: "review"` and `review_state` are durable from the same commit.

### Phase 4: Peer Review Cycle

**Precondition:** `paper/main.tex` exists. If missing, set `status: "failed"` and stop.

**Repeat up to MAX_REVIEW_CYCLES:**

1. Invoke `/nanoresearch:review`. The review command runs: 4 independent reviews → rebuttal (with optional experiments) → post-rebuttal rescoring → area chair decision. It returns a decision in `nanoresearch.json.review_state`.

2. **Post-review validation:** Re-read `nanoresearch.json`. If `status == "failed"`, commit and stop. If `review_state.decision` is null or not in `{"accepted", "rejected"}`, set `status: "failed"` and stop — the review phase did not produce a valid decision.

3. Read `nanoresearch.json.review_state.decision`:
   - **"accepted"** → break, proceed to summary.
   - **"rejected"** → check bounds:
     - If `review_state.cycle >= MAX_REVIEW_CYCLES` → break.
     - Otherwise → **revise and resubmit:**
       1. **Validate:** First, if `## Rebuttal Addendum` exists in `EXPERIMENT_SPEC.md`, remove it (stale from the prior cycle's rebuttal). Then verify `EXPERIMENT_SPEC.md` contains `## Resubmission Requirements`. If absent, re-read `AUTO_REVIEW.md` for the AC's requirements; if not found in `AUTO_REVIEW.md`, fall back to `paper/reviews/cycle-{N}/area-chair.md` and extract the `### If REJECT: Resubmission Requirements` section. Patch `EXPERIMENT_SPEC.md` and commit. If requirements are still missing or vague (fewer than 2 actionable items), set `status: "failed"`, commit, and **RETURN — do not execute steps 2 through 7**.
       2. Increment `review_state.cycle`. Reset all cycle-local fields to schema defaults. Set `results_row_at_cycle_start` to current `results.tsv` row count (excluding header).
       3. **Phase transition → loop:** Update `phase: "loop"`, `loop_started_at: null`, `timestamp: <now>`. Reset `loop_started_at` so the resubmission loop gets a fresh budget timer (the original value is stale from Phase 2). `git add nanoresearch.json EXPERIMENT_SPEC.md && git commit -m "revise: resubmission setup cycle [N]"`.
       4. Re-invoke `/nanoresearch:loop ${RESUBMISSION_EXPERIMENT_BUDGET_MINUTES}m`. The loop detects `## Resubmission Requirements` in `EXPERIMENT_SPEC.md` and enters fresh-from-spec mode.
       5. **Post-loop failure gate:** Re-read `nanoresearch.json`. If `status == "failed"` (resubmission loop preflight/baseline/runtime failure), commit and stop — do NOT proceed to write. Validate `results.tsv`: must have at least one row with `status == keep`. If not, set `status: "failed"` and stop.
       6. **Phase transition → write:** Update `best_metric` (per metric direction), `iteration_count`, `phase: "write"`. Reset `write_state` to defaults. Preserve `score_history`. `git add nanoresearch.json results.tsv autoresearch.md && { git diff --cached --quiet || git commit -m "revise: loop complete cycle [N]"; }`.
       7. Re-invoke `/nanoresearch:write` (write detects revision mode from `review_state.cycle > 1`).
       8. **Phase transition → review:** Update `phase: "review"`, `timestamp: <now>`, `review_state.results_row_at_cycle_start: <current row count of results.tsv, excluding header>`. This must be set for ALL cycles, not just cycle 1. `git add paper/ nanoresearch.json && git commit -m "revise: paper revised cycle [N]"`. Continue loop.

Update `nanoresearch.json`: `status: "completed"`, `phase: "completed"`, `decision: "accepted"/"rejected"`, `timestamp: <now>`.

**Commit:** `git add nanoresearch.json && git commit -m "nanoresearch: completed ([decision])"`.

**On any `status: "failed"` exit:** always commit: `git add nanoresearch.json && git commit -m "nanoresearch: failed at [phase]"`.

### Phase 5: Summary

```
=== nanoresearch complete ===
Topic: [topic]
Idea: [idea_summary]
Experiments: [iteration_count] iterations
Best metric: [best_metric]
Output: paper/main.pdf | RESEARCH_MEMO.md
Decision: {if decision == "memo": "memo — research memo (review skipped)" else if score_history exists and is non-empty: "[decision] — [score_history[-1].avg]/10 after [len(score_history)] cycle(s)" else: "[decision] — no scores recorded"}
Duration: [total time]
```

## Rules

- **AUTO_PROCEED = true.** Never block on user input.
- **Update `nanoresearch.json` BEFORE committing** on every phase transition.
- **Commit after each phase.** Scout outputs, loop results, paper draft.
- **On failure**, set `status: "failed"` with the phase name. Do not loop on failures.
- **Large files:** If Write tool fails, retry via Bash heredoc.
- **Budget tracking:** On resume, read `loop_limit` from `nanoresearch.json`. Precedence: resubmission budget > persisted `loop_limit` > default.
- **Git errors:** On stale `.git/index.lock`, remove and retry once. On merge conflict, set `status: "failed"` and stop.
- **Stash restore on exit:** Only pop stash on `status: "completed"` with `phase: "completed"`. If pop fails, log warning. Clear `pre_loop_stash_ref` after successful pop. Do NOT pop on `"failed"` or resumable states.

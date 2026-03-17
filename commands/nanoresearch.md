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

Parse `$ARGUMENTS` by splitting on ` — `. First segment = topic. Subsequent segments are `key: value` pairs or bare flags. Parse from the end: only segments matching a known override key or flag are overrides. Everything before the first valid override is the topic. Reject unknown keys silently. Echo applied config at startup. Persist `budget` and `loop` overrides into `nanoresearch.json` on creation so they survive resume.

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
- If present but invalid JSON → rename to `nanoresearch.json.bak`, warn. Attempt recovery: check for a `nanoresearch/*` branch matching the topic, and reconstruct state from branch artifacts. If recovery fails → **fresh start**.
- If `status` is `"completed"` or `"failed"` → **fresh start**.
- If `phase` is `"completed"` or `"scout_failed"` (regardless of `status`) → **fresh start**. Handles crash between phase and status updates.
- **Validate state schema first:** Verify `nanoresearch.json` contains required top-level fields: `topic`, `status`, `phase`, `branch`. If any are missing, rename to `nanoresearch.json.bak`, warn, and attempt recovery (same as invalid JSON path). This prevents corrupt state (e.g., empty `branch`) from reaching git commands below.
- If `status` is `"in_progress"` AND stale → **fresh start**. Staleness check: run `git log -1 --format=%cI <branch>` where `<branch>` is `nanoresearch.json.branch` (e.g., `git log -1 --format=%cI nanoresearch/mar16`). This queries the branch tip without checking it out. If the most recent commit is > 24 hours old, the run is stale. Do NOT use `nanoresearch.json.timestamp` for this check (it resets on resume, defeating the purpose).
- If `status` is `"in_progress"` AND topic does NOT match `$ARGUMENTS` → **error**: "Another run in progress for '[topic]'. Complete or delete nanoresearch.json first."
- If `status` is `"in_progress"` AND topic matches AND within 24 hours → **resume** from recorded phase.

**On fresh start:**
0. **Stash rescue:** Before replacing `nanoresearch.json`, check the old file (if it exists) for `pre_loop_stash_ref`. If present and the stash still exists (`git stash list | grep <ref>`), do NOT pop it here — popping would contaminate the new research branch. Instead, log warning: "Previous run stashed your working tree changes (ref: {ref}). To restore: `git stash pop` on your original branch after this run." The stash ref is preserved in git's stash list and the user can recover it manually.
1. Resolve the default branch (`git symbolic-ref refs/remotes/origin/HEAD` or fall back to `main`). Then `git checkout -b nanoresearch/<tag> <default-branch>` (tag = date, e.g., `mar16`). If branch exists, append sequence number: `mar16-2`. **Branch first, clean second** — never delete artifacts on the user's current branch. **Verify branch creation:** run `git branch --show-current` and confirm it matches `nanoresearch/<tag>`. If it does not match, print error "Branch creation failed — could not create nanoresearch/<tag>" and abort. Do NOT proceed to cleanup, do NOT write `nanoresearch.json` (it does not exist yet).
2. Clean stale artifacts on the new branch, preserving files required by active skip flags:
   - If `skip-scout`: preserve `EXPERIMENT_SPEC.md`.
   - If `skip-to-write`: preserve `EXPERIMENT_SPEC.md`, `results.tsv`, `autoresearch.md`.
   - Default (no skip): `rm -f LANDSCAPE.md IDEA.md EXPERIMENT_SPEC.md results.tsv autoresearch.md AUTO_REVIEW.md RESEARCH_MEMO.md SCOPING_MEMO.md && rm -rf paper/`.
   - Always clean crash-recovery artifacts: `rm -f .revert-pending *.bak *.episode-bak nanoresearch.json`.
3. Create `nanoresearch.json` per the state schema: `topic`, `status: "in_progress"`, `phase` (set by skip flag or `"scout"`), `branch: "nanoresearch/<tag>"`, `timestamp: <now ISO 8601>`, `venue`, `codex`, `decision: null`, `budget_override` (if set), `loop_override` (if set).
   - If phase is `"scout"`: set `scout_state: {sub_phase: "survey", novelty_confidence: "high"}`.
   - If `skip-scout` (phase `"loop"`): set `scout_state: {sub_phase: "specify", novelty_confidence: "low"}`. No survey was done, so novelty confidence must be low.
   - If `skip-to-write` (phase `"write"`): set `scout_state: {sub_phase: "specify", novelty_confidence: "low"}`, `write_state: {sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}`. Also validate `results.tsv`: verify header, unique ascending `#` column, at least one row with `status == keep`, parseable metrics for non-crash rows. Compute `best_metric` using metric direction from `EXPERIMENT_SPEC.md` (`## Metric` → `direction`): use `min()` of keep-row metrics if lower-is-better, `max()` if higher-is-better. Compute `iteration_count` from the max `#` value. Persist both in `nanoresearch.json`.

**On resume:**
1. State schema was already validated before the staleness check (see above). Read `branch` from `nanoresearch.json`.
2. `git checkout <branch>` (not `-b`). Verify current branch matches `nanoresearch.json.branch` — if not, warn and switch.
3. Update `timestamp` to now.
4. **Halt artifact check (before any cleanup):** If `phase == "scout"` and `SCOPING_MEMO.md` exists → the scout completed with a halt signal but the orchestrator crashed before recording it. Finalize: set `status: "failed"`, `phase: "scout_failed"`, commit, and stop. If `phase == "scout"` and `EXPERIMENT_SPEC.md` exists and `IDEA.md` exists → the scout completed successfully but the orchestrator crashed before transitioning to `"loop"`. Skip scout: update `nanoresearch.json`: `phase: "loop"`, `idea_summary: [title from IDEA.md]`, `timestamp: <now>`. Commit and proceed to Phase 2. If `phase == "write"` and `RESEARCH_MEMO.md` exists → finalize: set `status: "completed"`, `phase: "completed"`, `decision: "memo"`, commit, and stop.
5. If resuming `phase: "scout"` → check `scout_state.sub_phase`. If `"survey"`, clean all: `rm -f LANDSCAPE.md IDEA.md EXPERIMENT_SPEC.md SCOPING_MEMO.md`. If `"ideate"`, preserve `LANDSCAPE.md`, clean: `rm -f IDEA.md EXPERIMENT_SPEC.md SCOPING_MEMO.md`. If `"specify"`, preserve `LANDSCAPE.md`, `IDEA.md`, and `EXPERIMENT_SPEC.md` (if it exists, it is completed work from this sub_phase), clean: `rm -f SCOPING_MEMO.md`.
6. If resuming `phase: "write"` → delete stale `RESEARCH_MEMO.md` if it exists and `write_state` is absent or `write_state.sub_phase != "complete"` (prevents a stale memo from a prior write attempt from triggering the halt check on a subsequent crash-resume).
7. Jump to the recorded phase.

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

**Budget selection:** Check in order:
1. If `review_state` exists AND `review_state.cycle > 1` AND `EXPERIMENT_SPEC.md` contains `## Resubmission Requirements` → this is a resubmission loop. Use `RESUBMISSION_EXPERIMENT_BUDGET_MINUTES` (60m). This handles crash-resume during resubmission correctly.
2. If `budget` override (CLI or persisted) → `/nanoresearch:loop <budget value>` (e.g., `/nanoresearch:loop 8h`)
3. If `loop` override (CLI or persisted) → `/nanoresearch:loop <N>` (e.g., `/nanoresearch:loop 50`)
4. Otherwise → `/nanoresearch:loop ${EXPERIMENT_BUDGET_HOURS}h`

Invoke `/nanoresearch:loop` with the selected budget.

The loop reads `EXPERIMENT_SPEC.md` for setup, runs the autoresearch protocol (edit→commit→run→measure→keep/revert) until the budget is exhausted.

**Post-loop validation:** Re-read `nanoresearch.json`. If `status == "failed"` (loop set it on preflight/baseline failure), commit the failure and stop — do NOT proceed to write. Validate `results.tsv`: must have header row plus at least one data row with `status == keep`. If missing, has no data rows, or has no row with `status == keep`, set `status: "failed"` and stop.

**Resubmission-aware write_state reset:** If `review_state` exists AND `review_state.cycle > 1` AND `EXPERIMENT_SPEC.md` contains `## Resubmission Requirements` → this is a crash-resumed resubmission. Reset `write_state: {sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}` to prevent the stale `write_state.sub_phase == "complete"` from skipping the resubmission rewrite.

Update `nanoresearch.json`: `phase: "write"`, `best_metric: [computed using metric direction from EXPERIMENT_SPEC.md: min() of keep-row metrics if lower-is-better, max() if higher-is-better — never discard/crash rows]`, `iteration_count: [N]`, `timestamp: <now>`.

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

**Review transition (conditional on cycle):** If `review_state` already exists AND `review_state.cycle > 1` → this is a resubmission cycle. Update: `phase: "review"`, `timestamp: <now>`, `review_state.sub_phase: "initial_review"`, `review_state.results_row_at_cycle_start: <current row count of results.tsv, excluding header>`. **Defensively reset cycle-local fields** (guards against partial state from a prior crash): `review_state.decision: null`, `review_state.codex_threads: {}`, `review_state.reviewer_dispatch: {}`, `review_state.participating_reviewers: []`, `review_state.effective_synthesis_mode: null`, `review_state.scores: {initial: {}, post_rebuttal: {}, inherited: []}`, `review_state.rebuttal_revision_progress: null`. Preserve `cycle` and `score_history` only. If `review_state` exists AND `review_state.sub_phase` is NOT `"initial_review"` AND `review_state.cycle == 1` → review has already progressed beyond initialization in cycle 1. Do NOT re-initialize — just set `phase: "review"` and proceed to Phase 4 (prevents destroying in-progress review state on crash-resume through write). Otherwise (absent `review_state`) → initialize from scratch: `phase: "review"`, `timestamp: <now>`, `review_state: {cycle: 1, sub_phase: "initial_review", decision: null, codex_threads: {}, reviewer_dispatch: {}, participating_reviewers: [], effective_synthesis_mode: null, scores: {initial: {}, post_rebuttal: {}, inherited: []}, score_history: [], results_row_at_cycle_start: <current row count of results.tsv, excluding header>, rebuttal_revision_progress: null}`.

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
       2. Increment `review_state.cycle`. Reset cycle-local fields: `review_state.sub_phase: "initial_review"`, `review_state.decision: null`, `review_state.codex_threads: {}`, `review_state.reviewer_dispatch: {}`, `review_state.participating_reviewers: []`, `review_state.effective_synthesis_mode: null`, `review_state.scores: {initial: {}, post_rebuttal: {}, inherited: []}`, `review_state.rebuttal_revision_progress: null`. Set `review_state.results_row_at_cycle_start` to the current row count of `results.tsv` (excluding header) — this captures the pre-resubmission-loop baseline so write.md can detect which rows are truly new to this cycle. (Write uses `cycle > 1` to detect revision mode, not `decision`.)
       3. **Phase transition → loop:** Update `phase: "loop"`, `loop_started_at: null`, `timestamp: <now>`. Reset `loop_started_at` so the resubmission loop gets a fresh budget timer (the original value is stale from Phase 2). `git add nanoresearch.json EXPERIMENT_SPEC.md && git commit -m "revise: resubmission setup cycle [N]"`.
       4. Re-invoke `/nanoresearch:loop ${RESUBMISSION_EXPERIMENT_BUDGET_MINUTES}m`. The loop detects `## Resubmission Requirements` in `EXPERIMENT_SPEC.md` and enters fresh-from-spec mode.
       5. **Post-loop failure gate:** Re-read `nanoresearch.json`. If `status == "failed"` (resubmission loop preflight/baseline/runtime failure), commit and stop — do NOT proceed to write. Validate `results.tsv`: must have at least one row with `status == keep`. If not, set `status: "failed"` and stop.
       6. **Phase transition → write:** Parse `results.tsv` to update `best_metric` (using metric direction from `EXPERIMENT_SPEC.md`: `min()` of keep-row metrics if lower-is-better, `max()` if higher-is-better) and `iteration_count`. Update `phase: "write"`, `timestamp: <now>`. Reset `write_state: {sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}`. Preserve `score_history` (do NOT reset — it accumulates across cycles). `git add nanoresearch.json results.tsv autoresearch.md && { git diff --cached --quiet || git commit -m "revise: loop complete cycle [N]"; }`.
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
- **Large files:** If Write tool fails, retry via Bash heredoc silently.
- **Budget tracking:** The loop tracks its own wall-clock start time independently of `nanoresearch.json.timestamp`. On resume, explicitly read `budget_override` and `loop_override` from `nanoresearch.json` — these persisted fields are the authoritative source for budget/iteration limits. Precedence: resubmission budget > CLI override > persisted override > default.
- **Git errors:** If any git command fails, check for stale `.git/index.lock` (remove if no git process owns it — verify with `lsof .git/index.lock 2>/dev/null`, retry once). If merge conflict or unmerged paths, set `status: "failed"` and stop.
- **Stash restore on exit:** Only restore stash on **durable terminal** exits: `status: "completed"` with `phase: "completed"`. Do NOT pop stash on `status: "failed"`, manual stop, or any resumable state — the stash must remain across write and review phases. For failed runs the user doesn't intend to resume, they can manually `git stash pop`. The fresh-start stash rescue (step 0 above) handles recovery when a new run begins. Check `nanoresearch.json.pre_loop_stash_ref`. If present and stash still exists (`git stash list | grep <ref>`), attempt `git stash pop`. After successful pop, clear `pre_loop_stash_ref: null` and commit. If pop fails (conflict), log warning: "Could not restore stashed changes. Run `git stash pop` manually."

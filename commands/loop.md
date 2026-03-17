---
description: "Experiment loop: edit, commit, run, measure, keep or revert. Faithful to Karpathy's autoresearch. Use for 'nanoresearch:loop', 'experiment loop', 'autoresearch'."
argument-hint: [iterations-or-duration-or-resume]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---

# Loop: The Autoresearch Kernel

You are a completely autonomous researcher. Try things out. If they work, keep. If they don't, discard. Never stop.

Read TIMEOUT_MINUTES and EXPERIMENT_BUDGET_HOURS from the project's **CLAUDE.md**.

## Setup

### From EXPERIMENT_SPEC.md (pipeline or resubmission)

If `EXPERIMENT_SPEC.md` exists, read it for: metric, sanity metric, command, extraction, files in scope, constraints, guard.

### From autoresearch.md (resume)

If `autoresearch.md` and `results.tsv` exist on a `nanoresearch/*` branch:

**Crash recovery (ALWAYS runs — including rebuttal/resubmission):**
1. Read `nanoresearch.json.branch` and `git checkout <branch>` (use the stored authoritative branch name, not a synthesized `nanoresearch/<tag>`).
2. Read `autoresearch.md` for context.
3. **Check for pending revert:** If `.revert-pending` exists, a previous discard was interrupted. Read the target from `.revert-pending`. Restore backup files **idempotently** (check existence before each mv): `[ -f results.tsv.bak ] && mv results.tsv.bak results.tsv`, same for `autoresearch.md.bak`, `nanoresearch.json.bak`, `EXPERIMENT_SPEC.md.bak`. If a backup file is missing (already restored in a prior partial recovery), skip it. Remove `.revert-pending`. Clean up any remaining `.bak` files: `rm -f *.bak`. Log: "Recovered from interrupted revert."
4. **Check for interrupted exploration episode:** If `autoresearch.md` contains `## Active Exploration`, read the `start_commit` from that section. Reset to it: `git reset --hard <start_commit>`. Restore `.episode-bak` files if they exist (`mv results.tsv.episode-bak results.tsv` etc.). Remove the `## Active Exploration` section from `autoresearch.md`. **Commit the recovery immediately** to make it durable: `git add results.tsv autoresearch.md nanoresearch.json EXPERIMENT_SPEC.md && git commit -m "loop: abandoned exploration episode"`. Log: "Abandoned interrupted exploration episode." **Skip step 5 after this** (HEAD is known-safe at the exploration start commit).
5. **Verify HEAD is safe:** If `results.tsv` has no data rows or no baseline row #0, skip ancestry checking — recreate a clean header if needed and proceed to baseline (step 7). Otherwise: find the last `keep` row in `results.tsv` (if no `keep` row exists, use the baseline row #0). Check if HEAD **is exactly** the keep commit or a known-safe descendant: `git merge-base --is-ancestor <keep-commit> HEAD`. If YES → check whether HEAD has unevaluated commits on top: if `HEAD != <keep-commit>`, verify that every commit between <keep-commit> and HEAD is either a checkpoint commit (message starts with "loop: checkpoint" or "loop: keep" or "loop: budget timer") or a phase-boundary commit. If any commit looks like an unevaluated experiment (message matches a hypothesis description), it was committed before running but never evaluated — `git reset --hard <keep-commit>`. If NO → crash left HEAD on an unevaluated or divergent commit; `git reset --hard <keep-commit>`. After any reset, verify `results.tsv` and `autoresearch.md` still exist — if destroyed, restore from git: `git show <keep-commit>:results.tsv > results.tsv`.

**Resume context:**
6. Read only the last `keep` row in `results.tsv` for the current best.
7. Read `git log --oneline -20`
8. Read all in-scope files
9. If `EXPERIMENT_SPEC.md` contains `## Rebuttal Addendum` or `## Resubmission Requirements` → enter fresh-from-spec mode (re-read spec for new objectives, skip baseline if results.tsv has one, but DO NOT skip crash recovery above).
10. Continue the loop. Do not re-run baseline.

### Interactive (standalone)

If neither exists, establish with the user: metric (with direction), command, metric extraction, sanity metric, comparison protocol, files in scope, constraints, guard. Auto-detect from the repo. Confirm in one round.

## Initialization

1. Create branch `nanoresearch/<tag>` from main if not already on a `nanoresearch/*` branch.
2. **Clean tree check**: If `git status --porcelain` is non-empty AND this is NOT a rebuttal/resubmission invocation (i.e., `EXPERIMENT_SPEC.md` does NOT contain `## Rebuttal Addendum` or `## Resubmission Requirements`), `git stash push -m 'nanoresearch: pre-loop'`. Record the stash ref **immediately** — if `nanoresearch.json` exists, persist as `pre_loop_stash_ref` in `nanoresearch.json` and commit (`git add nanoresearch.json && git commit -m "loop: stashed user changes"`). Also record in `autoresearch.md` under `## Stashed Changes` once it is created (step 8). Do NOT pop stash at loop exit — the stash must remain across write and review phases to prevent user changes from leaking into experiment/paper/review commits. The orchestrator (`nanoresearch.md`) is responsible for popping on pipeline completion. On resume within the loop, check `nanoresearch.json.pre_loop_stash_ref` — if present, verify stash still exists (`git stash list | grep <ref>`). Do NOT pop during loop resume — just acknowledge the stash exists. Skip stash for rebuttal/resubmission — the caller manages the working tree.
3. Read every in-scope file AND relevant read-only files.
4. **Preflight**: Verify: (a) data files in EXPERIMENT_SPEC.md exist and are non-empty, (b) command is executable (`command -v`), (c) required packages importable, (d) `timeout` command exists — if not, check for `gtimeout` (macOS with coreutils) and alias it. If any fail → fix or set `status: "failed"` and stop.
5. Add `run.log`, `.revert-pending`, `*.bak`, `*.episode-bak` to `.gitignore` if not present. Commit.
6. If `results.tsv` already exists and contains a baseline row (experiment #0), do NOT overwrite — append after existing rows and skip baseline (step 7). Otherwise create with header.
7. Run **baseline**: execute the command as-is. Record as experiment #0 with `status: keep` (the baseline is the canonical initial keep row — all `best_metric` and recovery logic depends on this). If baseline crashes: diagnose from `run.log`, fix environment (not in-scope code), retry up to 3 times. If still failing → set `status: "failed"` in `nanoresearch.json` and stop.
8. Write `autoresearch.md` (recovery doc). Commit baseline and recovery doc together: `git add results.tsv autoresearch.md nanoresearch.json && git commit -m "loop: baseline"`. This ensures the #0 keep row is durable before any experiments run.
9. Start the loop.

### autoresearch.md

```
# Autoresearch: <goal>
## Objective  ## Metric  ## Sanity Metric  ## Command
## Metric Extraction  ## Comparison Protocol
## Files in Scope  ## Off Limits  ## Constraints  ## Guard
## What's Been Tried — (update every ~5 experiments)
```

## The Loop

**LOOP** until budget expires, iteration cap reached, or manually stopped.

**Budget detection:** Record `loop_started_at = $(date +%s)` on first entry. If `nanoresearch.json` exists, persist `loop_started_at` in it (if already present, reuse the stored value — do not reset on resume). **Immediately commit** after setting `loop_started_at` for the first time: `git add nanoresearch.json && git commit -m "loop: budget timer start"`. This ensures the timer survives any crash before the first checkpoint. Check `$(date +%s)` against `loop_started_at` before each iteration. When budget expires, finish current experiment and stop. Reject zero or negative budgets up front.

**Argument-based bounds (highest precedence):** If invoked with a number N, run exactly N iterations. If invoked with a duration (e.g., `30m`, `2h`), use that as wall-clock budget. **Explicit argument overrides `EXPERIMENT_BUDGET_HOURS` from CLAUDE.md.** If no argument, fall back to `EXPERIMENT_BUDGET_HOURS`.

**Timeout clamping:** Each run uses `timeout min(TIMEOUT_MINUTES, remaining_budget_minutes)m <command>`. Do not start a new iteration if remaining budget is zero.

Each iteration:

1. **Check git state.** Working tree should be clean.
2. **Plan.** One hypothesis, one line.
3. **Edit.** In-scope files only.
4. **Commit.** Stash `autoresearch.md` if dirty: `git stash push autoresearch.md`. Then `git add <files> && git commit -m "<description>"` — commit BEFORE running. Pop stash after: `git stash pop` (if stashed).
5. **Run.** `timeout ${clamped_timeout}m <command> > run.log 2>&1` (where `clamped_timeout = min(TIMEOUT_MINUTES, remaining_budget_minutes)`)
6. **Extract and validate.** Run extraction commands for primary and sanity metrics. No output → crash → `tail -n 50 run.log`. **Numeric validation gate:** Each extractor must yield exactly one finite numeric scalar. If extraction returns multiple numbers, non-numeric text, `nan`, `inf`, or an empty value, treat the run as `crash` with reason "metric extraction failed: [raw output]". Do not record non-numeric metrics in `results.tsv` or use them for keep/discard decisions.
7. **Record.** Append to `results.tsv`. Do NOT commit it.
8. **Guard.** Run guard only on improvement. Fail → rework (max 2 attempts) → discard.
9. **Sanity check.** Degraded beyond bound → discard even if primary improved.
10. **Decide.**
    - Improved (guard + sanity pass) → `keep`.
    - Equal or worse → `discard`.
    - Equal + simpler code → `keep` (simplicity criterion).
    - Crash → triage: trivial bug → fix+retry (max 2); environment → remediate; idea-breaking → skip.
    **On keep:** Immediately commit `results.tsv` to make the keep durable: `git add results.tsv && git commit -m "loop: keep iteration N, metric=X"`. This prevents loss of experiment data on crash before the next checkpoint.
    **On discard/revert:** Crash-safe revert protocol:
    1. Back up all state files: `cp results.tsv results.tsv.bak && cp autoresearch.md autoresearch.md.bak && cp nanoresearch.json nanoresearch.json.bak && cp EXPERIMENT_SPEC.md EXPERIMENT_SPEC.md.bak`.
    2. Write sentinel: `echo "HEAD~1" > .revert-pending`.
    3. Reset: `git reset --hard HEAD~1`.
    4. Restore: `mv results.tsv.bak results.tsv && mv autoresearch.md.bak autoresearch.md && mv nanoresearch.json.bak nanoresearch.json && mv EXPERIMENT_SPEC.md.bak EXPERIMENT_SPEC.md`.
    5. Remove sentinel: `rm -f .revert-pending`.
    If the process crashes between steps 2-4, the resume logic detects `.revert-pending` and completes the restore.
11. **Report.** Every 5 iterations: `=== Iteration N: metric at X.XX, K/D/C ===`
12. **Update `autoresearch.md`** every ~5 experiments.
13. **Checkpoint.** Every 10 iterations: `git add results.tsv autoresearch.md nanoresearch.json && git commit -m "loop: checkpoint at iteration N"`. Include `nanoresearch.json` to persist `loop_started_at` and any state updates durably.
14. **Repeat.**

### Simplicity Criterion

- Small improvement + ugly complexity → probably discard.
- Removing code + equal/better → definitely keep.
- Equal metric + simpler code → keep.

### Multi-Step Exploration (enabling refactors)

Some improvements require 2-3 coordinated edits that look bad individually. When you have a hypothesis that needs multiple steps:

1. **Start an exploration episode.** Record `exploration_start_commit = HEAD` and `exploration_budget = 3` (max commits in the episode). **Persist to `autoresearch.md`** under `## Active Exploration`: start commit hash, budget, and hypothesis. Back up state files before the first step: `cp results.tsv results.tsv.episode-bak && cp autoresearch.md autoresearch.md.episode-bak && cp nanoresearch.json nanoresearch.json.episode-bak && cp EXPERIMENT_SPEC.md EXPERIMENT_SPEC.md.episode-bak`.
2. **Commit each step** normally (steps 1-4 of the loop), but **skip steps 5-10** (Run through Decide) for intermediate commits. Do NOT log intermediate steps to `results.tsv` — keep episode-internal notes in `autoresearch.md` only.
3. **Evaluate at the episode end** (after all steps are committed, or after the exploration budget is exhausted). Run the command once and extract metrics. Record a single row in `results.tsv` with the episode's final metric and a description summarizing all steps.
4. **Decide on the whole episode:**
   - Improved → `keep` (all commits stay). Remove `## Active Exploration` from `autoresearch.md`. Clean up: `rm -f *.episode-bak`.
   - Not improved → `discard` the entire episode: first `git reset --hard <exploration_start_commit>` (restore code to episode start), THEN restore state files from episode backups (`mv results.tsv.episode-bak results.tsv && mv autoresearch.md.episode-bak autoresearch.md && mv nanoresearch.json.episode-bak nanoresearch.json && mv EXPERIMENT_SPEC.md.episode-bak EXPERIMENT_SPEC.md`). The order matters: reset first, then restore — otherwise the reset overwrites the restored files. Clean up: `rm -f *.episode-bak`.
5. **Limit:** Max 1 exploration episode per 5 total iterations (counting episode commits toward the 5). Do not start an episode if it would span a checkpoint boundary (iteration divisible by 10). Defer to after the checkpoint.

**Crash recovery:** On resume, if `autoresearch.md` contains `## Active Exploration`, an episode was interrupted. Read the `start_commit` from that section. Reset to it: `git reset --hard <start_commit>`, restore `.episode-bak` files if they exist, remove `## Active Exploration`. The episode is abandoned — it can be retried from scratch.

Use exploration episodes when: the next improvement clearly requires setup (refactoring, adding instrumentation, restructuring data flow) before the payoff.

### Strategy Checkpoint (every 10 iterations)

At every 10th iteration (aligned with the git checkpoint), perform a structured strategy review. **Do not run during an active exploration episode** — defer to after the episode concludes. If the stuck recovery protocol was triggered within the last 5 iterations, skip the direction-change analysis and just record the trajectory.

1. **Classify trajectory:** Summarize the last 10 evaluated experiments (exclude any abandoned exploration episodes) in a table: approach, result, likely cause.
2. **Assess momentum:** `making_progress` (2+ keeps in last 10) | `plateau` (1 keep) | `stuck` (0 keeps).
3. **If plateau or stuck:**
   a. Identify the common thread across recent failures.
   b. Generate 3 alternative directions with explicit predictions for each.
   c. Commit to one direction for the next 5 iterations.
4. Record the strategy assessment in `autoresearch.md` under `## Strategy Checkpoints`.

### When Stuck (3+ consecutive reverts)

Structured recovery protocol — do NOT just "think harder":

1. **Failure analysis:** Produce a table of the last 5+ failed approaches: what was tried, what happened, likely root cause.
2. **Pattern identification:** What do the failures have in common? Is the bottleneck in the model, data, evaluation, or approach?
3. **Direction change:** Generate 3 fundamentally different directions (not variations of the same idea). For each, state the predicted outcome and why it avoids the identified pattern.
4. **Commit to one** and run it. If it also fails, try the next. If all 3 fail, consider an exploration episode for a multi-step approach.
5. **Last resort:** Re-read source files. Read local papers in `papers/` or `literature/` if present. Rewind to an earlier successful commit and try a completely different approach.

**NEVER give up. But think differently, not just harder.**

## Logging

`results.tsv` — TAB-separated:

```
#	timestamp	commit	metric	sanity	status	description
```

- `#` — sequential experiment number (0 = baseline). On resume, continue from `last # + 1`.
- `timestamp` — ISO 8601 with seconds (e.g., `2026-03-16T14:30:05`).
- `status` values: `keep` | `discard` | `crash`. Exploration episodes log one summary row at episode end (keep or discard), not per intermediate step.
- `metric`/`sanity` — `NA` for crashes. Do NOT commit `run.log`. `results.tsv` is committed at phase boundaries and every 10 iterations (checkpoint).

## Constraints

No editing outside scope. No new dependencies. No modifying evaluation harness. No stopping to ask permission.

## NEVER STOP.

Loop until budget expires or manually stopped. Do NOT ask "should I continue?"

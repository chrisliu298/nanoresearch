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
5. **Check for interrupted exploration episode:** If `autoresearch.md` contains `## Active Exploration`, read the `start_commit` from that section. Reset to it: `git reset --hard <start_commit>`. Restore `.episode-bak` files if they exist (`mv results.tsv.episode-bak results.tsv` etc.). Remove the `## Active Exploration` section from `autoresearch.md`. **Commit the recovery immediately** to make it durable: `git add results.tsv autoresearch.md nanoresearch.json EXPERIMENT_SPEC.md && git commit -m "loop: abandoned exploration episode"`. Log: "Abandoned interrupted exploration episode." **Skip step 6 after this** (HEAD is known-safe at the exploration start commit).
6. **Verify HEAD is safe:** If no baseline row exists, skip to baseline (step 7). Otherwise find the last `keep` commit from `results.tsv`. If HEAD has unevaluated experiment commits on top of the keep commit, `git reset --hard <keep-commit>`. Infrastructure commits (prefixed `loop:`, `scout:`, `write:`, `review:`, `revise:`, `nanoresearch:`) are safe — only reset for uncommitted experiment hypotheses. After any reset, restore `results.tsv` and `autoresearch.md` from the keep commit if missing.

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
2. **Clean tree check**: If `git status --porcelain` is non-empty AND not a rebuttal/resubmission invocation, `git stash push -m 'nanoresearch: pre-loop'`. Persist `pre_loop_stash_ref` in `nanoresearch.json` and commit. Do NOT pop stash at loop exit — the orchestrator handles restore on pipeline completion.
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

**Budget detection:** Record `loop_started_at = $(date +%s)` on first entry. Persist in `nanoresearch.json` (reuse stored value on resume). Commit after first set. Check remaining budget before each iteration. When expired, finish current experiment and stop.

**Argument-based bounds (highest precedence):** If invoked with a number N, run exactly N iterations. If invoked with a duration (e.g., `30m`, `2h`), use that as wall-clock budget. **Explicit argument overrides `EXPERIMENT_BUDGET_HOURS` from CLAUDE.md.** If no argument, fall back to `EXPERIMENT_BUDGET_HOURS`.

**Timeout clamping:** Each run uses `timeout min(TIMEOUT_MINUTES, remaining_budget_minutes)m <command>`. Do not start a new iteration if remaining budget is zero.

Each iteration:

1. **Check git state.** Working tree should be clean.
2. **Plan.** One hypothesis, one line.
3. **Edit.** In-scope files only.
4. **Commit.** Stash `autoresearch.md` if dirty: `git stash push -m 'loop: autoresearch temp' autoresearch.md`. Then `git add <files> && git commit -m "<description>"` — commit BEFORE running. Pop stash after: `git stash pop` (if stashed).
5. **Budget check before run:** Compute `remaining = loop_started_at + budget_seconds - $(date +%s)`. If `remaining <= 0`, skip the run and exit the loop — do not record a crash row (this is a budget expiration, not a crash). Revert the committed-but-not-run experiment using the discard protocol. **Run.** `timeout ${clamped_timeout}m <command> > run.log 2>&1` (where `clamped_timeout = min(TIMEOUT_MINUTES, remaining_budget_minutes)`)
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

### Strategy Recovery (every 10 iterations OR 3+ consecutive reverts)

**Trigger:** At every 10th iteration (aligned with checkpoint), or immediately after 3+ consecutive reverts. Do not run during an active exploration episode.

1. **Classify trajectory:** Table of recent experiments: approach, result, likely cause. Assess momentum: `making_progress` (2+ keeps in last 10) | `plateau` (1 keep) | `stuck` (0 keeps).
2. **If plateau or stuck:** Identify the common failure pattern. Generate 3 fundamentally different directions (not variations) with predicted outcomes. Commit to one.
3. **Escalation (all 3 fail):** Re-read source files. Check `papers/` or `literature/`. Consider an exploration episode. Rewind to an earlier successful commit and try a completely different approach.
4. Record in `autoresearch.md` under `## Strategy Checkpoints`.

**Think differently, not harder.**

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

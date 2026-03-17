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

If `autoresearch.md` and `results.tsv` exist on a `nanoresearch/*` branch, AND `EXPERIMENT_SPEC.md` does NOT contain `## Rebuttal Addendum` or `## Resubmission Requirements`:
1. `git checkout nanoresearch/<tag>`
2. Read `autoresearch.md` for context
3. Read only the last `keep` row in `results.tsv` for the current best
4. **Verify HEAD is safe:** find the last `keep` row in `results.tsv` (if no `keep` row exists, use the baseline row #0). Check if that commit is an ancestor of HEAD: `git merge-base --is-ancestor <keep-commit> HEAD`. If YES → HEAD is safe (may be a checkpoint commit on top of the keep). If NO → crash left HEAD on an unevaluated or divergent commit; `git reset --hard <keep-commit>`.
5. Read `git log --oneline -20`
6. Read all in-scope files
7. Continue the loop. Do not re-run baseline.

### Interactive (standalone)

If neither exists, establish with the user: metric (with direction), command, metric extraction, sanity metric, comparison protocol, files in scope, constraints, guard. Auto-detect from the repo. Confirm in one round.

## Initialization

1. Create branch `nanoresearch/<tag>` from main if not already on a `nanoresearch/*` branch.
2. **Clean tree check**: If `git status --porcelain` is non-empty AND this is NOT a rebuttal/resubmission invocation (i.e., `EXPERIMENT_SPEC.md` does NOT contain `## Rebuttal Addendum` or `## Resubmission Requirements`), `git stash push -m 'nanoresearch: pre-loop'`. Pop stash after loop completes. Skip stash for rebuttal/resubmission — the caller manages the working tree.
3. Read every in-scope file AND relevant read-only files.
4. **Preflight**: Verify: (a) data files in EXPERIMENT_SPEC.md exist and are non-empty, (b) command is executable (`command -v`), (c) required packages importable, (d) `timeout` command exists — if not, check for `gtimeout` (macOS with coreutils) and alias it. If any fail → fix or set `status: "failed"` and stop.
5. Add `run.log` to `.gitignore` if not present. Commit.
6. If `results.tsv` already exists and contains a baseline row (experiment #0), do NOT overwrite — append after existing rows and skip baseline (step 7). Otherwise create with header.
7. Run **baseline**: execute the command as-is. Record as experiment #0. If baseline crashes: diagnose from `run.log`, fix environment (not in-scope code), retry up to 3 times. If still failing → set `status: "failed"` in `nanoresearch.json` and stop.
8. Write `autoresearch.md` (recovery doc). Commit it.
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

**Budget detection:** If `nanoresearch.json` exists with `phase: "loop"`, read `EXPERIMENT_BUDGET_HOURS` from CLAUDE.md. Check `$(date +%s)` against start time before each iteration. When budget expires, finish current experiment and stop. Reject zero or negative budgets up front.

**Argument-based bounds:** If invoked with a number N, run exactly N iterations. If invoked with a duration (e.g., `30m`, `2h`), use that as wall-clock budget.

**Timeout clamping:** Each run uses `timeout min(TIMEOUT_MINUTES, remaining_budget_minutes)m <command>`. Do not start a new iteration if remaining budget is zero.

Each iteration:

1. **Check git state.** Working tree should be clean.
2. **Plan.** One hypothesis, one line.
3. **Edit.** In-scope files only.
4. **Commit.** Stash `autoresearch.md` if dirty: `git stash push autoresearch.md`. Then `git add <files> && git commit -m "<description>"` — commit BEFORE running. Pop stash after: `git stash pop` (if stashed).
5. **Run.** `timeout ${clamped_timeout}m <command> > run.log 2>&1` (where `clamped_timeout = min(TIMEOUT_MINUTES, remaining_budget_minutes)`)
6. **Extract.** Run extraction commands for primary and sanity metrics. No output → crash → `tail -n 50 run.log`.
7. **Record.** Append to `results.tsv`. Do NOT commit it.
8. **Guard.** Run guard only on improvement. Fail → rework (max 2 attempts) → discard.
9. **Sanity check.** Degraded beyond bound → discard even if primary improved.
10. **Decide.**
    - Improved (guard + sanity pass) → `keep`.
    - Equal or worse → `discard`.
    - Equal + simpler code → `keep` (simplicity criterion).
    - Crash → triage: trivial bug → fix+retry (max 2); environment → remediate; idea-breaking → skip.
    **On discard/revert:** Back up uncommitted state first: `cp results.tsv results.tsv.bak && cp autoresearch.md autoresearch.md.bak && cp nanoresearch.json nanoresearch.json.bak`. Then `git reset --hard HEAD~1`. Then restore: `mv results.tsv.bak results.tsv && mv autoresearch.md.bak autoresearch.md && mv nanoresearch.json.bak nanoresearch.json`.
11. **Report.** Every 5 iterations: `=== Iteration N: metric at X.XX, K/D/C ===`
12. **Update `autoresearch.md`** every ~5 experiments.
13. **Checkpoint.** Every 10 iterations: `git add results.tsv autoresearch.md && git commit -m "loop: checkpoint at iteration N"`. This makes progress durable for preemptible machines.
14. **Repeat.**

### Simplicity Criterion

- Small improvement + ugly complexity → probably discard.
- Removing code + equal/better → definitely keep.
- Equal metric + simpler code → keep.

### When Stuck (3+ consecutive reverts)

Re-read source files. Read local papers in `papers/` or `literature/` if present. Combine near-misses. Try radical changes. Reason about bottlenecks. Last resort: rewind to earlier successful commit. **NEVER give up. Think harder.**

## Logging

`results.tsv` — TAB-separated:

```
#	timestamp	commit	metric	sanity	status	description
```

- `#` — sequential experiment number (0 = baseline). On resume, continue from `last # + 1`.
- `timestamp` — ISO 8601 with seconds (e.g., `2026-03-16T14:30:05`).
- `NA` for crashes. Do NOT commit `run.log`. `results.tsv` is committed at phase boundaries and every 10 iterations (checkpoint).

## Constraints

No editing outside scope. No new dependencies. No modifying evaluation harness. No stopping to ask permission.

## NEVER STOP.

Loop until budget expires or manually stopped. Do NOT ask "should I continue?"

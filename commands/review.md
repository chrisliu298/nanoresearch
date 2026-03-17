---
description: "Review phase: simulated peer review with 4 reviewers, rebuttal, area chair decision. Use for 'nanoresearch:review'."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply
---

# Review: Simulated Peer Review

Read from **CLAUDE.md**: NUM_REVIEWERS, ACCEPTANCE_THRESHOLD, STRONG_REJECT_VETO, REBUTTAL_EXPERIMENT_BUDGET_MINUTES, REVIEWER_MODEL.

Before composing Codex MCP prompts for R3/R4, read **docs/prompting-codex.md** for GPT-5.4-specific patterns.

## Overview

One submission cycle: 4 reviews → rebuttal → rescoring → area chair decision. The orchestrator (`nanoresearch.md`) handles the reject→revise outer loop; this command runs one cycle and returns a decision.

## Resumption

Check `nanoresearch.json.review_state`:
- Validate `review_state` has required fields (`cycle`, `sub_phase`, `scores`, `score_history`, `reviewer_dispatch`). If any field is missing, re-initialize it to its default value (don't restart the whole phase).
- If `sub_phase` exists and `cycle` matches current cycle → resume from recorded sub-phase, reuse saved `codex_threads` and `reviewer_dispatch`.
  - `initial_review` → start Phase 1. Check `AUTO_REVIEW.md` for already-completed reviews in this cycle — skip re-dispatching reviewers whose reviews are already saved.
  - `rebuttal_triage` → start Step 2a.
  - `rebuttal_experiments` → start Step 2b.
  - `rebuttal_revision` → start Step 2c.
  - `rebuttal_response` → start Step 2d.
  - `rescoring` → start Phase 3.
  - `area_chair` → start Phase 4.
  - `decision_gate` → start Phase 5.
- Otherwise → start from Phase 1.

## Reviewer Panel

| Reviewer | Model | Dispatch | Persona |
|----------|-------|----------|---------|
| R1 | Claude | Agent (subagent) | **Methodologist** — soundness, proofs, correctness |
| R2 | Claude | Agent (subagent) | **Empiricist** — baselines, ablations, significance |
| R3 | GPT-5.4 xhigh | Codex MCP | **Novelty Critic** — originality, related work |
| R4 | GPT-5.4 xhigh | Codex MCP | **Skeptic** — overclaiming, limitations |

Claude reviewers can read code + `results.tsv`. GPT-5.4 reviewers see paper text only.

**Codex fallback:** If Codex MCP unavailable OR `nanoresearch.json.codex == "off"`, run all 4 as Claude subagents.

## Phase 1: Initial Review

Concatenate all `.tex` section files into paper text.

Launch R1, R2 via Agent tool (`run_in_background: true`). Launch R3, R4 via `mcp__codex__codex` (REVIEWER_MODEL, `config: {"model_reasoning_effort": "xhigh"}` — xhigh is the only permitted effort level, no exceptions).

Each reviewer prompt uses the calibration and format defined in `agents/reviewer.md`.

**After each successful reviewer dispatch**, immediately persist its result:
- Save the review text to `AUTO_REVIEW.md` (append).
- Update `review_state.scores.initial.R<N>` with the extracted score.
- Update `review_state.reviewer_dispatch.R<N>` to `"claude"`, `"codex"`, or `"claude-fallback"`.
- If Codex: save `review_state.codex_threads.R<N>` with the threadId.
- Commit both `AUTO_REVIEW.md` and `nanoresearch.json` after each reviewer completes: `git add AUTO_REVIEW.md nanoresearch.json && git commit -m "review: R<N> complete"`.

**Score extraction:** Each review must contain `### Overall Score` followed by a single integer 1-10 on its own line. If the score is not parseable, retry the reviewer once with an appended instruction: "Your Overall Score line must contain exactly one integer 1-10, nothing else." If still unparseable after retry, fall back to Claude subagent.

Retry failures once. If Codex reviewers fail after retry, re-dispatch as Claude subagents with the same persona and record `reviewer_dispatch.R<N>: "claude-fallback"`.

**Minimum panel (coverage-based):** At least 3 reviewers must succeed AND at least one must cover methodology/empirics (R1 or R2) and at least one must cover novelty/skepticism (R3 or R4). Hard fail below 3.

Save final state to `AUTO_REVIEW.md`.

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_triage"`. Commit: `git add AUTO_REVIEW.md nanoresearch.json && git commit -m "review: initial scores"`.

## Phase 2: Rebuttal

Four sub-steps, each checkpointed. On resume, skip completed sub-steps based on `review_state.sub_phase`.

### Step 2a: Triage (`sub_phase: "rebuttal_triage"`)

**Triage** all weaknesses: `experiment_gap` | `editorial_fix` | `clarification` | `overclaim` | `out_of_scope` | `reviewer_error`.

Honesty check: if >50% classified as `reviewer_error`, re-examine — the paper is likely unclear.

Produce a **rebuttal plan** mapping each weakness to its category and planned response. Save to `paper/reviews/rebuttal-plan.md`.

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_experiments"`.

Commit: `git add paper/reviews/rebuttal-plan.md nanoresearch.json && git commit -m "review: rebuttal triage"`.

### Step 2b: Rebuttal Experiments (`sub_phase: "rebuttal_experiments"`)

Skip if no `experiment_gap` items in the rebuttal plan.

1. Replace (not append) the `## Rebuttal Addendum` section in `EXPERIMENT_SPEC.md` with targeted experiments. If no such section exists, append it. This ensures idempotency on crash-retry.
2. **Commit the spec before invoking the loop** — the loop's discard path uses `git reset --hard` which would destroy uncommitted changes: `git add EXPERIMENT_SPEC.md && git commit -m "review: rebuttal spec"`.
3. Invoke `/nanoresearch:loop ${REBUTTAL_EXPERIMENT_BUDGET_MINUTES}m`. The loop detects the `## Rebuttal Addendum` in `EXPERIMENT_SPEC.md` and enters fresh-from-spec mode automatically (skips resume, skips baseline if results.tsv has one).

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_revision"`.

Commit: `git add results.tsv autoresearch.md nanoresearch.json && git diff --cached --quiet || git commit -m "review: rebuttal experiments"`.

### Step 2c: Structured Paper Revision (`sub_phase: "rebuttal_revision"`)

Do NOT edit the paper in a single monolithic pass. Use the write phase's section-by-section machinery:

1. **Identify impacted sections** from the rebuttal plan + new results.tsv rows. Always revisit: experiments, introduction, conclusion, abstract. Revisit method only if protocol changed. Revisit related_work only if positioning/novelty claim changed. Apply dependency closure: if experiments changes → force revisit introduction, conclusion, abstract; if method changes → force revisit experiments + all downstream (see write.md revision mode for full rules).

2. **Edit each impacted section in-place.** For each section:
   - Read the existing `.tex` file + relevant reviewer weaknesses from `AUTO_REVIEW.md` + new results.
   - Edit to address weaknesses. Preserve parts reviewers did not criticize. Soften overclaims. Add new results where applicable.
   - Submit to GPT-5.4 via `mcp__codex__codex` at `xhigh` for per-section review (same protocol as write.md Step 3). Prompt includes: section text, relevant reviewer weaknesses, new results, instruction to check claim-evidence alignment.
   - **Codex fallback:** If Codex MCP unavailable OR `codex: off`, spawn a Claude `write-critic` subagent in section mode.
   - Fix blocking issues. One review round per section.

3. **Compile**: `cd paper && ${COMPILER} -pdf -interaction=nonstopmode main.tex`. Retry up to MAX_COMPILE_ATTEMPTS. Compile failure is non-fatal — rescoring reads .tex source.

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_response"`.

Commit: `git add paper/ nanoresearch.json && git commit -m "review: rebuttal revision"`.

### Step 2d: Author Response (`sub_phase: "rebuttal_response"`)

Compose per-reviewer author response using the rebuttal plan and actual changes made:

```
> W1: [quoted weakness]
Response: [evidence / fix / clarification, citing specific section edits and new results]
```

Save to `paper/reviews/rebuttal-response.md`.

Update `nanoresearch.json`: `review_state.sub_phase: "rescoring"`.

Commit: `git add paper/reviews/rebuttal-response.md nanoresearch.json && git commit -m "review: author response"`.

## Phase 3: Post-Rebuttal Rescoring

Concatenate the **current** (revised) `.tex` section files into updated paper text. Each reviewer must see the revised paper, not the original.

R1, R2 (Claude): spawn reviewer agent with rebuttal context + original review + **revised paper text**. Ask for updated score.

R3, R4: Check `reviewer_dispatch` to determine how each was originally dispatched:
- If `reviewer_dispatch.R<N> == "codex"`: use a **fresh** `mcp__codex__codex` call (REVIEWER_MODEL, `config: {"model_reasoning_effort": "xhigh"}`) — NOT `codex-reply` — because the original thread contains the old paper text. The fresh call includes: the reviewer's original review, the author's rebuttal response, and the **revised full paper source**. Prompt: "You are R<N> (<persona>). Here is your original review, the author's response, and the revised paper. Update your score. Raise ONLY if concerns genuinely addressed. Do not change out of politeness."
- If `reviewer_dispatch.R<N> == "claude-fallback"` or `"claude"`: re-spawn reviewer agent with original review + rebuttal context + revised paper text.
- If `reviewer_dispatch.R<N>` is missing: treat as Claude-fallback.

Retry failures once, then fall back to Claude subagent. Instruction for all: "Raise score ONLY if concerns genuinely addressed."

Collect updated scores. **For any reviewer who fails rescoring after all retries, carry forward their initial score as their post-rebuttal score** (conservative: assume no improvement). Same coverage-based minimum-panel rule as Phase 1.

Update `nanoresearch.json`: `review_state.sub_phase: "area_chair"`, `review_state.scores.post_rebuttal: {...}`.

**Commit:** `git add AUTO_REVIEW.md nanoresearch.json && git commit -m "review: rescoring"`.

## Phase 4: Area Chair Decision

Invoke `area-chair` agent with: all reviews (initial + updated, 3-4 depending on panel), author rebuttal, paper text, `results.tsv` content, pre-computed score table `{R1: X, R2: Y, ...}`, pre-computed average, `any_score <= STRONG_REJECT_VETO` boolean, ACCEPTANCE_THRESHOLD, STRONG_REJECT_VETO. Also include `reviewer_dispatch` so the AC knows which reviewers were cross-model vs. same-model.

AC produces: meta-review, decision (ACCEPT/REJECT), if reject → top 3 actionable resubmission requirements (each must be specific and measurable — not vague).

**Commit:** `git add nanoresearch.json && git commit -m "review: AC decision"`.

## Phase 5: Record Decision

Append full cycle to `AUTO_REVIEW.md`:

```markdown
## Submission Cycle N
### Reviews
| R | Persona | Model | Score | Updated | Confidence |
### Rebuttal [per-reviewer responses]
### Area Chair Meta-Review [verbatim]
### Decision: ACCEPT / REJECT
```

**If REJECTED:** Replace (not append) the `## Resubmission Requirements` section in `EXPERIMENT_SPEC.md` with the AC's requirements for Cycle N. Validate requirements: must have at least 2 specific, actionable items. Preserve all existing fields (metric, command, extraction, files in scope, constraints, guard). Remove any prior `## Rebuttal Addendum` section. The loop reads the full spec; the new requirements guide what changes to attempt. **Patch EXPERIMENT_SPEC.md BEFORE updating decision_gate** — if the process crashes after patching but before the state update, the AC phase will re-run and re-patch (idempotent). `git add EXPERIMENT_SPEC.md && git commit -m "review: resubmission requirements"`.

Update `nanoresearch.json`:
- `review_state.sub_phase: "decision_gate"`
- `review_state.decision: "accepted"/"rejected"`
- Compute `avg` as arithmetic mean of all post-rebuttal scores (use initial scores for any reviewer missing from `post_rebuttal`)
- Append `{cycle: N, avg: <computed>, decision: "accepted"/"rejected"}` to `review_state.score_history`

**Commit:** `git add AUTO_REVIEW.md nanoresearch.json && git commit -m "review: cycle [N] decision"`.

Return control to the orchestrator.

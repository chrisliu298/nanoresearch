---
description: "Review phase: simulated peer review with 4 reviewers, rebuttal, area chair decision. Use for 'nanoresearch:review'."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply
---

# Review: Simulated Peer Review

Read from **CLAUDE.md**: NUM_REVIEWERS, ACCEPTANCE_THRESHOLD, STRONG_REJECT_VETO, REBUTTAL_EXPERIMENT_BUDGET_MINUTES, REVIEWER_MODEL.

Before composing Codex prompts, read **docs/prompting-codex.md**.

## Overview

One submission cycle: 4 reviews → rebuttal → rescoring → area chair decision. The orchestrator (`nanoresearch.md`) handles the reject→revise outer loop; this command runs one cycle and returns a decision.

## Resumption

Check `nanoresearch.json.review_state`:
- If absent → initialize per state schema with `cycle: 1`, `sub_phase: "initial_review"`, `results_row_at_cycle_start: <current results.tsv row count excl. header>`.
- If present, validate required fields. Reconstruct missing fields from on-disk artifacts (`paper/reviews/cycle-{N}/`, `AUTO_REVIEW.md`) where possible; otherwise re-initialize to schema defaults. **Never blindly reset `score_history`** — it accumulates across cycles; reconstruct from `AUTO_REVIEW.md` cycle headers if missing.
- **On resume:** Reconcile state against saved cycle artifacts. Reuse valid cached reviews (> 100 bytes, correct headings, parseable scores). Skip re-dispatching reviewers with both valid cached files and populated scores. Rebuild `scores` and `reviewer_dispatch` from saved files if inconsistent.
- If `sub_phase` exists and `cycle` matches current cycle → resume from recorded sub-phase.
  - `initial_review` → start Phase 1. Skip re-dispatching reviewers whose reviews exist in `paper/reviews/cycle-{N}/R{K}.md` AND whose scores are in `review_state.scores.initial`.
  - `rebuttal_triage` → start Step 2a.
  - `rebuttal_experiments` → start Step 2b.
  - `rebuttal_revision` → start Step 2c. If `rebuttal_revision_progress` is null or `impacted_sections` is null, recompute from `paper/reviews/cycle-{N}/rebuttal-plan.md` and new results.tsv rows before proceeding.
  - `rebuttal_response` → start Step 2d.
  - `rescoring` → start Phase 3.
  - `area_chair` → start Phase 4. If `paper/reviews/cycle-{N}/area-chair.md` exists, consume it instead of re-running AC.
  - `decision_gate` → start Phase 5. If `review_state.decision` is null, fall back to `area_chair` sub-phase (re-run Phase 4).
- Otherwise → start from Phase 1.

## Reviewer Panel

| Reviewer | Model | Dispatch |
|----------|-------|----------|
| R1 | Claude | Agent (reviewer.md, Methodologist) |
| R2 | Claude | Agent (reviewer.md, Empiricist) |
| R3 | GPT-5.4 | Codex MCP (Novelty Critic) |
| R4 | GPT-5.4 | Codex MCP (Skeptic) |

Claude reviewers can read code + `results.tsv`. GPT-5.4 reviewers see paper text only. If Codex unavailable or `codex: off`, run all 4 as Claude subagents.

## Phase 1: Initial Review

Concatenate all `.tex` section files into paper text. Create `paper/reviews/cycle-{N}/` directory for this cycle's artifacts.

**Cycle header (idempotent):** Check if `AUTO_REVIEW.md` already contains `## Submission Cycle {N}`. If not, append `## Submission Cycle {N} — Initial Reviews`. If already present (crash-resume), skip. This provides a machine-parseable cycle boundary for resume logic without creating duplicates on resume.

Launch R1, R2 via Agent tool (`run_in_background: true`) with results.tsv context (baseline + best kept metric). Launch R3, R4 via `mcp__codex__codex` (REVIEWER_MODEL, `config: {"model_reasoning_effort": "xhigh"}`). Apply `CODEX_CALL_TIMEOUT_MINUTES` fallback to Claude subagent if timeout; record `reviewer_dispatch.R<N>: "claude-fallback"`.

**R3/R4 Codex prompt template** (use XML tags per docs/prompting-codex.md):

```xml
<goal>
You are an independent reviewer (R{N}, {persona_name}) at a top ML venue.
Provide a thorough, honest review of this paper submission.
</goal>

<context>
<paper_text>{concatenated .tex sections}</paper_text>
<results_summary>Baseline: {metric}={value}. Best kept: {metric}={value} ({delta} improvement). Total experiments: {N}.</results_summary>
{if venue}<venue>{venue}</venue>{/if}
</context>

<calibration>
Scores 4-7 are typical. 5=borderline reject, 6=weak accept, 7=accept.
Do not inflate. Do not go above 7 unless exceptional or below 3 unless fundamentally flawed.
</calibration>

<output_contract>
{verbatim review format from agents/reviewer.md}
The "Overall Score" section must contain exactly one integer 1-10 on its own line, nothing else.
The "Confidence" section must contain exactly one integer 1-5 on its own line, nothing else.
Every weakness must have a severity tag (CRITICAL/MAJOR/MINOR), the affected section name(s), and a concrete fix suggestion.
</output_contract>

<verification_loop>
Before finalizing:
- Verify your score is consistent with the severity of your listed weaknesses.
- Verify every weakness has a concrete fix suggestion.
- Verify your recommendation matches your score.
</verification_loop>
```

Each reviewer prompt uses the calibration and format defined in `agents/reviewer.md`. R3/R4 also receive the `results_summary` (baseline + best kept metric + total experiments) so the Skeptic can verify overclaiming.

**After each successful reviewer dispatch**, immediately persist:
- **Validate review** against reviewer.md format (all mandated headings, parseable score 1-10, confidence 1-5, severity-tagged weaknesses). If invalid, retry once then fall back to Claude. Never coerce or fabricate scores.
- Save to `AUTO_REVIEW.md` (append) and `paper/reviews/cycle-{N}/R{K}.md`.
- Update `review_state.scores.initial.R<N>` and `reviewer_dispatch.R<N>`.
- Commit: `git add AUTO_REVIEW.md paper/reviews/ nanoresearch.json && git commit -m "review: R<N> complete"`.

Retry failures once. If Codex reviewers fail after retry, re-dispatch as Claude subagents with the same persona and record `reviewer_dispatch.R<N>: "claude-fallback"`.

**Minimum panel:** At least 3 reviewers must succeed with at least one from each category (methodology/empirics and novelty/skepticism). Below 3 → `status: "failed"` (infrastructure failure, NOT a scientific rejection).

**Cross-model diversity check:** After all dispatches complete, if all successful reviewers are the same model family (all Claude, including fallbacks), log a warning — synthesis rules should use the `codex: off` consensus threshold for this cycle.

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_triage"`. Commit: `git add AUTO_REVIEW.md nanoresearch.json && { git diff --cached --quiet || git commit -m "review: initial scores"; }`.

## Phase 2: Rebuttal

Four sub-steps, each checkpointed. On resume, skip completed sub-steps based on `review_state.sub_phase`.

### Step 2a: Triage (`sub_phase: "rebuttal_triage"`)

**Triage** all weaknesses: `experiment_gap` | `editorial_fix` | `clarification` | `overclaim` | `out_of_scope` | `reviewer_error`.

Honesty check: if >50% classified as `reviewer_error`, re-examine — the paper is likely unclear.

Produce a **rebuttal plan** mapping each weakness to its category and planned response. Save to `paper/reviews/cycle-{N}/rebuttal-plan.md` (cycle-scoped so resubmission cycles don't overwrite prior rebuttal artifacts).

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_experiments"`.

Commit: `git add paper/reviews/cycle-{N}/ nanoresearch.json && git commit -m "review: rebuttal triage"`.

### Step 2b: Rebuttal Experiments (`sub_phase: "rebuttal_experiments"`)

If no `experiment_gap` items exist in the rebuttal plan, skip steps 1-5 below and proceed directly to the sub_phase transition and commit at the end of this block.

1. Replace (not append) the `## Rebuttal Addendum` section in `EXPERIMENT_SPEC.md` with targeted experiments. If no such section exists, append it. This ensures idempotency on crash-retry.
2. **Validate spec integrity:** After patching, verify that immutable core fields (`## Command`, `## Metric Extraction`, `## Files in Scope`, `## Constraints`) still exist and are unchanged from pre-patch. If any are missing or altered, restore from the last committed version and re-patch. This guards against LLM accidentally rewriting the spec.
3. **Reset loop timer:** Set `loop_started_at: null` in `nanoresearch.json` so the rebuttal loop gets a fresh budget timer (the original value is stale from Phase 2).
4. **Commit the spec and timer reset before invoking the loop** — the loop's discard path uses `git reset --hard` which would destroy uncommitted changes: `git add EXPERIMENT_SPEC.md nanoresearch.json && git commit -m "review: rebuttal spec"`. Including `nanoresearch.json` ensures the timer reset survives crashes.
5. Invoke `/nanoresearch:loop ${REBUTTAL_EXPERIMENT_BUDGET_MINUTES}m`. The loop detects the `## Rebuttal Addendum` in `EXPERIMENT_SPEC.md` and enters fresh-from-spec mode automatically (skips resume, skips baseline if results.tsv has one).

**Post-loop validation:** Re-read `nanoresearch.json`. If `status == "failed"` (rebuttal loop preflight/baseline/runtime failure), commit and stop — do NOT proceed to rebuttal revision. Infrastructure failures must not be papered over.

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_revision"`.

Commit: `git add results.tsv autoresearch.md nanoresearch.json && { git diff --cached --quiet || git commit -m "review: rebuttal experiments"; }`.

### Step 2c: Structured Paper Revision (`sub_phase: "rebuttal_revision"`)

Do NOT edit the paper in a single monolithic pass. Use the write phase's section-by-section machinery:

1. **Identify impacted sections** from the rebuttal plan + new results.tsv rows. Apply dependency closure per write.md's revision mode rules. Persist in `review_state.rebuttal_revision_progress: {current_section: 0, impacted_sections: [...]}`. Commit immediately.

2. **No-new-results disclosure:** Check `results_row_at_cycle_start` to identify new rows (default `0` if null). If no new `keep` rows exist but experiments were expected, disclose honestly — never fabricate. Update `best_metric` and `iteration_count` from `results.tsv`.

3. **Edit each impacted section in-place.** For each section at index `i` in SECTION_ORDER (0 through 5): if `i` is NOT in `rebuttal_revision_progress.impacted_sections`, skip. Otherwise:
   - If `review_state.rebuttal_revision_progress.current_section > i`, skip (already done on prior run).
   - Read the existing `.tex` file + relevant reviewer weaknesses from `AUTO_REVIEW.md` (grep for section name to extract relevant weaknesses only) + new results.
   - Edit to address weaknesses. Preserve parts reviewers did not criticize. Soften overclaims. Add new results where applicable.
   - Submit to GPT-5.4 via `mcp__codex__codex` at `xhigh` for per-section review (same protocol as write.md Step 3). Prompt includes: section text, relevant reviewer weaknesses, new results, instruction to check claim-evidence alignment.
   - **Codex fallback:** If Codex MCP unavailable OR `codex: off`, spawn a Claude `write-critic` subagent in section mode.
   - Fix blocking issues. One review round per section.
   - **Update progress and commit section:** `review_state.rebuttal_revision_progress.current_section = i + 1`. `git add paper/sections/{name}.tex nanoresearch.json && git commit -m "review: rebuttal revision section {name}"`. This per-section commit prevents crash from losing prior section edits.

4. **Compile**: `cd paper && ${COMPILER} -pdf -interaction=nonstopmode main.tex`. Retry up to MAX_COMPILE_ATTEMPTS. Compile failure is non-fatal — rescoring reads .tex source.

Update `nanoresearch.json`: `review_state.sub_phase: "rebuttal_response"`.

Commit: `git add paper/ nanoresearch.json && git commit -m "review: rebuttal revision"`.

### Step 2d: Author Response (`sub_phase: "rebuttal_response"`)

Compose per-reviewer author response using the rebuttal plan and actual changes made:

```
> W1: [quoted weakness]
Response: [evidence / fix / clarification, citing specific section edits and new results]
```

Save to `paper/reviews/cycle-{N}/rebuttal-response.md`.

Update `nanoresearch.json`: `review_state.sub_phase: "rescoring"`.

Commit: `git add paper/reviews/cycle-{N}/ nanoresearch.json && git commit -m "review: author response"`.

## Phase 3: Post-Rebuttal Rescoring

Concatenate the **current** (revised) `.tex` section files into updated paper text. Each reviewer must see the revised paper, not the original.

**Only rescore reviewers in `participating_reviewers`.** Reviewers who failed Phase 1 have no initial review or score — skip them entirely during rescoring.

**Rescoring output contract:** Use the rescoring format from `agents/reviewer.md` (Overall Score, Confidence, Recommendation, What Changed). Same parse/retry/fallback rules as Phase 1. Post-rebuttal recommendations are authoritative for AC decision rule 3.

R1, R2 (Claude): spawn reviewer agent in `rescore` mode (see agents/reviewer.md `## Rescoring Format`) with rebuttal context + original review + **revised paper text**. Ask for updated score using the rescoring output contract.

R3, R4: Check `reviewer_dispatch` to determine how each was originally dispatched:
- If `reviewer_dispatch.R<N> == "codex"`: use a **fresh** `mcp__codex__codex` call (REVIEWER_MODEL, `config: {"model_reasoning_effort": "xhigh"}`) — NOT `codex-reply` — because the original thread contains the old paper text. The fresh call includes: the reviewer's original review, the author's rebuttal response, the **revised full paper source**, and a **results summary** (baseline + best kept metric + any new rows added since `results_row_at_cycle_start`). Prompt template:

```xml
<goal>
You are R{N} ({persona}). Re-evaluate this paper after the author's rebuttal.
</goal>

<context>
<your_original_review>{original review text}</your_original_review>
<author_rebuttal>{rebuttal response for this reviewer}</author_rebuttal>
<revised_paper>{revised .tex sections}</revised_paper>
<new_experimental_evidence>{new results.tsv rows since cycle start, if any}</new_experimental_evidence>
</context>

<output_contract>
### Overall Score
[single integer 1-10]
### Confidence
[single integer 1-5]
### Recommendation
[STRONG REJECT / REJECT / BORDERLINE REJECT / BORDERLINE ACCEPT / ACCEPT / STRONG ACCEPT]
### What Changed
[1-3 sentences]
</output_contract>

<calibration>
Scores 4-7 are typical. 5=borderline reject, 6=weak accept, 7=accept.
Do not inflate. Do not go above 7 unless exceptional or below 3 unless fundamentally flawed.
</calibration>

<constraints>
- Raise score ONLY if your concerns were genuinely addressed with evidence.
- Do not change score out of politeness.
- If the author claims new experiments, verify them against new_experimental_evidence. Flag any claimed result not present in the data.
</constraints>
```

- If `reviewer_dispatch.R<N> == "claude-fallback"` or `"claude"`: re-spawn reviewer agent with original review + rebuttal context + revised paper text + results summary.
- If `reviewer_dispatch.R<N>` is missing: treat as Claude-fallback.

Retry failures once, then fall back to Claude subagent.

**Persist each rescoring result incrementally** (mirror Phase 1 pattern):
- Save rescoring text to `paper/reviews/cycle-{N}/R{K}-rescore.md`.
- Append to `AUTO_REVIEW.md` under a `### Rescoring` sub-section.
- Update `review_state.scores.post_rebuttal.R<N>`.
- Commit after each: `git add AUTO_REVIEW.md paper/reviews/ nanoresearch.json && git commit -m "review: R<N> rescoring"`.

**Rescoring failure:** Carry forward initial score as post-rebuttal score, add reviewer ID to `scores.inherited`. **Inherited scores:** Exclude from STRONG_REJECT_VETO check unless fewer than 2 genuine rescores exist. Pass `inherited` list to AC.

Update `nanoresearch.json`: `review_state.sub_phase: "area_chair"`.

**Commit:** `git add AUTO_REVIEW.md nanoresearch.json && { git diff --cached --quiet || git commit -m "review: rescoring complete"; }`.

## Phase 4: Area Chair Decision

**Check for persisted AC output:** If `paper/reviews/cycle-{N}/area-chair.md` already exists (crash-resume after AC completed but before Phase 5), consume it instead of re-running the AC. This prevents non-deterministic regeneration.

Otherwise, invoke `area-chair` agent with: all reviews (initial + updated from `paper/reviews/cycle-{N}/`), author rebuttal, paper text, `results.tsv` content, pre-computed score table `{R1: X, R2: Y, ...}` (from `review_state.scores.post_rebuttal`, falling back to `initial` for missing entries — **only for reviewers in `participating_reviewers`**, ignore scores for non-participants even if present), pre-computed average (over `participating_reviewers` only — never divide by 4 if only 3 reviewed), `any_post_rebuttal_score <= STRONG_REJECT_VETO` boolean (computed over `participating_reviewers` only, excluding reviewers in `scores.inherited` unless fewer than 2 non-inherited scores exist — see Phase 3), `scores.inherited` list, ACCEPTANCE_THRESHOLD, STRONG_REJECT_VETO, post-rebuttal recommendation table. Also include `reviewer_dispatch` so the AC knows which reviewers were cross-model vs. same-model. **If `review_state.cycle > 1`**, also pass `EXPERIMENT_SPEC.md`'s `## Resubmission Requirements` section and instruct the AC to verify each requirement is addressed before making a decision.

**Score average:** Arithmetic mean over reviewers with scores in `scores.initial` (use `post_rebuttal` if available, else `initial`). Fewer than 3 with scores → `status: "failed"`.

AC produces: meta-review, decision (ACCEPT/REJECT), if reject → top 3 actionable resubmission requirements (each must be specific and measurable — not vague). Each requirement must specify (a) a specific change and (b) a measurable criterion for success. Items like "improve X" without specifying what to change or how to measure success are vague and must be re-prompted once.

**Persist AC output immediately:** Save to `paper/reviews/cycle-{N}/area-chair.md`. Update `review_state.decision` to `"accepted"/"rejected"`.

**Commit:** `git add paper/reviews/ nanoresearch.json && git commit -m "review: AC decision"`.

## Phase 5: Record Decision

**Idempotency check:** Before appending, check if `### Cycle {N} Summary` already exists in `AUTO_REVIEW.md`. If present, skip the append (already recorded on a prior crash-resume run). Otherwise, append cycle summary **under the existing `## Submission Cycle {N}` header** (created in Phase 1). Do NOT create a new top-level `## Submission Cycle N` header — one already exists. Append these sub-sections:

```markdown
### Cycle {N} Summary
| R | Persona | Model | Score | Updated | Confidence |
### Rebuttal [per-reviewer responses]
### Area Chair Meta-Review [verbatim from paper/reviews/cycle-{N}/area-chair.md]
### Decision: ACCEPT / REJECT
```

**If REJECTED:** Write `## Resubmission Requirements` in `EXPERIMENT_SPEC.md` (from AC output), then remove any prior `## Rebuttal Addendum`. Requirements must have at least 2 specific, actionable items with measurable success criteria; re-prompt AC once if vague. Validate spec integrity (same check as Step 2b).

Update `nanoresearch.json`:
- `review_state.sub_phase: "decision_gate"`
- `review_state.decision`: already set in Phase 4 (verify consistency)
- Compute `avg` as arithmetic mean over `participating_reviewers` only (use `post_rebuttal` scores where available, `initial` for missing)
- **Dedup check:** Before appending to `score_history`, check if an entry with the current `cycle` number already exists. If so, update it in place rather than appending (prevents duplicates on crash-resume).
- Append `{cycle: N, avg: <computed>, decision: "accepted"/"rejected"}` to `review_state.score_history`

**Atomic commit (includes both EXPERIMENT_SPEC.md and nanoresearch.json):** `git add AUTO_REVIEW.md EXPERIMENT_SPEC.md nanoresearch.json && git commit -m "review: cycle [N] decision"`. This ensures `review_state.sub_phase: "decision_gate"` and resubmission requirements are set atomically, preventing non-deterministic re-generation of requirements on crash-resume.

Return control to the orchestrator.

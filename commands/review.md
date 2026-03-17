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
- If `review_state` does not exist at all, initialize from scratch: `{cycle: 1, sub_phase: "initial_review", decision: null, codex_threads: {}, reviewer_dispatch: {}, participating_reviewers: [], effective_synthesis_mode: null, scores: {initial: {}, post_rebuttal: {}, inherited: []}, score_history: [], results_row_at_cycle_start: <current row count of results.tsv, excluding header>, rebuttal_revision_progress: null}`.
- Validate `review_state` has required fields (`cycle`, `sub_phase`, `scores`, `score_history`, `reviewer_dispatch`, `participating_reviewers`, `results_row_at_cycle_start`, `rebuttal_revision_progress`). If any field is missing, re-initialize it with care:
  - `score_history`: if missing, attempt to reconstruct from `## Submission Cycle` headers in `AUTO_REVIEW.md` (parse each header → find `### Decision` → extract decision; find `### Scores` table → compute average from "Post" column over participating reviewers); if impossible, initialize to `[]` with a warning. **Never blindly reset** — it accumulates across cycles.
  - `scores`: if missing, re-initialize to `{initial: {}, post_rebuttal: {}, inherited: []}`. If `scores` exists but `inherited` is missing, add `inherited: []`.
  - `codex_threads`, `reviewer_dispatch`, `decision`: safe to re-initialize to defaults.
  - `participating_reviewers`: reconstruct from `paper/reviews/cycle-{N}/` — any reviewer with both a file `R{K}.md` on disk and a score in `scores.initial` is a participant. If impossible, initialize to `[]`.
  - `results_row_at_cycle_start`: if missing, first attempt to reconstruct from the git commit at the cycle's review entry; if impossible, default to `0` with a warning (treats all post-baseline rows as potentially new).
  - `rebuttal_revision_progress`: if missing and `sub_phase` is `"rebuttal_revision"`, re-initialize to `{current_section: 0, impacted_sections: null}` (will be recomputed from the rebuttal plan). If `sub_phase` is not `"rebuttal_revision"`, safe to set to `null`.
  - `cycle`: if missing and `score_history` is empty, default to `1`. If `score_history` is non-empty: if the latest entry has a decision, infer `max(entry.cycle for entry in score_history) + 1`; otherwise `max(entry.cycle for entry in score_history)` (current cycle still active).
- **Reconciliation step (ALWAYS runs on resume):** Before skipping any reviewer dispatch, parse `paper/reviews/cycle-{N}/` for already-completed review artifacts. Rebuild missing `scores.initial`, `reviewer_dispatch`, `codex_threads`, and `participating_reviewers` entries from saved files. Also rebuild `scores.post_rebuttal` from existing `R{K}-rescore.md` files (validate full rescoring schema: must contain `### Overall Score` with parseable integer 1-10, `### Confidence` with parseable integer 1-5, and `### Recommendation` with a valid category; if any required field is missing or malformed, delete the cached file and mark the reviewer for redispatch). Only treat a reviewer as complete when both the raw review file AND its structured JSON fields are present. Update `participating_reviewers` incrementally as each reviewer is confirmed. During Phase 3 (rescoring), skip re-dispatching reviewers whose valid `R{K}-rescore.md` already exists and whose `scores.post_rebuttal.R{K}` is populated.
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

| Reviewer | Model | Dispatch | Persona |
|----------|-------|----------|---------|
| R1 | Claude | Agent (subagent) | **Methodologist** — soundness, proofs, correctness |
| R2 | Claude | Agent (subagent) | **Empiricist** — baselines, ablations, significance |
| R3 | GPT-5.4 xhigh | Codex MCP | **Novelty Critic** — originality, related work |
| R4 | GPT-5.4 xhigh | Codex MCP | **Skeptic** — overclaiming, limitations |

Claude reviewers can read code + `results.tsv`. GPT-5.4 reviewers see paper text only.

**Codex fallback:** If Codex MCP unavailable OR `nanoresearch.json.codex == "off"`, run all 4 as Claude subagents.

## Phase 1: Initial Review

Concatenate all `.tex` section files into paper text. Create `paper/reviews/cycle-{N}/` directory for this cycle's artifacts.

**Cycle header (idempotent):** Check if `AUTO_REVIEW.md` already contains `## Submission Cycle {N}`. If not, append `## Submission Cycle {N} — Initial Reviews`. If already present (crash-resume), skip. This provides a machine-parseable cycle boundary for resume logic without creating duplicates on resume.

Launch R1, R2 via Agent tool (`run_in_background: true`). Include in the R1/R2 Agent prompt: "Read `results.tsv` to verify claims against experimental data. Baseline metric: {value}. Best kept metric: {value}." This ensures Claude reviewers actively check claims against ground truth. Launch R3, R4 via `mcp__codex__codex` (REVIEWER_MODEL, `config: {"model_reasoning_effort": "xhigh"}` — xhigh is the only permitted effort level, no exceptions). **Codex timeout:** If any `mcp__codex__codex` call does not return within `CODEX_CALL_TIMEOUT_MINUTES` (10 minutes), treat it as a failure and immediately fall back to a Claude subagent with the same persona. Record `reviewer_dispatch.R<N>: "claude-fallback"`. This timeout applies to ALL Codex calls across the review phase (initial review, rescoring).

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

**After each successful reviewer dispatch**, immediately persist its result:
- **Validate review schema** (not just score): verify all mandated headings exist (`### Summary`, `### Overall Score`, `### Confidence`, `### Weaknesses`, `### Recommendation`), score is a parseable integer 1-10, confidence is a parseable integer 1-5, at least one weakness with severity tag exists, recommendation is one of the defined categories. If schema validation fails, retry once; if still invalid, fall back to Claude subagent.
- Save the review text to both `AUTO_REVIEW.md` (append under cycle header) and `paper/reviews/cycle-{N}/R{K}.md` (cycle-scoped file for resume).
- Update `review_state.scores.initial.R<N>` with the extracted score.
- Update `review_state.reviewer_dispatch.R<N>` to `"claude"`, `"codex"`, or `"claude-fallback"`.
- If Codex: save `review_state.codex_threads.R<N>` with the threadId. When re-dispatching a reviewer on resume, always overwrite `codex_threads.R<N>` with the new threadId.
- Commit: `git add AUTO_REVIEW.md paper/reviews/ nanoresearch.json && git commit -m "review: R<N> complete"`.

**Score extraction:** Each review must contain `### Overall Score` followed by a single integer 1-10 on its own line. If not parseable, issue a **fresh** `mcp__codex__codex` call (not `codex-reply`) with the score format constraint in the `<output_contract>`. If still unparseable after retry, do NOT coerce or fabricate a score — fall back to Claude subagent redispatch. Only persist scores that satisfy the exact output contract (single integer 1-10).

Retry failures once. If Codex reviewers fail after retry, re-dispatch as Claude subagents with the same persona and record `reviewer_dispatch.R<N>: "claude-fallback"`.

**Minimum panel (coverage-based):** At least 3 reviewers must succeed AND at least one must cover methodology/empirics (R1 or R2) and at least one must cover novelty/skepticism (R3 or R4). **Hard fail below 3:** set `status: "failed"` in `nanoresearch.json` with reason "insufficient_review_panel", append the failure to `AUTO_REVIEW.md`, commit, and stop. Do NOT convert panel insufficiency into `decision: "rejected"` — infrastructure failures must not be recorded as scientific rejections. **Record the participating reviewer set** in `review_state` (e.g., `participating_reviewers: ["R1", "R2", "R4"]`) — update incrementally as each reviewer completes. This set defines the denominator for all subsequent averaging.

**Cross-model diversity check:** After all dispatches complete, inspect `reviewer_dispatch`. If all successful reviewers are the same model family (all Claude, including fallbacks), log a warning and set `review_state.effective_synthesis_mode: "codex_off"` so synthesis rules use the `codex: off` heuristic ("3+ of N reviewers → must-fix") for this cycle, even if `codex` is `"on"` in nanoresearch.json.

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

1. **Identify impacted sections** from the rebuttal plan + new results.tsv rows. Always revisit: experiments, introduction, conclusion, abstract. Revisit method only if protocol changed. Revisit related_work only if positioning/novelty claim changed. Apply dependency closure: if experiments is in the impacted set → force include introduction, conclusion, abstract; if method is in the impacted set → force include experiments + all downstream; if related_work is in the impacted set → force include introduction (contribution framing depends on positioning). **Persist** the impacted set in `review_state.rebuttal_revision_progress: {current_section: 0, impacted_sections: [...]}`. **Commit immediately:** `git add nanoresearch.json && git commit -m "review: computed rebuttal impacted sections"`. This makes the computed set durable and prevents non-deterministic recomputation on crash-resume.

2. **No-new-results disclosure:** Before editing, check `review_state.results_row_at_cycle_start`. If null, first attempt to reconstruct from the git commit at the cycle's review entry; if impossible, default to `0` (treat all post-baseline rows as potentially new — this is conservative in the correct direction) and log a warning. If no new `keep` rows were added since the cycle began and the rebuttal plan contained `experiment_gap` items, note this explicitly in the experiments section ("We attempted X but found no improvement") rather than fabricating claims. Same honesty rule as write.md revision mode. **Update metrics:** After rebuttal experiments, always recompute `iteration_count` from the max `#` in `results.tsv` (regardless of whether metrics improved). Parse `results.tsv` for the best `keep` row metric; if improved, update `nanoresearch.json.best_metric`.

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

**Rescoring output contract** (apply to ALL rescoring prompts — Claude and Codex):

```
### Overall Score
[single integer 1-10 on its own line]
### Confidence
[single integer 1-5 on its own line]
### Recommendation
[STRONG REJECT / REJECT / BORDERLINE REJECT / BORDERLINE ACCEPT / ACCEPT / STRONG ACCEPT]
### What Changed
[1-3 sentences: what specifically improved, worsened, or stayed the same]
```

Post-rebuttal recommendations are authoritative for AC decision rule 3 (majority recommendation). Pass a post-rebuttal recommendation table to the AC alongside the score table.

Use the same parse/retry/fallback rules as Phase 1 for post-rebuttal reviews.

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

**For any reviewer who fails rescoring after all retries, carry forward their initial score as their post-rebuttal score** (conservative: assume no improvement). Fill `review_state.scores.post_rebuttal.R<N>` from `review_state.scores.initial.R<N>` and add the reviewer ID to `review_state.scores.inherited` (e.g., `inherited: ["R3"]`). Persist this immediately. The coverage-based minimum-panel rule applies to the initial panel, not rescoring — if the initial panel met coverage, rescoring proceeds with inherited scores for failed reviewers. **Inherited score veto guard:** When computing `any_post_rebuttal_score <= STRONG_REJECT_VETO`, exclude reviewers listed in `scores.inherited` — the reviewer did not have the opportunity to re-evaluate after seeing the rebuttal. **Exception:** If fewer than 2 non-inherited post-rebuttal scores exist, apply the veto check against ALL post-rebuttal scores (including inherited), since insufficient fresh evidence exists. Pass the `inherited` list to the AC.

Update `nanoresearch.json`: `review_state.sub_phase: "area_chair"`.

**Commit:** `git add AUTO_REVIEW.md nanoresearch.json && { git diff --cached --quiet || git commit -m "review: rescoring complete"; }`.

## Phase 4: Area Chair Decision

**Check for persisted AC output:** If `paper/reviews/cycle-{N}/area-chair.md` already exists (crash-resume after AC completed but before Phase 5), consume it instead of re-running the AC. This prevents non-deterministic regeneration.

Otherwise, invoke `area-chair` agent with: all reviews (initial + updated from `paper/reviews/cycle-{N}/`), author rebuttal, paper text, `results.tsv` content, pre-computed score table `{R1: X, R2: Y, ...}` (from `review_state.scores.post_rebuttal`, falling back to `initial` for missing entries — **only for reviewers in `participating_reviewers`**, ignore scores for non-participants even if present), pre-computed average (over `participating_reviewers` only — never divide by 4 if only 3 reviewed), `any_post_rebuttal_score <= STRONG_REJECT_VETO` boolean (computed over `participating_reviewers` only, excluding reviewers in `scores.inherited` unless fewer than 2 non-inherited scores exist — see Phase 3), `scores.inherited` list, ACCEPTANCE_THRESHOLD, STRONG_REJECT_VETO, post-rebuttal recommendation table. Also include `reviewer_dispatch` so the AC knows which reviewers were cross-model vs. same-model. **If `review_state.cycle > 1`**, also pass `EXPERIMENT_SPEC.md`'s `## Resubmission Requirements` section and instruct the AC to verify each requirement is addressed before making a decision.

**Score average:** Compute as arithmetic mean over reviewers in `participating_reviewers` only. For each reviewer in `participating_reviewers`, use `post_rebuttal` score if available, else `initial` score. **Ignore scores for any reviewer not in `participating_reviewers`**, even if present in `scores.initial` or `scores.post_rebuttal`. If fewer than 3 reviewers in `participating_reviewers` have scores, set `status: "failed"` (insufficient panel — do NOT convert to `decision: "rejected"`).

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

**If REJECTED:** Write `## Resubmission Requirements` in `EXPERIMENT_SPEC.md` FIRST (from `paper/reviews/cycle-{N}/area-chair.md`), THEN remove any prior `## Rebuttal Addendum` section. This order ensures the spec always has at least one marker if the process crashes mid-update. Validate requirements: must have at least 2 specific, actionable items — each must specify (a) a concrete change and (b) a measurable success criterion. If requirements fail validation, re-prompt the AC once before persisting. **Validate spec integrity after patch:** verify immutable core fields (`## Command`, `## Metric Extraction`, `## Files in Scope`, `## Constraints`) still exist and are unchanged. If any are missing or altered, restore from the last committed version and re-patch.

Update `nanoresearch.json`:
- `review_state.sub_phase: "decision_gate"`
- `review_state.decision`: already set in Phase 4 (verify consistency)
- Compute `avg` as arithmetic mean over `participating_reviewers` only (use `post_rebuttal` scores where available, `initial` for missing)
- **Dedup check:** Before appending to `score_history`, check if an entry with the current `cycle` number already exists. If so, update it in place rather than appending (prevents duplicates on crash-resume).
- Append `{cycle: N, avg: <computed>, decision: "accepted"/"rejected"}` to `review_state.score_history`

**Atomic commit (includes both EXPERIMENT_SPEC.md and nanoresearch.json):** `git add AUTO_REVIEW.md EXPERIMENT_SPEC.md nanoresearch.json && git commit -m "review: cycle [N] decision"`. This ensures `review_state.sub_phase: "decision_gate"` and resubmission requirements are set atomically, preventing non-deterministic re-generation of requirements on crash-resume.

Return control to the orchestrator.

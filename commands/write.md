---
description: "Write phase: turn experiment results into a paper or research memo. Use for 'nanoresearch:write', 'write paper', 'draft paper'."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, mcp__codex__codex, mcp__codex__codex-reply
---

# Write: Results → Paper (or Research Memo)

Two-sub-phase writing loop: section-by-section drafting with per-section review, followed by whole-paper revision passes with a 4+4 panel.

Read from **CLAUDE.md**: COMPILER, MAX_COMPILE_ATTEMPTS, NUM_REVISION_PASSES, REVISION_PANEL, SECTION_REVIEW_EFFORT.

Before composing Codex prompts, read **docs/prompting-codex.md**.

## Constants

```
SECTION_ORDER = [method, experiments, related_work, introduction, conclusion, abstract]
REVISION_LENSES = [evidence, structure, positioning, clarity]
```

## Inputs

Read `results.tsv` (ground truth) and optionally `IDEA.md`. Construct the results narrative directly from `results.tsv`.

If `IDEA.md` does not exist (e.g., `skip-scout` mode), construct context from `EXPERIMENT_SPEC.md` or `autoresearch.md`.

Check `nanoresearch.json.scout_state.novelty_confidence`. If `"low"` or if `scout_state` is absent/missing `novelty_confidence`, default to cautious novelty language throughout (avoid strong novelty claims; frame contributions as empirical findings rather than novel methods). Only use confident novelty framing when `novelty_confidence` is explicitly `"high"`.

## Mode Detection

Check `nanoresearch.json`:
- If `review_state` exists AND `review_state.cycle > 1` AND `paper/main.tex` exists → **revision mode** (edit existing paper).
- If `paper/main.tex` does not exist → **new paper mode** or **memo mode**.
- Default → assess quality gate below.

## Resumption

Check `nanoresearch.json.write_state`:
- If absent → **initialize**: `{sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}`.
- If `write_state.sub_phase == "complete"` AND `paper/main.tex` exists → skip write phase (already done). If `write_state.sub_phase == "complete"` AND `paper/main.tex` does NOT exist → log warning "write_state is complete but paper/main.tex missing; resetting", then reset `write_state` to `{sub_phase: "section_drafting", current_section: 0, revision_pass: 0, impacted_sections: null}` and proceed.
- **Gap-state normalization (run before generic resume):**
  - If `write_state.sub_phase == "section_drafting"` and `current_section >= len(SECTION_ORDER)` → transition to `{sub_phase: "revision", current_section: len(SECTION_ORDER), revision_pass: 0}`.
  - If `write_state.sub_phase == "revision"` and `revision_pass >= NUM_REVISION_PASSES` → transition to `sub_phase: "complete"`.
- **Resume from recorded position:**
  - If `write_state.sub_phase == "section_drafting"` → resume at `current_section`. Sections with existing `.tex` files at indices < `current_section` are kept.
  - If `write_state.sub_phase == "revision"` → resume at `revision_pass`.

**Revision mode pre-flight:** If revision mode (`review_state.cycle > 1`) and `impacted_sections` is null → recompute from AC requirements and new results.tsv rows. Persist and commit immediately. Never iterate with null `impacted_sections` in revision mode.

Detect revision mode from `review_state.cycle > 1`, not from write_state. The orchestrator resets `write_state` when entering a new write cycle after rejection.

## Quality Gate (new paper mode only)

**Paper-worthy** (all must hold):
- Best metric shows meaningful improvement over baseline (not noise)
- At least 3 kept experiments
- Improvement attributable to a specific change

**Memo-worthy** (any): null/negative results, fewer than 3 keeps, no clear improvement.

If memo → write `RESEARCH_MEMO.md` and return. (Format: question, hypothesis, approach, results table, why it didn't work, what we learned, next directions.)

## Planning

Before any section drafting, build these artifacts:

### Claims-Evidence Matrix

```
| Claim | Evidence (experiment) | Metric delta | Strength |
```

Every claim must map to an accepted experiment in `results.tsv`. No claim without evidence.

### Paper Facts Packet

Build a compact digest shared with all reviewers. Use this XML template so all Codex prompts receive structured, consistent context:

```xml
<paper_facts>
  <problem>
  - [2-3 bullet points]
  </problem>
  <method>
  - [3-5 bullet points]
  </method>
  <results>
  - Baseline: [metric] = [value]
  - Best kept: [metric] = [value] ([delta] improvement)
  - Key ablations: [1-2 sentences]
  </results>
  <limitations>
  - [explicit limitation 1]
  - [explicit limitation 2]
  </limitations>
  <claims_evidence>
  | Claim | Experiment # | Metric delta | Strength |
  </claims_evidence>
</paper_facts>
```

**Ground truth rule:** `best_metric` and all claims must reference `keep` rows only from `results.tsv`. Never cite `discard` or `crash` rows as evidence.

### Outline

Section structure matching SECTION_ORDER. For each section: purpose, key content, target length, figures/tables planned.

### Paper Skeleton

Create `paper/main.tex` + `paper/sections/*.tex` stubs + `paper/references.bib` + `paper/figures/` + `paper/reviews/`. Compile skeleton to catch template/package issues early.

## Sub-Phase 1: Section Drafting

`write_state.sub_phase = "section_drafting"`

Draft sections in SECTION_ORDER: **method → experiments → related_work → introduction → conclusion → abstract**.

Evidence-bearing sections first; introduction and abstract written last.

For each section at index `i`:

### Step 1: Draft

- If revision mode: read existing `paper/sections/{name}.tex` + relevant feedback from `AUTO_REVIEW.md`. Edit in place — do not rewrite from scratch.
- If new paper mode: draft from IDEA.md + results.tsv + previously drafted sections + paper facts packet.
- Apply section-specific guidelines:
  - **method**: All hyperparameters, notation defined before use, design choices motivated.
  - **experiments**: Setup, baselines, metrics, every claim backed by a number from results.tsv.
  - **related_work**: ≥1 page. Organize by category with `\paragraph{}`. No fabricated citations. Position paper clearly against each cited work.
  - **introduction**: ~1.5 pages. Hook → gap → approach → contributions list → roadmap.
  - **conclusion**: ~0.5 pages. No new claims. Honest limitations. Concrete future work.
  - **abstract**: 150-250 words. Problem, approach, key quantitative result, implication. No citations.
- Use `\citep`/`\citet` (natbib). No AI slop: avoid "delve", "pivotal", "landscape", "furthermore", "it is worth noting".

### Step 2: Citations

For related_work and any section needing citations:
- Fetch real BibTeX via DBLP/CrossRef. Never fabricate.
- DBLP: `curl -s "https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json&h=3"` → extract key → `curl -s "https://dblp.org/rec/{key}.bib"`
- CrossRef fallback: `curl -sLH "Accept: application/x-bibtex" "https://doi.org/{doi}"`
- Mark unverified with `% [VERIFY]`. Resolve all `% [VERIFY]` before completion.

### Step 3: Per-Section Review

Submit section to GPT-5.4 via `mcp__codex__codex` at `xhigh`. Apply `CODEX_CALL_TIMEOUT_MINUTES` fallback to Claude `write-critic` subagent if timeout. Use XML-tagged prompt per `docs/prompting-codex.md`:

```xml
<goal>
Review ONE section of an ML paper draft before the author moves on.
Section: {section_name}. Purpose: {from outline}.
Review this section only. Do not review the whole paper.
</goal>

<context>
{paper_facts_packet}

<results_data>
Ground truth — filtered subset, max 15 rows.
For evidence-bearing sections (method, experiments): baseline row (#0), best kept row, and rows referenced by claims. For non-evidence sections: baseline and best only.
{filtered results.tsv rows}
</results_data>

<previous_sections>
{list of completed section names + 2-3 bullet digest each}
</previous_sections>

<current_section>
{section .tex content}
</current_section>
</context>

<constraints>
Review for:
1. Claims not supported by results data (check exact numbers against results_data)
2. Missing required content for this section type
3. Confusing argument flow
4. Generic filler, AI-sounding prose, overclaiming
5. Inconsistency with previously drafted sections
6. Citation gaps (describe what type of citation is needed — do NOT suggest specific papers by name)
7. Notation consistency with previously drafted sections
8. Paragraph structure and sentence variety
Max 500 words. Be specific and actionable. No praise. No listing things that are fine.
</constraints>

<output_contract>
## Verdict: PASS | REVISE
## Blocking Issues
- [location] — [issue] → [smallest concrete fix]
...or "None"
## Non-Blocking Issues
- [location] — [issue] → [suggested fix]
...or "None"
## Cross-Section Risks
- [potential inconsistency with previously drafted sections]
...or "None"
If a category has no issues, output "None" — do not hallucinate issues to fill space.
</output_contract>

<verification_loop>
Before finalizing:
- For each claim you flag as unsupported, verify it is actually absent from the results data provided.
- For each inconsistency you flag, cite the specific text in the previous section that conflicts.
</verification_loop>
```

**Codex fallback:** If Codex unavailable or `codex: off`, spawn Claude `write-critic` subagent in section mode.

**Review failure:** After Codex + Claude fallback both fail, log `REVIEW SKIPPED` and advance. Revision passes catch issues later.

### Step 4: Fix, Commit, and Advance

Fix all blocking issues. One review round per section — do not re-review. Non-blocking issues are deferred to revision passes.

Update `write_state.current_section = i + 1`. Update `nanoresearch.json`.

**Per-section checkpoint commit:** `git add paper/sections/{name}.tex paper/reviews/section-{idx}-{name}.md nanoresearch.json && git commit -m "write: drafted {name}"`.

### Post-Drafting

After all sections complete:

1. Verify all section files exist and `main.tex` `\input` paths are correct.
2. **Cross-section consistency check:** Verify claims-evidence matrix matches actual section claims. Fix contradictions and unsupported claims.
3. Clean `references.bib`: keep only cited entries.
4. **Compile**: `cd paper && ${COMPILER} -pdf -interaction=nonstopmode main.tex`. Fix errors, retry up to MAX_COMPILE_ATTEMPTS. Compile failure is non-fatal — revision reviewers read `.tex` source.
5. Transition: `write_state = {sub_phase: "revision", current_section: len(SECTION_ORDER), revision_pass: 0}`.

## Sub-Phase 2: Revision Passes

`write_state.sub_phase = "revision"`

NUM_REVISION_PASSES = 2 total passes. Each pass: review → synthesize → fix → compile.

### Pass Focus

| Pass | Focus | Scope |
|------|-------|-------|
| 0 (structural) | Claims-evidence alignment, cross-section coherence, narrative flow, missing content, section balance, reverse outline test | May trigger content restructuring |
| 1 (presentation) | De-AI polish, notation consistency, figure/table integration, citation hygiene, LaTeX quality, overfull hboxes, page count | Do NOT reopen structure unless factual error |

### Panel Dispatch

For each pass, launch 8 reviewers in parallel:

**4 Claude write-critic subagents** (`run_in_background: true`):
- Each spawned with the `write-critic` agent, a different lens from REVISION_LENSES, and the pass focus.

**4 GPT-5.4 via Codex MCP** (4 separate `mcp__codex__codex` calls at `xhigh`). Apply `CODEX_CALL_TIMEOUT_MINUTES` fallback to Claude `write-critic` subagent:
- Each assigned a lens from REVISION_LENSES.
- Fresh call per pass (no thread continuity between passes).

Each reviewer receives:
- Full paper `.tex` source (all sections concatenated)
- Paper facts packet
- Claims-evidence matrix
- Pass focus (structural or presentation)
- Lens assignment
- In revision mode: AUTO_REVIEW.md feedback

Prompt for Codex reviewers (XML-tagged per docs/prompting-codex.md):

```xml
<goal>
You are one critic in an 8-reviewer paper-improvement panel.
Your job is NOT to score the paper. Find the highest-leverage edits.
</goal>

<context>
<pass_focus>{structural | presentation}</pass_focus>
<your_lens>{evidence | structure | positioning | clarity}</your_lens>
{paper_facts_packet}
<paper_source>{all .tex sections concatenated}</paper_source>
{if revision_mode}<reviewer_feedback>{relevant AUTO_REVIEW.md excerpts}</reviewer_feedback>{/if}
</context>

<constraints>
1. Find at most 5 issues inside your lens.
2. For each: cite section/paragraph, explain why it matters, propose smallest fix.
3. Stay in your lane. Mention out-of-lens issues only if fatal.
4. If you find fewer than 5 issues, output only what you found. Zero is acceptable.
5. Do NOT suggest specific papers to cite by name — describe what type of citation is needed.
6. In structural pass: ignore grammar unless it blocks meaning.
7. In presentation pass: do not request major restructuring unless factual error.
8. Honest null-result reporting ("We attempted X but found no improvement") should NOT be flagged as a weakness.
</constraints>

<output_contract>
## Issues (max 5, ranked by severity)
1. [CRITICAL|MAJOR|MINOR] [section, paragraph] — [issue]
   Evidence: [quote or reference from paper_source]
   Fix: [smallest concrete change]
...or "No issues found under this lens"

## Cross-Cutting Problems
- [issues spanning multiple sections]
...or "None"

Max 600 words. No scores. No generic praise.
</output_contract>

<verification_loop>
For each issue, verify the cited evidence quote actually appears in paper_source.
Drop any issue you cannot ground in the provided text.
</verification_loop>
```

**Save each reviewer's output** to `paper/reviews/cycle-{C}/revision-pass-{n}/slot-{K}-{lens}.md` (C from `review_state.cycle` or `1`; K = stable slot 1-8, unchanged by fallback). Tag with `[Claude-{lens}]` or `[GPT5.4-{lens}]`. On resume, reuse valid cached outputs (> 100 bytes, correct headings); re-dispatch invalid ones.

### Panel Failure

Retry failed Codex once → Claude fallback. Minimum panel = 4 of 8; below 4, proceed with warning.

### Synthesis

Tag each review with its source: `[Claude-{lens}]` or `[GPT5.4-{lens}]` when collecting results.

After collecting all reviews:

1. **Deduplicate**: Group identical issues by location.
2. **Prioritize**: CRITICAL severity → always must-fix. **If cross-model diversity exists** (panel has both `[Claude-{lens}]` and `[GPT5.4-{lens}]` tags): same-lens slot-pair agreement OR 2+ lenses → must-fix. **If same model family** (`codex: off` or all fallbacks): 3+ of N reviewers → must-fix. **Degenerate panel** (<3 reviewers): 2+ → must-fix; single reviewer: CRITICAL and MAJOR are must-fix.
3. **Conflict resolution**: When reviewers disagree, prefer the action that keeps the paper closer to results.tsv. Ties favor the shorter paper (cut over expand).
4. **Produce**: Ordered fix list, mandatory items first.

### Implementation and Compile

1. Implement fixes top-down.
2. Compile: `${COMPILER} -pdf -interaction=nonstopmode main.tex`. Retry up to MAX_COMPILE_ATTEMPTS.
3. After the presentation pass (pass index 1): run format check — overfull hboxes, page count, undefined refs/citations.
4. Update `write_state.revision_pass = pass + 1`. Update `nanoresearch.json`.
5. **Per-pass checkpoint commit:** `git add paper/ nanoresearch.json && git commit -m "write: revision pass {n} complete"`.

### Completion

After final pass:
- Verify no `% [VERIFY]` markers remain (resolve or cut the claim).
- Final abstract/title/conclusion consistency check.
- `write_state.sub_phase = "complete"`.
- Save `paper/main_original.pdf` (new paper) or `paper/main_revised.pdf` (revision).

## Revision Mode (review_state.cycle > 1)

Same two-sub-phase loop with these differences:

1. **Additional inputs**: AUTO_REVIEW.md and EXPERIMENT_SPEC.md `## Resubmission Requirements` (source of truth for what must be addressed).
2. **Section drafting = section editing**: Each section is edited in-place, not rewritten. Per-section review prompt includes relevant reviewer feedback.
3. **Targeted sections**: Determine impacted sections from AC requirements (in EXPERIMENT_SPEC.md) and new results.tsv rows. Compute the impacted set once at the start and persist in `write_state.impacted_sections` (array of SECTION_ORDER indices). Always revisit: experiments, introduction, conclusion, abstract. Revisit method only if protocol changed. Revisit related_work only if positioning/novelty claim changed.
4. **Section skip**: When iterating through SECTION_ORDER, check if `i` is in `impacted_sections`. If not, skip: increment `current_section` and continue. Do not edit, review, or compile non-impacted sections.
5. **Dependency closure (computed once when building `impacted_sections`)**: experiments → force introduction, conclusion, abstract. method → force experiments + downstream. related_work → force introduction.
6. **New results check**: Rows with `#` > `results_row_at_cycle_start` (default `0` if null) are new for this cycle. If no new `keep` rows exist but experiments were expected, disclose honestly — never fabricate.
7. **Revision pass augmentation**: Pass 0 prompt adds: "Verify each resubmission requirement from EXPERIMENT_SPEC.md is addressed. Mark each as ADDRESSED or UNADDRESSABLE."

Per-section review prompt addition for revision mode:
```
## Reviewer Feedback (from previous submission)
{Relevant excerpts from AUTO_REVIEW.md for this section}

## Resubmission Requirements (from EXPERIMENT_SPEC.md)
{Area chair's structured requirements}

Edit the existing section to address these issues. Preserve parts reviewers did not criticize.
```

## Figures

Write Python scripts in `paper/figures/` that read `results.tsv`. Generate PDF plots (matplotlib, 300 DPI, serif, colorblind-safe, no titles). Run scripts during section drafting when a section references a planned figure.

## Rules

- **Large files**: If Write tool fails, retry via Bash heredoc.
- **No fabricated citations.** Every BibTeX entry must come from DBLP/CrossRef or be marked `% [VERIFY]`.
- **No unsupported claims.** Every claim must trace to results.tsv.
- **Bias toward deletion.** When evidence is weak, soften the claim.
- **One review round per section.** Revision passes catch the rest.
- **Update write_state before each expensive operation**, not after.
- **Reviews on disk.** Save to `paper/reviews/`. Keep nanoresearch.json small.
- If `nanoresearch.json` has a `venue` field, use appropriate formatting.

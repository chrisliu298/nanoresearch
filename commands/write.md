---
description: "Write phase: turn experiment results into a paper or research memo. Use for 'nanoresearch:write', 'write paper', 'draft paper'."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, mcp__codex__codex, mcp__codex__codex-reply
---

# Write: Results → Paper (or Research Memo)

Two-sub-phase writing loop: section-by-section drafting with per-section review, followed by whole-paper revision passes with a 4+4 panel.

Read from **CLAUDE.md**: COMPILER, MAX_COMPILE_ATTEMPTS, NUM_REVISION_PASSES, REVISION_PANEL, SECTION_REVIEW_EFFORT.

Before composing any Codex MCP prompt, read **docs/prompting-codex.md** for model-specific patterns (XML tags, output contracts, verification loops) that improve GPT-5.4 output quality.

## Constants

```
SECTION_ORDER = [method, experiments, related_work, introduction, conclusion, abstract]
REVISION_LENSES = [evidence, structure, positioning, clarity]
```

## Inputs

Read `results.tsv` (ground truth) and optionally `IDEA.md`. Construct the results narrative directly from `results.tsv`.

If `IDEA.md` does not exist (e.g., `skip-scout` mode), construct context from `EXPERIMENT_SPEC.md` or `autoresearch.md`.

## Mode Detection

Check `nanoresearch.json`:
- If `review_state` exists AND `review_state.cycle > 1` AND `paper/main.tex` exists → **revision mode** (edit existing paper).
- If `paper/main.tex` does not exist → **new paper mode** or **memo mode**.
- Default → assess quality gate below.

## Resumption

Check `nanoresearch.json.write_state`:
- If `write_state` exists and `sub_phase != "complete"` → **resume** from recorded position.
- If `write_state.sub_phase == "section_drafting"` and `current_section >= len(SECTION_ORDER)` → transition to `sub_phase: "revision", revision_pass: 0` (gap state recovery).
- If `write_state.sub_phase == "section_drafting"` → resume at `current_section`. Sections with existing `.tex` files at indices < `current_section` are kept.
- If `write_state.sub_phase == "revision"` and `revision_pass >= NUM_REVISION_PASSES` → transition to `sub_phase: "complete"` (gap state recovery).
- If `write_state.sub_phase == "revision"` → resume at `revision_pass`.
- If `write_state.sub_phase == "complete"` → skip write phase (already done).
- If absent → **initialize**: `{sub_phase: "section_drafting", current_section: 0, revision_pass: 0}`.

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

Build a compact digest shared with all reviewers:
- Problem statement (2-3 bullets)
- Proposed method (3-5 bullets)
- Key results with exact numbers from results.tsv
- Explicit limitations / caveats
- Claims-evidence matrix

### Outline

Section structure matching SECTION_ORDER. For each section: purpose, key content, target length, figures/tables planned.

### Paper Skeleton

Create `paper/main.tex` + `paper/sections/*.tex` stubs + `paper/references.bib` + `paper/figures/` + `paper/reviews/`. Compile skeleton to catch template/package issues early.

## Sub-Phase 1: Section Drafting

`write_state.sub_phase = "section_drafting"`

Draft sections in SECTION_ORDER: **method → experiments → related_work → introduction → conclusion → abstract**.

Evidence-bearing sections first (method, experiments) anchor the paper. Introduction and abstract are written after the core to prevent claim drift.

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

Submit section to GPT-5.4 via `mcp__codex__codex` at `xhigh` effort (the only permitted effort level — no other level is allowed):

```
mcp__codex__codex:
  model: gpt-5.4
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    You are reviewing ONE section of an ML paper draft before the author moves on.

    Section: {section_name}
    Section purpose: {from outline}

    Review this section only. Do not review the whole paper.

    Paper facts:
    {paper_facts_packet}

    Results data (ground truth):
    {results.tsv content or relevant subset}

    Previously drafted sections (for consistency checking):
    {list of completed section names + 2-3 bullet digest each}

    Current section draft:
    {section .tex content}

    Review for:
    1. Claims not supported by results.tsv data (check exact numbers)
    2. Missing required content for this section type
    3. Confusing argument flow
    4. Generic filler, AI-sounding prose, overclaiming
    5. Inconsistency with previously drafted sections
    6. Citation gaps (for any section, not just related_work)

    Output exactly:
    ## Verdict: PASS | REVISE
    ## Blocking Issues
    - [location] — [issue] → [smallest concrete fix]
    ## Non-Blocking Issues
    - [location] — [issue] → [suggested fix]
    ## Cross-Section Risks
    - [potential inconsistency with previously drafted sections]

    Rules: max 500 words. Be specific and actionable. No praise. No listing things that are fine.
```

**Codex fallback:** If Codex MCP unavailable OR `nanoresearch.json.codex == "off"`, spawn a Claude `write-critic` subagent in section mode with the same prompt content.

### Step 4: Fix and Advance

Fix all blocking issues. One review round per section — do not re-review. Non-blocking issues are deferred to revision passes.

Update `write_state.current_section = i + 1`. Update `nanoresearch.json`.

### Post-Drafting

After all sections complete:

1. Verify all section files exist and `main.tex` `\input` paths are correct.
2. Clean `references.bib`: keep only cited entries.
3. **Compile**: `cd paper && ${COMPILER} -pdf -interaction=nonstopmode main.tex`. Fix errors, retry up to MAX_COMPILE_ATTEMPTS. Compile failure is non-fatal — revision reviewers read `.tex` source.
4. Transition: `write_state = {sub_phase: "revision", current_section: len(SECTION_ORDER), revision_pass: 0}`.

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

**4 GPT-5.4 via Codex MCP** (4 separate `mcp__codex__codex` calls at `xhigh` — the only permitted effort level):
- Each assigned a lens from REVISION_LENSES.
- Fresh `mcp__codex__codex` call per pass (no thread continuity between passes — different pass foci would contaminate each other).

Each reviewer receives:
- Full paper `.tex` source (all sections concatenated)
- Paper facts packet
- Claims-evidence matrix
- Pass focus (structural or presentation)
- Lens assignment
- In revision mode: AUTO_REVIEW.md feedback

Prompt for Codex reviewers:

```
You are one critic in an 8-reviewer paper-improvement panel.

Pass focus: {structural | presentation}
Your lens: {evidence | structure | positioning | clarity}

Your job is NOT to score the paper. Find the highest-leverage edits.

Paper facts:
{paper_facts_packet}

Paper source:
{all .tex sections concatenated}

Instructions:
1. Find at most 5 issues inside your lens.
2. For each: cite section/paragraph, explain why it matters, propose smallest fix.
3. Stay in your lane. Mention out-of-lens issues only if fatal.

Output exactly:
## Issues (max 5, ranked by severity)
1. [CRITICAL|MAJOR|MINOR] [section, paragraph] — [issue]
   Evidence: [quote or reference]
   Fix: [smallest concrete change]

## Cross-Cutting Problems
- [issues spanning multiple sections]
...or "None"

Rules: Max 600 words. No scores. No generic praise. Do not list things that are fine.
In structural pass: ignore grammar unless it blocks meaning.
In presentation pass: do not request major restructuring unless factual error.
```

### Panel Failure Handling

- Retry failed Codex calls once. If still failing, replace with Claude `write-critic` subagent using same lens.
- Minimum panel = 4 of 8. Below 4, retry all failed reviewers once more. If still below 4, proceed with available reviews and log warning.

### Synthesis

After collecting all reviews:

1. **Deduplicate**: Group identical issues by location.
2. **Prioritize**: Issues flagged by both model families in the same lens → must-fix. Issues flagged across 2+ lenses → must-fix. Single-reviewer stylistic nits → optional.
3. **Conflict resolution**: When reviewers disagree, prefer the action that keeps the paper closer to results.tsv. Ties favor the shorter paper (cut over expand).
4. **Produce**: Ordered fix list, mandatory items first.

### Implementation and Compile

1. Implement fixes top-down.
2. Compile: `${COMPILER} -pdf -interaction=nonstopmode main.tex`. Retry up to MAX_COMPILE_ATTEMPTS.
3. After the presentation pass (pass index 1): run format check — overfull hboxes, page count, undefined refs/citations.
4. Update `write_state.revision_pass = pass + 1`. Update `nanoresearch.json`.

### Completion

After final pass:
- Verify no `% [VERIFY]` markers remain (resolve or cut the claim).
- Final abstract/title/conclusion consistency check.
- `write_state.sub_phase = "complete"`.
- Save `paper/main_original.pdf` (new paper) or `paper/main_revised.pdf` (revision).

## Revision Mode (review_state.cycle > 1)

Same two-sub-phase loop with these differences:

1. **Additional inputs**: AUTO_REVIEW.md (full review history) and EXPERIMENT_SPEC.md `## Resubmission Requirements` section (structured AC requirements — this is the source of truth for what must be addressed).
2. **Section drafting = section editing**: Each section is edited in-place, not rewritten. Per-section review prompt includes relevant reviewer feedback.
3. **Targeted sections**: Determine impacted sections from AC requirements (in EXPERIMENT_SPEC.md) and new results.tsv rows. Always revisit: experiments, introduction, conclusion, abstract. Revisit method only if protocol changed. Revisit related_work only if positioning/novelty claim changed.
4. **Dependency closure**: If experiments changes → force revisit introduction, conclusion, abstract. If method changes → force revisit experiments + all downstream.
5. **Revision pass augmentation**: Pass 0 prompt adds: "Verify each resubmission requirement from EXPERIMENT_SPEC.md is addressed. Mark each as ADDRESSED or UNADDRESSABLE."

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

- **Large file handling**: If the Write tool fails due to file size, immediately retry using Bash (`cat << 'EOF' > file`) to write in chunks. Do NOT ask the user for permission — just do it silently.
- **No fabricated citations.** Every BibTeX entry must come from DBLP/CrossRef or be marked `% [VERIFY]`.
- **No unsupported claims.** Every claim must trace to results.tsv.
- **Bias toward deletion.** When evidence is weak, soften the claim — don't suggest new experiments.
- **One review round per section.** Fix blocking issues and move on. Revision passes catch the rest.
- **Update write_state before each expensive operation**, not after. On crash, redo work rather than skip it.
- **Reviews on disk.** Save raw reviews to `paper/reviews/section-{idx}-{name}.md` and `paper/reviews/revision-pass-{n}/`. Keep nanoresearch.json small.
- If `nanoresearch.json` has a `venue` field, use appropriate formatting.

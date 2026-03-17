---
description: "Write phase: turn experiment results into a paper or research memo. Use for 'nanoresearch:write', 'write paper', 'draft paper'."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch
---

# Write: Results → Paper (or Research Memo)

Read COMPILER, MAX_COMPILE_ATTEMPTS from the project's **CLAUDE.md**.

## Inputs

Read `results.tsv` (ground truth) and optionally `IDEA.md`. Construct the results narrative directly from `results.tsv`.

If `IDEA.md` does not exist (e.g., `skip-scout` mode), construct context from `EXPERIMENT_SPEC.md` or `autoresearch.md`.

## Mode Detection

Check `nanoresearch.json`:
- If `review_state` exists AND `review_state.cycle > 1` AND `paper/main.tex` exists → **revision mode** (update existing paper).
- If `paper/main.tex` does not exist → **new paper mode** or **memo mode**.
- Default → assess quality gate below.

## Quality Gate (new paper mode only)

**Paper-worthy** (all must hold):
- Best metric shows meaningful improvement over baseline (not noise)
- At least 3 kept experiments
- Improvement attributable to a specific change

**Memo-worthy** (any): null/negative results, fewer than 3 keeps, no clear improvement.

If memo → write `RESEARCH_MEMO.md` and return. (Format: question, hypothesis, approach, results table, why it didn't work, what we learned, next directions.)

## New Paper Mode

### Step 1: Claims-Evidence Matrix

```
| Claim | Evidence (experiment) | Metric delta | Strength |
```

Every claim must map to an accepted experiment in `results.tsv`. No claim without evidence.

### Step 2: Outline

Title, abstract (~150 words), introduction, related work, method, experiments, conclusion. Plan figures/tables.

### Step 3: Write LaTeX

Write `paper/main.tex` + `paper/sections/*.tex` + `paper/references.bib`.

- Search WebSearch for real BibTeX. Never fabricate. Mark unverified with `% [VERIFY]`.
- Use `\citep`/`\citet` (natbib).
- No AI slop: avoid "delve", "pivotal", "landscape", "furthermore".
- Vary sentence openings. Honest limitations.
- If `nanoresearch.json` has a `venue` field, use appropriate formatting.

### Step 4: Figures

Write Python scripts in `paper/figures/` that read `results.tsv`. Generate PDF plots (matplotlib, 300 DPI, serif, colorblind-safe, no titles). Run scripts.

### Step 5: Compile

```bash
cd paper && ${COMPILER} -pdf -interaction=nonstopmode main.tex
```

Fix errors (missing packages, undefined refs, BibTeX syntax). Retry up to MAX_COMPILE_ATTEMPTS. Save `paper/main_original.pdf`.

If compilation fails after all attempts: commit `.tex` files anyway. The review phase reads `.tex` source, not PDF. Note in summary that PDF compilation failed.

## Revision Mode

Triggered when `nanoresearch.json` has `review_state.cycle > 1` and `paper/main.tex` exists.

1. Read `AUTO_REVIEW.md` for area chair's resubmission requirements.
2. Read new entries in `results.tsv` (experiments run after rejection).
3. **Edit existing sections in-place** — do not rewrite from scratch.
4. Add new results to experiments section.
5. Update claims-evidence matrix if new evidence exists.
6. Strengthen or add content addressing each AC requirement.
7. Recompile. Save as `paper/main_revised.pdf`.

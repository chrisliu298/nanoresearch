# nanoresearch

Autonomous ML research distilled to its irreducible core: **scout → loop → write → review**.

The algorithm of research is `propose → evaluate → keep/discard`, applied at three levels: ideas (scout), code (loop), and papers (review). The review cycle simulates venue peer review: 4 independent reviewers, author rebuttal, area chair decision, and bounded reject-revise iteration.

## Constants

### Reviewer

- **REVIEWER_MODEL = `gpt-5.4`** — External model via Codex MCP for brainstorming and review.

### Experiment Loop

- **TIMEOUT_MINUTES = 10** — Kill runs exceeding this.

### Peer Review

- **NUM_REVIEWERS = 4** — 2 Claude subagents + 2 GPT-5.4 xhigh via Codex MCP.
- **MAX_REVIEW_CYCLES = 2** — Full submit-review-rebuttal-decision cycles (initial + 1 resubmission).
- **ACCEPTANCE_THRESHOLD = 6.0** — Average post-rebuttal score >= 6.0 for acceptance.
- **STRONG_REJECT_VETO = 3** — Any reviewer score at or below this is a strong signal to reject (area chair may override only if the reviewer clearly misread the paper).
- **REBUTTAL_EXPERIMENT_BUDGET_MINUTES = 30** — Wall-clock cap for experiments during rebuttal.
- **RESUBMISSION_EXPERIMENT_BUDGET_MINUTES = 60** — Wall-clock cap for experiments after rejection.

### Write Phase

- **NUM_REVISION_PASSES = 2** — Whole-paper revision passes with Prism-style review. Pass 1 catches structural issues, pass 2 catches remaining presentation issues.
- **REVISION_PANEL = 4+4** — 4 Claude write-critic subagents + 4 GPT-5.4 xhigh via relay per revision pass.
- **SECTION_REVIEW_EFFORT = `xhigh`** — Codex effort level for per-section review during drafting.

### Paper

- **COMPILER = `latexmk`**
- **MAX_COMPILE_ATTEMPTS = 3**

### Pipeline

- **EXPERIMENT_BUDGET_HOURS = 4** — Default wall-clock budget for the experiment loop in the full pipeline. Override: `— budget: Nh`.
- **AUTO_PROCEED = true** — Do not block on checkpoints. Never wait for user input.

## State Schema (nanoresearch.json)

Single state file. Updated on every phase transition. No separate state files.

```
topic: string                          # set at creation
status: "in_progress" | "completed" | "failed"
phase: "scout" | "loop" | "write" | "review" | "completed" | "scout_failed"
branch: "nanoresearch/<tag>"               # set at creation
timestamp: ISO 8601                    # updated on every phase transition and resume
idea_summary: string                   # set after scout
best_metric: number                    # set after loop
iteration_count: number               # set after loop
venue: string | null                   # from override, passed to write and review
codex: "on" | "off"                    # from override, default "on"
decision: "accepted" | "rejected" | "memo" | null
write_state: {                         # initialized on phase: "write" entry
  sub_phase: "section_drafting" | "revision" | "complete",
  current_section: number,             # 0-indexed, tracks section_drafting progress
  revision_pass: number,               # 0-indexed, incremented after each completed pass, max NUM_REVISION_PASSES
}
review_state: {                        # initialized on phase: "review" entry
  cycle: number,                       # starts at 1, incremented on resubmission
  sub_phase: "initial_review" | "rebuttal" | "rescoring" | "area_chair" | "decision_gate",
  decision: "accepted" | "rejected" | null,  # per-cycle decision from AC
  codex_threads: {R3?: string, R4?: string},  # present only when Codex used
  scores: {initial: {R1..R4: number}, post_rebuttal: {R1..R4: number}},
  score_history: [{cycle, avg, decision}]
}
```

## File Contracts

| File | Producer | Consumer | Notes |
|------|----------|----------|-------|
| `nanoresearch.json` | orchestrator | all phases | Single state file |
| `IDEA.md` | scout | write | Research plan |
| `EXPERIMENT_SPEC.md` | scout (or review on reject) | loop, write (fallback) | Experiment contract; review replaces (not appends) resubmission requirements |
| `SCOPING_MEMO.md` | scout (failure) | orchestrator | Pipeline halts |
| `results.tsv` | loop | write, review | Columns: #, timestamp, commit, metric, sanity, status, description. Committed at phase boundaries and every 10 loop iterations |
| `autoresearch.md` | loop | loop (resume), write (fallback) | Recovery doc, evolves during loop |
| `paper/main.tex` | write | review | Paper source (reviewers read .tex; PDF is compilation artifact) |
| `paper/reviews/*.md` | write | write (resume) | Per-section and per-pass review artifacts; raw reviewer feedback on disk, not in nanoresearch.json |
| `RESEARCH_MEMO.md` | write (weak results) | orchestrator | Review skipped |
| `AUTO_REVIEW.md` | review | orchestrator, write (revision) | Full review history |

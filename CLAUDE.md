# nanoresearch

Autonomous ML research distilled to its irreducible core: **scout → loop → write → review**.

The algorithm of research is `propose → evaluate → keep/discard`, applied at three levels: ideas (scout), code (loop), and papers (review). The review cycle simulates venue peer review: 4 independent reviewers, author rebuttal, area chair decision, and bounded reject-revise iteration.

## Constants

### Reviewer

- **REVIEWER_MODEL = `gpt-5.4`** — External model via Codex MCP for brainstorming and review. **Must always use `xhigh` reasoning effort. No other effort level is permitted.**

### Experiment Loop

- **TIMEOUT_MINUTES = 10** — Kill runs exceeding this.

### Scout

- **SCOUT_BUDGET_MINUTES = 30** — Wall-clock cap for the entire scout phase. On expiry, write SCOPING_MEMO.md with current progress.

### Peer Review

- **NUM_REVIEWERS = 4** — 2 Claude subagents + 2 GPT-5.4 xhigh via Codex MCP.
- **MAX_REVIEW_CYCLES = 2** — Full submit-review-rebuttal-decision cycles (initial + 1 resubmission).
- **ACCEPTANCE_THRESHOLD = 6.0** — Average post-rebuttal score >= 6.0 for acceptance.
- **STRONG_REJECT_VETO = 3** — Any reviewer score at or below this is a strong signal to reject (area chair may override only if the reviewer clearly misread the paper).
- **REBUTTAL_EXPERIMENT_BUDGET_MINUTES = 30** — Wall-clock cap for experiments during rebuttal.
- **RESUBMISSION_EXPERIMENT_BUDGET_MINUTES = 60** — Wall-clock cap for experiments after rejection.

### Write Phase

- **NUM_REVISION_PASSES = 2** — Whole-paper revision passes with Prism-style review. Pass 0 (structural) catches structural issues, pass 1 (presentation) catches remaining presentation issues.
- **REVISION_PANEL = 4+4** — 4 Claude write-critic subagents + 4 GPT-5.4 xhigh via Codex MCP per revision pass.
- **SECTION_REVIEW_EFFORT = `xhigh`** — Codex effort level for per-section review during drafting. This is the only allowed effort level for all GPT-5.4 usage.

### Paper

- **COMPILER = `latexmk`**
- **MAX_COMPILE_ATTEMPTS = 3**

### Enforced Rule: GPT-5.4 Effort Level

- **All GPT-5.4 calls MUST use `config: {"model_reasoning_effort": "xhigh"}`**. No other effort level (`none`, `low`, `medium`, `high`) is permitted. This applies to every `mcp__codex__codex` call across all phases (scout, write, review). No exceptions, no compromise.

### Pipeline

- **EXPERIMENT_BUDGET_HOURS = 4** — Default wall-clock budget for the experiment loop in the full pipeline. Override: `— budget: Nh`.
- **AUTO_PROCEED = true** — Do not block on checkpoints. Never wait for user input.
- **CODEX_CALL_TIMEOUT_MINUTES = 10** — If a `mcp__codex__codex` call does not return within this time, fall back to a Claude subagent with the same prompt.

### codex: off Mode

When `codex: off`, all Codex MCP calls are replaced with Claude subagents. This eliminates cross-model diversity. Adjust synthesis rules:
- Write revision panel: replace "issues flagged by both model families → must-fix" with "issues flagged by 3+ of 8 reviewers → must-fix".
- Review panel: all 4 reviewers are Claude; use persona diversity as the discriminating signal, not model family.
- Tag each review output with `[Claude-{persona/lens}]` for tracking.

## State Schema (nanoresearch.json)

Single state file. Updated on every phase transition. No separate state files.

```
topic: string                          # set at creation
status: "in_progress" | "completed" | "failed"
phase: "scout" | "loop" | "write" | "review" | "completed" | "scout_failed"
branch: "nanoresearch/<tag>"               # set at creation
timestamp: ISO 8601                    # updated on every phase transition and resume
idea_summary: string                   # set after scout
best_metric: number                    # set after loop and updated after rebuttal experiments; MUST come from keep rows only — never discard/crash rows
iteration_count: number               # set after loop
venue: string | null                   # from override, passed to write and review
codex: "on" | "off"                    # from override, default "on"
pre_loop_stash_ref: string | null      # stash ref for user's dirty worktree; set before loop, cleared after stash pop
decision: "accepted" | "rejected" | "memo" | null
budget_override: string | null         # persisted from override (e.g., "8h"), survives resume
loop_override: number | null           # persisted from override (e.g., 50), survives resume
loop_started_at: number | null          # epoch seconds ($(date +%s)); set on first loop entry, preserved on resume, reset to null before resubmission/rebuttal loops
scout_state: {                         # initialized on phase: "scout" entry; also set for skip-scout/skip-to-write with novelty_confidence: "low"
  sub_phase: "survey" | "ideate" | "specify",
  novelty_confidence: "high" | "low",  # "low" if web search failed, <5 papers found, or scout was skipped
}
write_state: {                         # initialized on phase: "write" entry; reset on fresh entry, preserved on crash resume
  sub_phase: "section_drafting" | "revision" | "complete",
  current_section: number,             # 0-indexed, tracks section_drafting progress
  revision_pass: number,               # 0-indexed, incremented after each completed pass, max NUM_REVISION_PASSES
  impacted_sections: [number] | null,  # revision mode only: indices into SECTION_ORDER to revisit
}
review_state: {                        # initialized on phase: "review" entry
  cycle: number,                       # starts at 1, incremented on resubmission
  sub_phase: "initial_review" | "rebuttal_triage" | "rebuttal_experiments" | "rebuttal_revision" | "rebuttal_response" | "rescoring" | "area_chair" | "decision_gate",
  decision: "accepted" | "rejected" | null,  # per-cycle decision from AC
  codex_threads: {R3?: string, R4?: string},  # present only when Codex used; saved incrementally per reviewer
  reviewer_dispatch: {R1..R4: "claude" | "codex" | "claude-fallback"},  # actual dispatch method per reviewer
  participating_reviewers: ["R1".."R4"],  # subset that succeeded; defines denominator for averaging
  effective_synthesis_mode: "normal" | "codex_off" | null,  # set to "codex_off" if all reviewers are same model family
  scores: {initial: {R1..R4: number}, post_rebuttal: {R1..R4: number}},
  score_history: [{cycle, avg, decision}],  # dedup: at most one entry per cycle
  results_row_at_cycle_start: number | null,  # row count of results.tsv when this review cycle began; set for ALL cycles including cycle 1
  rebuttal_revision_progress: {current_section: number, impacted_sections: [number]} | null,  # tracks section-level progress during rebuttal revision
}
```

## File Contracts

| File | Producer | Consumer | Notes |
|------|----------|----------|-------|
| `nanoresearch.json` | orchestrator | all phases | Single state file |
| `LANDSCAPE.md` | scout | scout (resume), write (context) | Literature survey checkpoint; produced after survey sub-phase |
| `IDEA.md` | scout | write | Research plan |
| `EXPERIMENT_SPEC.md` | scout (or review on reject) | loop, write (fallback) | Experiment contract; review replaces (not appends) resubmission requirements |
| `SCOPING_MEMO.md` | scout (failure) | orchestrator | Pipeline halts |
| `results.tsv` | loop | write, review | Columns: #, timestamp, commit, metric, sanity, status, description. Row #0 (baseline) has `status=keep`. Committed on every keep decision, at phase boundaries, and every 10 loop iterations |
| `autoresearch.md` | loop | loop (resume), write (fallback) | Recovery doc, evolves during loop |
| `paper/main.tex` | write | review | Paper source (reviewers read .tex; PDF is compilation artifact) |
| `paper/reviews/section-{idx}-{name}.md` | write | write (resume) | Per-section review artifacts from drafting |
| `paper/reviews/cycle-{N}/revision-pass-{n}/` | write | write (resume) | Cycle-scoped revision-pass review artifacts; `{model_family}-{lens}.md` |
| `paper/reviews/cycle-{N}/` | review | review (resume), write (revision) | Cycle-scoped review artifacts: `R{K}.md` (initial), `R{K}-rescore.md`, `area-chair.md`. Authoritative for resume |
| `RESEARCH_MEMO.md` | write (weak results) | orchestrator | Review skipped |
| `AUTO_REVIEW.md` | review | orchestrator, write (revision) | Full review history |

# nanoresearch

Autonomous ML research in one command. Scout ideas, run experiments, write a paper, survive peer review — overnight.

```
/nanoresearch "efficient attention mechanisms for long-context LMs"
```

Wake up to an accepted paper or an honest rejection analysis.

## Philosophy

nanoresearch is the irreducible core of [ARIS](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep), rebuilt from first principles following [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and the [nanorepl](https://github.com/chrisliu298/nanorepl) philosophy: strip everything that isn't the core algorithm, then layer complexity back one concept at a time.

The algorithm of research is `propose → evaluate → keep/discard`, applied at three levels:

| Level | Phase | What it does |
|-------|-------|-------------|
| Ideas | **Scout** | Survey literature, generate ideas, verify novelty, produce experiment spec |
| Code | **Loop** | Edit → commit → run → measure → keep or revert (autoresearch) |
| Papers | **Review** | 4 reviewers score → rebuttal → area chair decides → revise if rejected |

## Pipeline

```
                          /nanoresearch "topic"
                                │
                                ▼
                ┌───────────────────────────────┐
                │            SCOUT              │
                │  survey lit, generate ideas,  │
                │  verify novelty, lock spec    │
                └───────────────┬───────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
          ┌────▶│             LOOP              │
          │     │                               │
          │     │    ┌──────────────────────┐   │
          │     │    │  edit → commit → run │   │
          │     │    └───┬─────────────┬────┘   │
          │     │   revert             keep     │
          │     │    │   └──────┬──────┘        │
          │     │    └──────────┘               │
          │     │          budget exceeded?     │
          │     │          yes ──▶ exit         │
          │     └───────────────┬───────────────┘
          │                     │
          │                     ▼
          │     ┌───────────────────────────────┐
          │     │            WRITE              │
          │     │  section-by-section drafting  │
          │     │  + 4+4 panel revision passes  │
          │     │  (or memo if results weak)    │
          │     └───────────────┬───────────────┘
          │                     │
          │                     ▼
          │     ┌───────────────────────────────┐
          │     │           REVIEW              │
          │     │  4 reviewers + rebuttal +     │
          │     │  area chair decision          │
          │     └──────┬────────────────┬───────┘
          │            │                │
          │         rejected         accepted
          │            │                │
          │            ▼                ▼
          │     ┌─────────────┐  ┌─────────────┐
          └─────┤   REVISE    │  │    DONE     │
                │  update spec│  │  paper.pdf  │
                │  & resubmit │  │  + reviews  │
                └─────────────┘  └─────────────┘
```

## Setup

```bash
# 1. Clone the repo
git clone https://github.com/chrisliu298/nanoresearch.git ~/.nanoresearch

# 2. Register the marketplace — add this to ~/.claude/settings.json:
#    "extraKnownMarketplaces": {
#      "nanoresearch": {
#        "source": { "source": "directory", "path": "~/.nanoresearch" }
#      }
#    }

# 3. Install the plugin
claude plugin install nanoresearch@nanoresearch

# Required: Codex MCP (for GPT-5.4 reviewers and brainstorming)
npm install -g @openai/codex
codex setup  # set model to gpt-5.4
claude mcp add codex -s user -- codex mcp-server

# Required for paper writing: LaTeX
# macOS: brew install --cask mactex && brew install poppler
# Ubuntu: sudo apt install texlive-full latexmk poppler-utils
```

## Usage

```bash
# Full pipeline — one command, go to sleep
/nanoresearch "factorized attention for efficient long-context transformers"

# Individual phases
/nanoresearch:scout "topic"                         # just scout
/nanoresearch:loop                                  # just experiment loop
/nanoresearch:write                                 # just paper writing
/nanoresearch:review                                # just peer review

# Override defaults
/nanoresearch "topic" — budget: 8h              # longer experiment budget
/nanoresearch "topic" — loop: 50                # bound to 50 iterations
/nanoresearch "topic" — venue: NeurIPS          # target venue
/nanoresearch "topic" — skip-scout              # skip idea discovery
/nanoresearch "topic" — skip-to-write           # skip to paper writing
```

## Paper Writing

The write phase is a two-sub-phase loop, not a one-shot draft:

**Sub-phase 1: Section-by-section drafting.** Sections are written in dependency order (method → experiments → related work → introduction → conclusion → abstract). Each section gets a GPT-5.4 xhigh review gate before the next begins — catching issues early before they propagate.

**Sub-phase 2: Whole-paper revision passes.** Two passes over the complete paper, each reviewed by a panel of 4 Claude write-critics + 4 GPT-5.4 critics. Pass 1 fixes structural issues (claims-evidence alignment, narrative coherence). Pass 2 fixes presentation issues (de-AI polish, notation, formatting). The panel uses 4 lenses (evidence, structure, positioning, clarity) × 2 model families for blind-spot diversity.

## Peer Review

The review phase simulates a real ML venue review process:

| Role | Model | Focus |
|------|-------|-------|
| Reviewer 1 | Claude | **Methodologist** — technical soundness, proofs, correctness |
| Reviewer 2 | Claude | **Empiricist** — baselines, ablations, statistical rigor |
| Reviewer 3 | GPT-5.4 xhigh | **Novelty Critic** — originality, related work coverage |
| Reviewer 4 | GPT-5.4 xhigh | **Skeptic** — overclaiming, missing limitations |
| Area Chair | Claude | Synthesizes reviews, decides accept/reject |

Two model families (Claude + GPT-5.4) provide genuine blind-spot diversity — not self-play.

After initial reviews, the system writes author rebuttals, runs additional experiments if reviewers request them, and resubmits for rescoring. The area chair makes the final accept/reject decision. If rejected, the paper is revised and resubmitted (up to 2 cycles).

## Architecture

```
nanoresearch/                              9 components
  .claude-plugin/plugin.json
  CLAUDE.md                            constants + state schema + file contracts
  commands/
    nanoresearch.md                    /nanoresearch — full pipeline orchestrator
    scout.md                           /nanoresearch:scout — literature + ideation + spec
    loop.md                            /nanoresearch:loop — autoresearch kernel
    write.md                           /nanoresearch:write — section drafting + revision loop
    review.md                          /nanoresearch:review — peer review cycle
  agents/
    write-critic.md                    parameterized writing critic (4 lenses)
    reviewer.md                        parameterized venue reviewer (4 personas)
    area-chair.md                      meta-review + accept/reject decision
```


## Outputs

Every run produces one of two honest outcomes:

**If results are strong → paper path:**
- `paper/main.pdf` — submission-ready paper
- `AUTO_REVIEW.md` — full review history with scores
- `IDEA.md` — research idea and hypothesis
- `results.tsv` — experiment log

**If results are weak → memo path:**
- `RESEARCH_MEMO.md` — what was tried, why it didn't work, next directions
- `results.tsv` — experiment log

Both are valuable. A memo documenting why something doesn't work saves future researchers from the same dead end.

## Typical Timeline

| Phase | Duration | Can sleep? |
|-------|----------|------------|
| Scout | 15-30 min | No |
| Loop | 1-8 hours | Yes |
| Write | 45-90 min | Yes |
| Review (per cycle) | 30-90 min | Yes |

Run scout interactively, launch the rest before bed, wake up to results.

## Configuration

All constants live in `CLAUDE.md`. Key defaults:

| Constant | Default | What it controls |
|----------|---------|-----------------|
| `EXPERIMENT_BUDGET_HOURS` | 4 | Wall-clock cap for the experiment loop |
| `NUM_REVISION_PASSES` | 2 | Whole-paper revision passes (structural + presentation) |
| `REVISION_PANEL` | 4+4 | Write-critics per revision pass (Claude + GPT-5.4) |
| `MAX_REVIEW_CYCLES` | 2 | Submit-review-rebuttal cycles before stopping |
| `ACCEPTANCE_THRESHOLD` | 6.0 | Average score needed for acceptance |
| `REVIEWER_MODEL` | gpt-5.4 | External model for GPT-5.4 reviewers |
| `TIMEOUT_MINUTES` | 10 | Kill individual experiment runs after this |

## Dependencies

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (required)
- [Codex MCP](https://github.com/openai/codex) (required for GPT-5.4 reviewers)
- LaTeX with `latexmk` (required for paper compilation)
- Git (required for experiment branching)

## Acknowledgments

nanoresearch is a nano reimplementation of [Auto-claude-code-research-in-sleep (ARIS)](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) by [@wanshuiyin](https://github.com/wanshuiyin). ARIS pioneered the idea of autonomous overnight ML research using cross-model collaboration (Claude Code + GPT-5.4). nanoresearch distills ARIS's 25-component architecture down to 8 components while adding simulated peer review.

Also inspired by [autoresearch](https://github.com/karpathy/autoresearch) by Andrej Karpathy.

## License

MIT

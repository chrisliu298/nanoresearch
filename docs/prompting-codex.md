# How to Prompt Codex (GPT-5.4) Effectively

Read this before composing complex relay tasks for Codex. These patterns come from the official GPT-5.4 prompt guidance and improve reliability, completeness, and token efficiency.

## Use XML Tags for Structure

GPT-5.4 responds best to modular, block-structured prompts using XML tags. Wrap each type of guidance in its own tag so the model can parse instructions unambiguously. Use consistent, descriptive tag names.

## Prompt Skeleton: Goal, Context, Constraints, Done-When

Every relay task body benefits from four elements, even if some are brief:

1. **Goal** — one or two sentences stating what you want Codex to do.
2. **Context** — files, folders, errors, or background Codex needs. Point at specific paths rather than asking it to search the whole repo.
3. **Constraints** — standards, conventions, or safety requirements that apply to this task.
4. **Done when** — completion criteria (covered in detail in Completeness Contracts below).

If response shape matters, add an `<output_contract>` as well (see Output Contracts below).

```xml
<goal>
Refactor src/db/pool.py to add connection timeouts and auto-reconnection.
</goal>

<context>
Files:
- src/db/pool.py (main file to change)
- src/db/connection.py (dependency — read but do not modify)
- tests/test_pool.py (test file to update)

The pool currently has no timeout handling. Stale connections cause 502s in production under load.
</context>

<constraints>
- Keep backward compatibility with existing ConnectionPool callers.
- Do not add new dependencies.
- Preserve the existing CLI output format.
</constraints>

<completeness_contract>
All requirements implemented, pytest passes, no new lint errors.
</completeness_contract>
```

For error-driven tasks, include the actual error output in `<context>`:

```xml
<context>
CI failure:
FAILED tests/test_pool.py::test_timeout - ConnectionError: pool exhausted
</context>
```

For simple, well-scoped tasks, a flat version without XML tags is fine:

```
Fix the off-by-one error in src/parser.py line 42. The loop should iterate through the last element. Run pytest tests/test_parser.py to verify.
```

## Output Contracts

Define exactly what you want returned — format, length, structure. This is the single most effective way to improve Codex output quality.

```xml
<output_contract>
- Return exactly the sections requested, in the requested order.
- If a format is required (JSON, Markdown, SQL), output only that format.
- Apply length limits only to the section they are intended for.
</output_contract>
```

**Example:**

```xml
<output_contract>
- Return a markdown list of issues.
- Each item: file path, line number, severity (high/medium/low), one-line description, and suggested fix.
</output_contract>
```

## Completeness Contracts

Define what "done" means explicitly. Codex is more reliable when the prompt specifies the finish line.

```xml
<completeness_contract>
- Treat the task as incomplete until all requested items are covered or explicitly marked [blocked].
- Keep an internal checklist of required deliverables.
- For lists, batches, or paginated results: track processed items and confirm coverage before finalizing.
- If any item is blocked by missing data, mark it [blocked] and state exactly what is missing.
</completeness_contract>
```

## Verification Loops

Ask Codex to check its own work before finalizing. This catches errors that higher reasoning effort alone would not.

```xml
<verification_loop>
Before finalizing:
- Check correctness: does the output satisfy every requirement?
- Check grounding: are claims backed by the provided context or tool outputs?
- Check formatting: does the output match the requested schema or style?
</verification_loop>
```

For programmatically checkable assertions, tell it to write and run a script rather than eyeballing.

For code-change tasks, name the exact verification commands unless they are already documented in `AGENTS.md`:

```xml
<verification_commands>
After making changes:
1. Run tests: pytest tests/test_pool.py -x
2. Run lint: ruff check src/
3. Run type checks: mypy src/
4. Review your own diff for unintended changes.
Only report "done" after all checks pass.
</verification_commands>
```

## Tool Persistence

Tell Codex to persist until the task is fully complete.

```xml
<tool_persistence_rules>
- Use tools whenever they materially improve correctness or completeness.
- Do not stop early when another tool call would improve the result.
- Keep calling tools until the task is complete and verification passes.
- If a tool returns empty or partial results, retry with a different strategy.
</tool_persistence_rules>
```

## Dependency Checks

Codex can skip prerequisite steps when the end state seems obvious. Prevent this.

```xml
<dependency_checks>
- Before taking an action, check whether prerequisite discovery or lookup steps are required.
- Do not skip prerequisite steps just because the intended final action seems obvious.
- If the task depends on the output of a prior step, resolve that dependency first.
</dependency_checks>
```

## Plan Before Complex Tasks

For multi-step or ambiguous tasks, ask Codex to plan before editing. This surfaces wrong assumptions early and prevents wasted effort on the wrong approach.

```xml
<planning>
Before making changes:
1. Read the files listed in the context section, then discover any additional relevant files before editing.
2. State the likely approach or root cause.
3. List the files you expect to change and why.
4. Call out any assumptions or open questions.
5. Then execute the plan and verify.
</planning>
```

For well-scoped tasks (single-file fix, clear instructions), skip planning — it adds overhead without benefit.

## Empty Result Recovery

When a lookup or search returns nothing, don't conclude no results exist.

```xml
<empty_result_recovery>
If a lookup returns empty, partial, or suspiciously narrow results:
- Do not immediately conclude that no results exist.
- Try at least 1-2 fallback strategies (alternate query wording, broader filters, prerequisite lookups, or alternate tools).
- Only report absence after documenting what was attempted.
</empty_result_recovery>
```

## Follow-Through Policy

Clarify when to proceed autonomously vs. stop and report.

```xml
<default_follow_through_policy>
- If the user's intent is clear and the next step is reversible and low-risk, proceed without asking.
- Ask permission only if the next step is irreversible, has external side effects, or requires missing information.
- If proceeding, briefly state what you did and what remains optional.
</default_follow_through_policy>
```

## Parallel vs Sequential Tool Calling

Structure tasks to enable Codex to parallelize independent steps while sequencing dependent ones.

```xml
<parallel_tool_calling>
- When multiple retrieval or lookup steps are independent, prefer parallel tool calls to reduce wall-clock time.
- Do not parallelize steps that have prerequisite dependencies.
- After parallel retrieval, pause to synthesize results before making more calls.
</parallel_tool_calling>
```

## Keep Durable Rules Out of the Prompt

Use the prompt body for task-specific instructions. If a rule applies to most Codex tasks in the repo — build commands, coding conventions, directory layout — put it in `AGENTS.md` instead. Codex is designed to load `AGENTS.md` from the repo root before each task. Repeating durable rules in each prompt adds length without signal.

Reserve the relay prompt body for what changes from task to task: the goal, context, constraints, and completion criteria.

## Prompt Structure

- Use modular sections with clear headers and XML tags
- Keep formatting flat: avoid deeply nested bullets, use `1. 2. 3.` for ordered steps
- Never use nested bullets — keep lists single-level
- If you need hierarchy, split into separate sections

## Reasoning Effort Selection

**Enforced rule: All GPT-5.4 calls in nanoresearch MUST use `xhigh`. No other effort level is permitted. No exceptions.**

Every `mcp__codex__codex` call must include `config: {"model_reasoning_effort": "xhigh"}`. This applies to all phases — scout brainstorming, per-section write reviews, revision panel reviews, and peer review dispatches.

**`mcp__codex__codex-reply` calls** inherit the originating thread's model and effort settings. They are permitted only when the thread was created by an `mcp__codex__codex` call configured with `xhigh`. No explicit effort config is needed on reply calls, but the originating call must have set xhigh.

For reference, the available effort levels in the Codex API are listed below, but only `xhigh` is authorized for use:

| Level | Status in nanoresearch |
|-------|----------------------|
| `none` | **NOT ALLOWED** |
| `low` | **NOT ALLOWED** |
| `medium` | **NOT ALLOWED** |
| `high` | **NOT ALLOWED** |
| `xhigh` | **REQUIRED** — the only permitted effort level for all GPT-5.4 usage |

**Prompt quality still matters.** Improve the prompt first — add an `<output_contract>`, `<completeness_contract>`, or `<verification_loop>` — and always pair it with `xhigh` effort. If the model feels too literal or stops at the first plausible answer, combine a dig-deeper nudge with `xhigh`:

```xml
<dig_deeper_nudge>
- Don't stop at the first plausible answer.
- Look for second-order issues, edge cases, and missing constraints.
- If the task is safety or accuracy critical, perform at least one verification step.
</dig_deeper_nudge>
```

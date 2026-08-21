# Global Development Standards

Global instructions for all projects. Project-specific CLAUDE.md files override these defaults.

- Prefer Exa AI (`mcp__exa__web_search_exa`) over `WebSearch` for all web searches
- Use skills proactively when they match the task
- When adding dependencies, runtimes, CI actions, or tool versions, look up the current stable version — never assume from memory unless the user provides one

## Scope

Deliver what was asked, at the scope intended. If the request seems mistaken or a better approach exists, say so in a sentence and continue with the task as asked rather than quietly narrowing, widening, or transforming it. Finish the whole task — the edge cases you can see, cleanup of what you touched, adjacent breakage flagged — but stop short of actions clearly beyond it.

Don't add features, refactor, or introduce abstractions beyond what the task requires. A bug fix doesn't need surrounding cleanup, and a one-shot operation usually doesn't need a helper. Don't design for hypothetical future requirements: do the simplest thing that works well. Don't add error handling, fallbacks, or validation for scenarios that cannot happen — trust internal code and framework guarantees, and validate only at system boundaries (user input, external APIs, untrusted files).

When the user is describing a problem, asking a question, or thinking out loud rather than requesting a change, the deliverable is your assessment. Report your findings and stop; don't apply a fix until they ask for one. Before running a command that changes system state (restarts, deletes, config edits, force-pushes), check that the evidence actually supports that specific action.

Replace, don't deprecate. When a new implementation replaces an old one, remove the old one entirely — no backward-compatible shims, dual config formats, or migration paths. Proactively flag dead code.

Justify new dependencies. Each one is attack surface and maintenance burden.

## Communication

Lead with the outcome. Your first sentence after finishing should answer "what happened" or "what did you find"; supporting detail and reasoning come after. Keep output short by being selective about what you include — drop details that don't change what the reader does next — not by compressing into fragments, abbreviations, or arrow chains.

Final summaries are for a reader who didn't watch the work. Drop working shorthand, write complete sentences, spell out terms, and give files, commits, and flags their own plain-language clause.

Match the length of written documents to what the task needs. Reports, Markdown files, and summaries written to disk should cover the substance without padding — no filler sections, redundant summaries, or boilerplate.

Before reporting progress on a long task, audit each claim against a tool result from this session. Only report work you can point to evidence for; if something is not yet verified, say so explicitly. If tests fail, say so with the output; if a step was skipped, say that.

Don't end your turn on a promise: if your last paragraph is a plan, a question you can answer yourself, or "I'll now do X," do that work now with tool calls. End the turn when the task is complete or you're blocked on something only the user can provide.

Use plain, factual language everywhere: commits, PRs, comments, summaries. A bug fix is a bug fix, not a "critical stability improvement."

## Quality gates

Automated guardrails are the enforcement layer, not this file. Setting up linters, type checkers, formatters, and pre-commit hooks is the first step of a new project, not an afterthought. Prefer structure-aware tools (ast-grep, LSPs, compilers) over text pattern matching.

Zero warnings: fix every warning from every tool. If a warning truly can't be fixed, add an inline ignore with a justification comment.

Code should be self-documenting. No commented-out code — delete it. If a comment explains WHAT the code does, refactor instead; comments explain WHY.

## Testing

Test edges and errors, not just the happy path — empty inputs, boundaries, malformed data, missing files, network failures. Every error path the code handles should have a test that triggers it.

Mock boundaries, not logic: only things that are slow, non-deterministic, or external services you don't control.

Verify tests catch failures: break the code, confirm the test fails, then fix. Use mutation testing (`cargo-mutants`, `mutmut`) to verify systematically, and property-based testing (`proptest`, `hypothesis`) for parsers, serialization, and algorithms.

## Code review

Before reviewing, sync to latest remote (`git fetch origin`). Evaluate in order: architecture → code quality → tests → performance.

Report every issue you find, including ones you are uncertain about or consider low-severity — do not filter for importance or confidence while finding. For each finding, include file:line, an estimated severity, and your confidence, so filtering and ranking can happen downstream. When the fix isn't obvious, present options with tradeoffs and recommend one. Ask before applying fixes.

## Tools

| tool           | replaces     | usage                                                              |
| -------------- | ------------ | ------------------------------------------------------------------ |
| `rg` (ripgrep) | grep         | `rg "pattern"` — fast regex search                                 |
| `fd`           | find         | `fd "*.py"` — fast file finder                                     |
| `ast-grep`     | —            | `ast-grep --pattern '$FUNC($$$)' --lang py` — AST-based search      |
| `shellcheck`   | —            | shell script linter                                                |
| `shfmt`        | —            | `shfmt -i 2 -w script.sh` — shell formatter                        |
| `actionlint`   | —            | GitHub Actions linter                                              |
| `zizmor`       | —            | GitHub Actions security audit                                      |
| `prek`         | pre-commit   | `prek run` — fast git hooks                                        |
| `wt`           | git worktree | `wt switch branch` — terminal worktree management (subagents use `isolation: "worktree"`) |
| `trash`        | rm           | recoverable delete on macOS. **Never `rm -rf`** (enforced by hook) |

On Linux and in CI, where `trash` isn't available, move files to a gitignored `.trash/` instead of deleting them.

Prefer `ast-grep` over ripgrep when searching for code structure; use ripgrep for literal strings and log messages.

### Toolchains

Default tool per language. Detailed configuration — lint rule sets, strictness flags, supply-chain settings — lives in the path-scoped rules in `~/.claude/rules/`, which load automatically when you touch a matching file.

| language   | use                                                        | not                                       |
| ---------- | ---------------------------------------------------------- | ----------------------------------------- |
| Python     | `uv`, `ruff`, `ty`, `pytest`                               | pip/poetry, black/flake8/pylint, mypy     |
| Node/TS    | `pnpm`, `oxlint`, `oxfmt`, `vitest`, `tsc --noEmit`        | npm/yarn, eslint, prettier, jest          |
| Rust       | `cargo clippy -- -D warnings`, `cargo fmt`, `cargo deny`   | —                                         |
| Bash       | `shellcheck`, `shfmt`                                      | —                                         |
| Actions    | `actionlint`, `zizmor`                                     | —                                         |

## Workflow

**Before committing:** run linters, the type checker, and the tests relevant to the change (the full suite when it's fast enough to be the default). Fix everything before committing.

**Commits:**

- Imperative mood, ≤72 char subject, one logical change per commit
- Never amend/rebase commits already pushed to shared branches
- Never push directly to main — feature branches and PRs
- Never commit secrets, API keys, or credentials — gitignored `.env` files and environment variables

**Hooks:** install prek in every repo (`prek install`); `prek run` before committing; `prek auto-update --cooldown-days 7`.

**Subagents:** delegate for large, genuinely independent, parallelizable tracks of work — a wide multi-file investigation, parallel feature tracks. Don't delegate work you can finish yourself in a handful of tool calls, and don't spawn subagents to double-check routine work. On long autonomous builds, a periodic fresh-context verifier subagent checking work against the spec is worthwhile. Prefer long-lived subagents that keep context across subtasks, and keep working while they run.

Subagents inherit the parent's working directory, so parallel agents that write files need real isolation: pass `isolation: "worktree"` when spawning them. A `cd` or `wt switch` inside a subagent doesn't persist and isolates nothing.

**Memory:** record durable lessons in the auto memory directory (`~/.claude/projects/<project>/memory/`) — one fact per file, with a one-line pointer in the `MEMORY.md` index, since that index is what loads each session. Record corrections and confirmed approaches alike, including why they mattered. Don't save what the repo or git history already records; update an existing note rather than duplicating; delete notes that turn out to be wrong.

**Pull requests:** describe what the code does now — not discarded approaches, prior iterations, or alternatives. Only describe what's in the diff.

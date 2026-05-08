# Contributing to compress-agent-sessions

This is a **single-script macOS utility**. Scope is frozen to the current 5
subcommands (`estimate`, `compress`, `list`, `stats`, `restore`) and the
existing 3 backends (`applesauce`, `afsctool`, `ditto`). PRs that add new
subcommands, new compression backends, or non-macOS platform support **will
not be merged**. Fork instead — that's what forks are for.

Bug-fix PRs are welcome under a few conditions documented below.

## Hard-close labels

Three labels are used to close out-of-scope or low-quality PRs and issues:

| Label | When applied |
| --- | --- |
| `wontfix:platform` | Linux/Windows port, or any code that adds non-macOS support. Also applied to CVE reports for the chflags-on-decmpfs zero-byte truncation, which is documented Apple kernel behavior, not a vulnerability in this tool. |
| `wontfix:scope` | New subcommand, new backend, or any feature that expands the frozen scope. |
| `wontfix:ai-generated` | Boilerplate or speculative changes without clear human intent — typically PRs that touch many files for cosmetic reasons. |

Hard-closes are not personal. They protect the maintainer's time so the tool
can stay alive and minimal. Forks are encouraged.

## Bug-fix PR requirements

1. **Linked issue.** No issue, no review. Open the issue first.
2. **Smoke-test assertion.** If your fix changes observable behavior, add an
   assertion to `tests/smoke.zsh`. The test file is your contract.
3. **`zsh tests/smoke.zsh` passes locally.** All 24 assertions must remain
   green. Tests are local-only — there is no CI by design.
4. **CHANGELOG entry.** Prepend a new `## [X.Y.Z] — YYYY-MM-DD` heading with
   `### Fixed` describing the bug and the fix.
5. **No scope expansion.** Re-read the Scope section above before opening.

## Local development

This is a single zsh script with one test file. There is no build step.

    git clone https://github.com/ariccio/compress-agent-sessions
    cd compress-agent-sessions
    zsh tests/smoke.zsh

The smoke test creates a synthetic `$TMPDIR` fixture, exercises every
subcommand and defensive check, and asserts a 5×+ compression ratio on the
synthetic JSONL payload. It does not touch real user data.

## What "best-effort" means

The maintainer will review PRs and respond to bugs when time permits. There
is no SLA. The tool is intentionally feature-frozen — the goal is for it to
keep working without ongoing maintenance, not to grow.

## What about feature ideas?

Use the [Discussions section](https://github.com/ariccio/compress-agent-sessions/discussions)
if it's enabled. Otherwise, fork. The maintainer's bandwidth is finite and
goes to bug-fix triage, not feature gardening.

<!-- Thanks for contributing. Please confirm each item below before opening this PR. -->

## What & why

<!-- One short paragraph. What does this PR change, and why? -->

## Linked issue

Fixes #

<!-- PRs without a linked issue (or a clear bug-fix justification) will not be reviewed. -->

## Checklist

- [ ] I ran `zsh tests/smoke.zsh` and it passed (24 assertions).
- [ ] I added a smoke-test assertion for the change (if it affects behavior).
- [ ] No new subcommands, compression backends, or platform support added.
- [ ] No AI-generated boilerplate or speculative features.
- [ ] macOS-only assumption preserved (no Linux/Windows code paths).
- [ ] CHANGELOG entry added under a new `## [X.Y.Z]` heading at the top.
- [ ] No `chflags` calls anywhere (this is a hard invariant — see source banner).

## Scope acknowledgment

- [ ] I have read the [Scope](https://github.com/ariccio/compress-agent-sessions#scope) section and confirm this PR fits within the frozen scope.

<!--
PRs that fail the scope check will be closed with one of:
- wontfix:platform   — Linux/Windows port or related code
- wontfix:scope      — new subcommand, backend, or feature outside frozen scope
- wontfix:ai-generated — boilerplate or speculative changes without clear human intent
-->

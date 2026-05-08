# Security policy

## Reporting a vulnerability

Please use [GitHub's private vulnerability reporting](https://github.com/ariccio/compress-agent-sessions/security/advisories/new)
rather than opening a public issue. This lets the maintainer triage and
patch without exposing users to the vulnerability before a fix is available.

## What is and isn't a vulnerability

### Not a vulnerability — `chflags(1)`/`chflags(2)` truncation on decmpfs files

The behavior documented in the README's
[chflags truncation](https://github.com/ariccio/compress-agent-sessions#chflags-truncation--known-macos-kernel-behavior)
section and in the source-file safety banner — where calling `chflags` with
any flag list on a decmpfs-compressed file silently truncates the file to
zero bytes — is **documented Apple kernel behavior**, not a vulnerability
in this tool.

This script never calls `chflags` on a compressed file. The runtime safety
banner exists to warn future contributors and operators who pipe the output
into other tooling.

**Reports of this behavior as a CVE will be closed with `wontfix:platform`.**

If you need to mutate flags on a compressed file, decompress first via
`applesauce decompress` or `compress-agent-sessions restore <path>`, then
mutate flags, then re-compress if desired.

### Genuinely security-relevant cases

- A path that bypasses the `--allow-system-paths` denylist
- A way for `compress` to corrupt user data despite the existing safety gates
  (open-handle check, mtime grace, age threshold, idempotency)
- A way for `restore` to bypass the `--confirm-destructive` gate
- A way for the script to escalate privileges or write outside the targeted
  paths
- Any unsafe shell-injection pattern in argument handling

These should be reported privately. The maintainer will respond within a
best-effort timeframe; there is no SLA.

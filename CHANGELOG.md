# Changelog

All notable changes to `compress-agent-sessions` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] — 2026-05-07

Defensive pass before public OSS release. Hardens first-run UX across non-Darwin hosts, malformed environments, and dangerous-path arguments without expanding scope.

### Added
- `--confirm-destructive` flag (parser + usage). Required when `restore` scope holds >100 files with the decmpfs flag set. Compress is NOT gated (compression is reversible). Gate counts LIVE decmpfs flags, not raw file count, and applies to `restore --dry-run` as well as real restore.
- `--allow-system-paths` flag (parser + usage). Bypasses the new system-path denylist. Operator takes ownership of consequences.
- `COMPRESSION_TOOL_BIN` environment override (default `/usr/bin/compression_tool`). Documented in `--help`. Enables both portable test injection and explicit non-default macOS installs.
- Three-stage decmpfs availability gate: `uname -s == Darwin` → `compression_tool` executable → tmpfile compression probe. Catches Asahi Linux and Docker-on-Mac which lie about Darwin-ness but lack a working decmpfs userspace.
- System-path denylist (canonical-prefix-with-trailing-slash, boundary-checked): refuses `/`, `/System/`, `/usr/`, `/bin/`, `/sbin/`, `/etc/`, `/Library/`. Does NOT include `/Volumes/` (legitimate user external storage) or `/private/` (canonical resolution of `/tmp`). Substring matching deliberately rejected — `/Users/alex/Library/...` must not falsely match `/Library`.

### Fixed
- `LEDGER_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/$SCRIPT_NAME"` no longer expands at script-load. On a non-Darwin host without `$HOME`, `set -u` would crash before the platform diagnostic could fire. `LEDGER_DIR` is now lazy-initialized in `main()` after the platform check passes.
- chflags banner copy now closes a CVE-bait vector by adding the sentence "This is documented Apple kernel behavior, not a vulnerability in this tool." to both the source-file comment block and the runtime stderr banner. Same caveat will appear in README troubleshooting + SECURITY.md (OSS metadata).
- `cmd_restore` now calls `build_open_handle_set` before the restore loop, adding the same open-handle protection that `cmd_compress` has always had. The `_restore_via_rewrite` cat-rewrite fallback can corrupt a live session file held open for writing; files present in the lsof open-handle set are skipped with a `skip:open-handle` warning (`log_warn`) and counted in the summary line (printed only when non-zero). Note: compress surfaces open-handle skips through the structured `skip_counts` breakdown; restore surfaces them via a separate conditional `log_warn` line — the observable summary format differs. Applies to all restore tool paths (applesauce, afsctool, rewrite).
- All five tool dispatchers (`_compress_with_applesauce`, `_compress_with_afsctool`, `_compress_with_ditto`, `_restore_with_applesauce`, `_restore_with_afsctool`) previously discarded tool stderr unconditionally via `2>/dev/null` / `>/dev/null 2>&1`. On failure, the operator saw only a generic `compress/restore failed: <file>` warning with no cause. A shared `_run_tool_capturing_stderr` helper now captures stderr to a tmpfile; the dispatcher surfaces it via `log_warn` on non-zero exit and discards it silently on success. When the tool exits non-zero but produces no stderr (e.g. signal kill, exit 137/139), a synthetic `(no stderr captured; exit=N — possible signal kill)` message is emitted instead of silent failure. Success-path output remains quiet.

#### Iter-3 hardening (post-review)
- `_compress_with_ditto`: `mv -f` rename step after a successful ditto compress now emits `log_warn` on failure, distinguishing the rename failure from the compression failure and confirming the original file is intact. Previously the caller saw only the generic `compress failed: <file>` message with no indication that ditto succeeded and only the atomic rename failed.
- `_run_tool_capturing_stderr`: mktemp failure now returns sentinel exit code 251 (`_CAS_RC_MKTEMP_FAILED`) instead of 1. All five dispatchers check for this sentinel and suppress the misleading `(no stderr captured; exit=N — possible signal kill)` message when the real cause is already named by the helper's `log_error`. Previously a full `/tmp` caused every compress/restore to emit a confusing signal-kill warning in addition to the root-cause log_error.

#### Iter-2 hardening (post-review)
- `_compress_with_ditto` now uses `_run_tool_capturing_stderr` instead of `>/dev/null 2>&1`, making it the fifth (and final) dispatcher to capture tool stderr on failure. Previously ditto failures were fully silent.
- `_run_tool_capturing_stderr`: `mktemp` failure now emits `log_error` naming `/tmp` exhaustion as the likely cause before returning 1. Previously a disk-full `/tmp` caused every subsequent compress/restore to show `compress/restore failed: <file>` with no root-cause hint.
- `_run_tool_capturing_stderr`: named variable is now pre-initialized to empty string before `mktemp`, ensuring callers always have a defined (if empty) `_stmp` even on mktemp failure — eliminates the latent `rm -f ""` portability hole.
- All five dispatchers: when a tool exits non-zero but writes nothing to stderr (e.g. SIGKILL/exit 137, SIGSEGV/exit 139), a synthetic `(no stderr captured; exit=N — possible signal kill)` message is now emitted so signal-killed tools are not silently swallowed.
- `_run_tool_capturing_stderr` comment: fixed two inaccuracies — removed the wrong "via REPLY" claim (the mechanism is a named variable, not `REPLY`); corrected the usage line from `_run_capture_stderr` to `_run_tool_capturing_stderr`; clarified that stderr is captured to a tmpfile that the caller (not the helper) is responsible for removing.

### Changed (observable)
- `--help` output now lists `--confirm-destructive`, `--allow-system-paths`, and an "Environment overrides" section (`COMPRESSION_TOOL_BIN`, `XDG_STATE_HOME`).
- Non-Darwin invocation now exits with a structured diagnostic (kernel name, footgun explanation, link to README Limitations) instead of silently failing late or crashing in `set -u`.
- `cmd_estimate` failure when all samples fail now reports the configured `COMPRESSION_TOOL_BIN`, candidate count, and override syntax — replaces the older "is /usr/bin/compression_tool present?" copy.

### Test coverage
- `tests/smoke.zsh` extended from 11 → 24 assertions. New cases:
  - `--help` advertises new flags and env overrides.
  - Non-Darwin platform refusal via fake-`uname` executable on `PATH` (PATH-shadowed shell functions do not survive shebang re-exec, so a fake executable is required).
  - `--help` still works under fake-Linux `uname` (early-return ordering regression guard).
  - Missing `compression_tool` via `COMPRESSION_TOOL_BIN=/nonexistent` env override (PATH manipulation alone is defeated by the script's `PATH` append at line 7).
  - System-path denylist positive (`/Library` refused) AND negative (`/Users/test/Library` accepted) — false-positive guard for substring matching.
  - `--allow-system-paths` bypass.
  - `restore` gated without `--confirm-destructive` when fixture holds 101 decmpfs-flagged files.
  - `restore --dry-run` respects gate (no mutation but still gated).
  - `restore` proceeds with `--confirm-destructive`.
  - Cross-coupling guard: `estimate`, `list`, `stats` all succeed end-to-end on read-only paths after defensive additions.

### Internal
- New helpers: `require_decmpfs_capable_host`, `require_path_not_system`, `require_no_system_paths_in_targets`. Wired into `main()` after the `--help`/`--version` early-return and before subcommand dispatch.
- `cmd_restore` now performs a pre-pass scan for live decmpfs-flagged files, capped at 101 for early exit, before any mutation OR dry-run log path.
- All new diagnostics follow the existing `log_error` shape and the repo's diagnostic-richness rule.

## [0.2.0] — 2026-04-27

`zsh-native-classifier` performance refactor (Approach C): pushes age, recent-mtime, and current-codex-month filters into BSD `find` itself; replaces 43k per-file `lsof` invocations with one bulk system-wide `lsof -nP -w -F n` call; replaces ~130k per-file `stat` calls with `find -print0 \| xargs -0 stat` batches; replaces per-file `date` forks with `zsh/datetime` builtins (`$EPOCHSECONDS` and `strftime`).

### Performance
- `estimate` on real-world 18.3 GB / 43,593 files: **2 h 46 m → 2 m 17 s (~72× speedup)**.
- Real-data first-pilot run (this version): 18.2 GB compressed to 5.5 GB across 43,562 files (12.8 GB / 70% reclaimed) in ~17.5 minutes wall-clock.

### Safety
- New non-suppressible stderr banner on every `compress` and `restore` invocation about the macOS `chflags`-on-decmpfs-truncates-to-0-bytes data-loss footgun. Source-code guardrail comment block added above all flag-mutating dispatchers. Verified empirically that this destruction is silent and unrecoverable from the live filesystem.
- Bulk `lsof` failure now aborts hard with a clear error rather than silently shipping without open-handle protection.
- Both candidate paths AND lsof-emitted held-open paths are `realpath`-canonicalized via batched `xargs realpath` before set lookup, so files held open via symlinked paths are correctly detected.

### Changed (observable behavior)
- `list` / `compress` summaries gained a `pruned-by-find: N (too-young=N, recent-mtime=N, current-month=N)` line. Files matching these reasons are now eliminated by `find` predicates rather than per-row `skip:` annotations, which would have lost the count.
- The smoke-test assertion that previously checked per-row `skip:too-young` output now checks the new `pruned-by-find` summary line (existing-assertion REPLACE per the test plan).

### Internal
- `discover_candidate_files_print0` — single `find` per scope root with `-mtime +(AGE_DAYS-1)`, `! -mmin -ACTIVE_GRACE_MINUTES`, and `-prune` for the current Codex month directory.
- `count_pruned_by_reason` — separate `find` invocations per reason for operator-visible pruning summary.
- `build_open_handle_set` — system-wide `lsof -nP -w -F n` parsed via `awk '/^n\//{print substr($0,2)}'`, batched-`xargs realpath` canonicalized, populated into `OPEN_HANDLES` associative array.
- `build_metadata_cache` — batched `stat -f '%z|%m|%Sf|%N'` via `printf '%s\0' | xargs -0`.
- `build_candidate_canonical_cache` — batched `xargs realpath` per user choice (Approach C: realpath both sides for 100%-correct symlink coverage).
- `classify_file` reduced to two O(1) hash lookups: open-handle set + decmpfs-flag substring match. All other reasons handled at the `find` level.
- Loaded zsh modules: `zsh/datetime` for `$EPOCHSECONDS` and `strftime` builtin (no fork).
- `setopt extended_glob` for negated glob qualifier support.

### Tactical corrections caught during research (would have shipped as bugs without research subagents)
- `find -mtime +14` ≡ ≥15 days, not ≥14 (BSD rounds up). Use `+13` for the "older than 14 days" intent.
- `find -newermt 'X minutes ago'` is undocumented on macOS BSD `find`. Use `! -mmin -N` instead.
- `find -path PATTERN -prune -o ...` skips descent into excluded subtrees; `! -path PATTERN` does not.
- `find ... -print0` and `find ... -exec ... {} +` semantically conflict (mixed output streams) — pick one.
- `lsof` argv list hits ARG_MAX hard fail at ~6500 paths. System-wide enumeration is constant-time and TCC-safer (lsof only reads kernel FD tables, doesn't open files).
- `lsof -F n` outputs many non-path `n` lines. Filter with `^n/`, not just `^n`.
- `lsof` reports symlink-resolved canonical paths. Both sides of the held-open lookup must be `realpath`-normalized.
- zsh `${(%):-%D{%Y/%m}}` has a literal-`}` leak. Use `strftime "%Y/%m" $EPOCHSECONDS` instead.
- zsh `(^pat)` negated glob requires `setopt extended_glob`.
- zsh `local <name>` (no `=value`) PRINTS the variable's current value if the name is already in scope — caused a `n='       0'` debug leak from a `for` loop body. Hoist `local` declarations outside loops.

### Documentation
- New skill `.claude/skills/shell-cli-tool-research-protocol/` documenting the four-phase pattern (parallel design exploration → tool-grounding research subagents → dedicated test-design pass → implementation) used to design this refactor.
- New reference memory `reference_chflags_truncates_decmpfs_files` recording the empirical chflags-on-decmpfs data-loss behavior.

## [0.1.0] — 2026-04-24

First release.

### Added
- `estimate` subcommand — sample-based savings predictor using macOS
  `compression_tool`; reports candidate count, total bytes, mean ratio,
  and extrapolated savings.
- `compress` subcommand — dispatch to `applesauce` / `afsctool` / `ditto`
  with auto-detection preference (applesauce > afsctool > ditto).
  Captures before/after apparent + physical bytes and writes a run
  entry to the ledger. Supports `--dry-run` and SIGINT flushing.
- `list` subcommand — shows all files in scope with per-file
  classification (candidate / skip:too-young / skip:excluded-glob /
  skip:recent-mtime / skip:open-handle / skip:current-month /
  skip:already-compressed).
- `stats` subcommand — reads the ledger and prints cumulative totals
  (first_run, last_run, total_runs, total_files_compressed,
  total_bytes_saved) plus the 10 newest per-run JSON entries.
- `restore` subcommand — decompresses files via `applesauce decompress` /
  `afsctool -d` / cat-rewrite fallback. Verifies the `UF_COMPRESSED`
  flag is cleared.
- Active-session protection: mtime grace window, `lsof` open-handle
  check, Codex-current-month exclusion, `UF_COMPRESSED` idempotency
  check.
- Ledger I/O: per-run JSON + cumulative KV; atomic write via
  tmp+rename; `$XDG_STATE_HOME`-aware default location.
- CLI: `--help`, `--version`, `--age-days`, `--tool`, `--algorithm`,
  `--level`, `--include` (claude|codex|gemini|all, repeatable),
  `--exclude GLOB` (repeatable), `--active-grace-minutes`,
  `--sample-fraction`, `--ledger-dir`, `--dry-run`, `--quiet`,
  `--verbose`.
- `tests/smoke.zsh` — 11-assertion synthetic-fixture E2E that
  exercises estimate / compress / idempotent-skip / restore /
  stats / exclude / list.
- `docs/launchd-weekly-example.plist` — launchd template for weekly
  automated runs.

### Compression detection
- Uses the `UF_COMPRESSED` inode flag (surfaced via `stat -f '%Sf'`)
  rather than the `com.apple.decmpfs` xattr, since modern `applesauce`
  stores its xattr in a form that isn't user-visible via `xattr -l`.

### Notes
- macOS-only. Uses APFS/HFS+ decmpfs mechanism.
- Single-file zsh script; no runtime dependencies beyond `applesauce`
  or `afsctool`. `ditto` is always available as a last-resort fallback
  but is unreliable on APFS.

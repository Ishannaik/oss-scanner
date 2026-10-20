# Threat model

## What this project does and where untrusted input enters
- agentsweep is a local, offline CLI (`agentsweep` / `asweep`). It finds secrets in the history files of AI coding agents (Claude Code, Codex, Cursor, Aider, OpenCode, VS Code extensions and others) and redacts them in place.
- The input is the content of those history files: JSONL, JSON and SQLite databases under the user's home directory. Treat that content as untrusted. It holds text from LLM output, web pages, tool results and pasted files, so an attacker can plant arbitrary bytes in it.
- Source discovery lives in `src/agentsweep/sources/`. Detection rules (many ported from gitleaks) live in `src/agentsweep/scanner.py`, with an optional RE2 engine in `regex_engine.py`. Writes go through `safe_write()` in `src/agentsweep/redactor.py`.
- The only network call is the optional PyPI version check in `cli.py` (a fixed https URL). Any other outbound request is a bug.

## Security goals
- A secret that a rule matches must not survive a redaction run, in the main file or in a SQLite `-wal` / `-shm` sidecar.
- agentsweep must never print, log or write a raw secret value outside the original file and its `.bak` backup.
- Writes must stay inside the source's root. `safe_write()` refuses symlinks, paths outside the root, and files changed since the scan. It writes a `0o600` `.bak` with `O_EXCL` before replacing a file.
- A crafted history file must not corrupt other data, escape the root, or make agentsweep write a file it did not scan.

## Components that matter most / least
- Most: `redactor.py` (`safe_write`, backups, validation after write), `scanner.py` and `regex_engine.py` (missed detections, catastrophic backtracking), the SQLite handling in `sources/_helpers.py` and `sources/_more.py`.
- Less: the interactive UI under `src/agentsweep/ui/`, the banner and tips. `scripts/` is maintainer tooling and is out of scope.

## How to exercise it
- `python -m pytest tests/ -q` runs the suite. `tests/fixtures/` holds synthetic, secret-shaped data.
- `agentsweep --help` lists commands. Point a scan at a fixture copy, never at a real home directory.

## How you rate severity
- Critical: a redaction run that reports success but leaves a matched secret on disk, or that writes a raw secret to a new location (log, stdout, temp file left behind).
- High: writing outside the source root, following a symlink, or destroying a history file without a backup. Any new network call.
- Medium: a regex that a crafted line can drive into catastrophic backtracking (DoS of a scan).
- Low: a missed detection for a single rule with an unusual secret format.

## Anything to leave alone
- Do not report the PyPI version check itself. It is documented and can be turned off with `AGENTSWEEP_NO_UPDATE=1`.
- Test fixtures contain fake secrets on purpose.

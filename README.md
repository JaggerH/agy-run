# agy-run

Headless dispatch for **Antigravity CLI (`agy`)** — use a cheap Gemini model as a subagent from
any orchestrator: Claude Code, Codex, CI, or plain shell.

```bash
agy-run -C ~/projects/myrepo "Find every file that mentions FeedUnit and list them."
```

One ~80-line bash script. No server, no MCP bridge, no extra auth — it drives the `agy` binary
you already have, with your existing Antigravity subscription and models.

## Why this exists

`agy --print` looks like a ready-made headless mode, but four defects break real orchestration
(all reproduced on agy 1.0.14):

| # | Defect | Fix in agy-run |
|---|--------|----------------|
| 1 | [issue #76](https://github.com/google-antigravity/antigravity-cli/issues/76): `--print` **silently drops stdout** when stdout is not a TTY (pipes, subprocesses — i.e. every orchestrator) | wraps the call in `script -qec` (pseudo-TTY), strips the CRLF it introduces |
| 2 | `--print` runs in agy's **own scratch workspace**, not your cwd — commands execute against the wrong directory/repo | binds the target dir with `--add-dir` (`-C` flag, defaults to cwd) |
| 3 | Weak models **fabricate command output** instead of running commands | every prompt gets a discipline preamble: paste real output verbatim, never guess |
| 4 | Usage limit on the cheap model kills the run | detects quota/rate-limit errors, retries once on a fallback model, loudly labeled |

## Permission model

In print mode agy **auto-approves every tool call** — including `rm` (verified empirically).
`agy-run` assumes a deny-list in `~/.gemini/antigravity-cli/settings.json`:

```json
{
  "permissions": {
    "deny": [
      "command(rm)",
      "command(git push)"
    ]
  }
}
```

Deny rules ARE enforced in print mode (verified). A denied command surfaces in the reply as:

```
DENIED: Permission denied for command(rm /tmp/x). Matches user-configured deny rule.
```

…which the preamble instructs the worker to report verbatim instead of working around. Your
orchestrator (e.g. Claude Code) then escalates: it re-runs the command itself under **its** own
user-facing approval flow. Net effect: everything is auto-approved except the destructive pair,
which becomes "ask the human upstream" — without any interactive channel into the subprocess.

Adjust the deny list to taste (`command(sudo)`, `command(docker rm)`, …).

## Install

```bash
cp agy-run ~/.local/bin/ && chmod +x ~/.local/bin/agy-run
```

Requirements: `agy` (authenticated), `script` (util-linux, preinstalled on Linux; on macOS the
BSD `script` has different flags — see Caveats), `timeout` (coreutils).

## Usage

```bash
agy-run [-C workdir] [-m model] [-f fallback_model] [-t timeout_seconds] "<prompt>"
```

| Flag | Default | Meaning |
|------|---------|---------|
| `-C` | `$PWD` | Directory the worker operates in (bound via `--add-dir`) |
| `-m` | `Gemini 3.5 Flash (Low)` | Primary model (see `agy models`) |
| `-f` | `Claude Sonnet 4.6 (Thinking)` | Fallback on usage limit |
| `-t` | `300` | Hard timeout in seconds (a ceiling — returns as soon as agy finishes) |

Output streams through in real time (`tee`), so orchestrators that poll a background process
see progress. Note `agy --print` itself only emits the final answer — there is no token-level
streaming to pass through.

### From Claude Code

Run it with the Bash tool; for long tasks use a background run and get notified on completion.
An installable skill ships in [`skills/agy-run/`](skills/agy-run/) — copy it to
`~/.claude/skills/agy-run/` and Claude picks the pattern up automatically. Suggested permission
allowlist entry so dispatch never prompts: `Bash(agy-run:*)`.

### From Codex / anything else

It's a plain command that prints the answer on stdout and exits non-zero on failure. Call it
from AGENTS.md-driven workflows, Makefiles, CI — anywhere.

## Caveats

- **Fallback trigger is text-matched** (`quota|rate limit|429|…`) against the worker's output;
  a task whose *content* legitimately contains those words could false-positive the retry.
- macOS: BSD `script` uses `script -q /dev/null command...` syntax; adapt `run_once`.
- The discipline preamble reduces but cannot eliminate fabricated output from small models —
  for load-bearing results, have the orchestrator spot-check (e.g. ask for `git rev-parse HEAD`
  and compare).
- Deny rules match the command head (`command(rm)` blocks any `rm …` invocation). They are
  enforced by agy, not by this script — keep them in agy's settings, not here.

## License

MIT

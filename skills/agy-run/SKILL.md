---
name: agy-run
description: Use when delegating text-heavy or mechanical work (bulk search/read/summarize across many files, consistency sweeps, boilerplate edits, doc audits) to a cheap Gemini worker via the local `agy-run` command, instead of doing it with the expensive main model — or when the user says "让 agy 做" / "用 agy" / "dispatch to gemini". Covers dispatch syntax, background runs, the DENIED escalation protocol, and result verification.
---

# agy-run — dispatch work to a cheap Gemini subagent

`agy-run` runs Antigravity CLI (`agy`) headlessly with a Gemini 3.5 Flash (Low) worker. It is a
local command, already on PATH. It fixes agy's non-TTY stdout bug, binds the working directory,
enforces output discipline, and auto-falls-back to Claude Sonnet 4.6 on usage limits.

## When to delegate (and when not)

Delegate: bulk file reading/searching/summarizing, terminology-consistency sweeps, mechanical
multi-file edits, generating boilerplate, first-pass doc audits — anything where volume is high
and per-step judgment is low.

Do NOT delegate: architectural decisions, subtle refactors, anything whose result you cannot
cheaply verify, tasks touching credentials.

## Dispatch

```bash
agy-run [-C workdir] [-m model] [-f fallback] [-t seconds] "<task prompt>"
```

- Always pass `-C <absolute project dir>` — the worker does NOT inherit your cwd reliably.
- Default timeout 300s is a ceiling, not a wait; short tasks return in tens of seconds.
- For tasks likely >1 min, use a background Bash run and continue other work; read the output
  when notified. Output streams incrementally, so polling shows progress.
- Write task prompts like orders to a junior: exact files/globs, exact deliverable format,
  "paste real command output verbatim".

## Permission / DENIED protocol

The agy side denies `rm` and `git push` (deny rules in `~/.gemini/antigravity-cli/settings.json`);
everything else is auto-approved. If the reply contains a line starting with `DENIED:`, the
worker hit a deny rule. Do not tell it to work around the denial. Instead: decide whether the
command is genuinely needed; if yes, run it yourself through your own Bash tool so the human
approves it through the normal permission prompt.

## Verify before trusting

Small models fabricate. For any load-bearing result:
- Have the worker include verifiable anchors (file paths + line numbers, `git rev-parse HEAD`).
- Spot-check 1-2 claims with your own Read/Grep before acting on the result.
- If output smells templated or too smooth, re-run with `-m "Gemini 3.5 Flash (High)"` or do it
  yourself.

## Failure modes

- Empty output / `authentication failed`: agy auth expired — tell the user to run `agy` once
  interactively to re-login.
- `[agy-run: answered by fallback model …]` header: the cheap model hit its usage limit; the
  answer came from Claude Sonnet 4.6 — costlier, mention it to the user.
- Exit 124: hard timeout; re-dispatch with a bigger `-t` or split the task.

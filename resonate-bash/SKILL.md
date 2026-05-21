---
name: resonate-bash
description: Use Resonate's `resonate-bash` MCP tool to run shell scripts as durable, asynchronous tasks. Use when waiting on something that takes minutes-to-hours (CI runs, deploys, DNS/SSL propagation, image-generation jobs, data-export polls), when work must survive a session close or host crash, or when you want a named, queryable promise ID a later session can look up. Covers what `resonate-bash` is good at, how to install the local Resonate server + Claude Code MCP wiring, the tool's parameter surface, target addresses (local / Docker / Tensorlake), the env vars injected into every script, failure semantics, and idempotency rules. Reach for this instead of `Bash(run_in_background)` whenever the work is longer-running than a foreground command or needs to outlive the session.
license: Apache-2.0
---

# resonate-bash

## Overview

`resonate-bash` is a Resonate MCP tool that runs shell scripts as durable, asynchronous tasks. An agent submits a script, gets a promise ID back immediately, and is notified when the command terminates. The script runs in the background — on the local host, inside a Docker container, or in a remote sandbox — and outlives the session it was submitted from.

The polling loop runs in the shell, not in the model. The model is not invoked during the wait. That is the entire point.

## When to use this skill

Reach for `resonate-bash` when **any** of the following is true:

- The work is likely to take longer than ~2 minutes.
- The work must survive a session close, a host crash, or a new conversation tomorrow.
- You want a named promise ID a later session can look up.
- You want the script to outlive the calling agent — fire-and-watch coordination with external systems.

Keep regular `Bash` (foreground or `run_in_background: true`) for fast commands, single-session work where survival doesn't matter, or non-idempotent fire-then-poll patterns the durable runtime would amplify (the script restarts from the top on crash — guard with check-then-trigger if a side effect must not double-fire).

## What `resonate-bash` is good at

**Long-running polling loops.** Waiting for an external system to reach a state — a CI run finishes, a deploy goes live, DNS propagates, an image-generation job completes, a Tensorlake sandbox returns, a BigQuery export job finishes. The `until X; sleep N; done` loop runs in the shell, not in the model.

```json
{
  "id": "ci-watch-2026-05-21",
  "script": "until gh run view 12345 --json status -q .status | grep -q completed; do sleep 60; done",
  "timeout_ms": 3600000
}
```

**Operations that need to outlive the current session.** Promises live on the Resonate server, not in the calling Claude Code session. If the laptop closes, the host hiccups, or a new conversation starts tomorrow, the work continues. A later session can look up the promise ID and read the result via `promise-get` or `promise-search`.

**Named, queryable state.** Promise IDs are first-class. Prefix them by project and date (`tamrack-ssl-watch-2026-05-21`, `echo-deploy-2026-05-21`), then filter `promise-search` later via the `tags` parameter to audit what fired across days. Cross-conversation lookup isn't automatic — a later session only finds the ID if it was recorded somewhere that session reads (handoff doc, project AGENT.md, the prompt the operator gives you).

**Fire-and-watch coordination with external systems.** Most CI tools, deploy platforms, image-generation APIs, and data-export jobs expose a status endpoint but no webhook. `resonate-bash` is the right shape for that: submit the work, poll in the shell, get notified on completion.

**Composable with the other Resonate promise primitives.** The same promise IDs work with `promise-create`, `promise-listen`, and `promise-settle`. A one-off script can be promoted into a multi-step durable workflow later without re-architecture.

## Tool reference

### Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `script` | yes | — | Inline bash script. Base64-encoded into `param.data` for you. |
| `target` | no | `bash://` | Where the script runs. See [target addresses](#target-addresses). |
| `timeout_ms` | no | 5 min | Promise deadline relative to now. |
| `id` | no | `bash-<millis>-<nanos>` | Deterministic promise id for idempotency. **Always set this** for queryability. |
| `tags` | no | — | JSON object merged into promise tags. `resonate:target` is set automatically. |

The tool registers a listener and resolves via channel notification when the script terminates, returning `{exit_code, stdout, stderr}`.

### Target addresses

| Address | Where it runs | Notes |
|---|---|---|
| `bash://` | Local shell on the resonate host | Default if `target` omitted |
| `bash://docker/<image>` | `docker run --rm <image> bash -c <script>` | Image required; the resonate host must have Docker |
| `bash://tensorlake/<image>` | Tensorlake Sandboxes API | `<image>` optional — empty path uses Tensorlake's default sandbox. Requires `TENSORLAKE_API_KEY` on the resonate process |

### Env vars injected into every script

| Variable | Meaning |
|---|---|
| `RESONATE_PROMISE_ID` | The promise/task id |
| `RESONATE_PROMISE_CREATED_AT` | ms since epoch — **stable across retries** |
| `RESONATE_PROMISE_TIMEOUT_AT` | ms since epoch — **stable across retries** |

Loop **until `$RESONATE_PROMISE_TIMEOUT_AT`**, not for a fixed duration. That's what keeps a restart-from-top retry idempotent.

### Failure semantics

- **Script exits non-zero** → workflow failure; the promise rejects with the exit info.
- **Script killed** (local signal / Docker exit 137 or 143 / Tensorlake `signaled`) → treated as **infrastructure failure**: the lease expires and the message is redispatched to a fresh worker. Don't rely on signals to short-circuit a workflow.
- **Crash/restart** → the script restarts from the top. Always write idempotent scripts. For "trigger external action then poll" patterns, structure as `check-then-trigger + poll` so a restart does not double-fire.

### Promise-ID conventions

- Always pass `id`. Auto-generated IDs are unqueryable and hostile to handoffs.
- Prefix by project and date: `tamrack-ssl-watch-2026-05-21`, `echo-deploy-2026-05-21`, `pulse-bq-freshness-2026-05-21`.
- Set `tags: { project: "<name>" }` so `promise-search` can filter cleanly.

## Installation

`resonate-bash` is delivered through the Resonate MCP. The setup is the same regardless of the IDE — a local Resonate server, a Claude Code MCP entry, and (today) one preview-channel flag on the `claude` CLI.

### Prerequisites

- macOS (Apple Silicon or Intel) with Homebrew, or a Linux host with the resonate binary on `$PATH`.
- Claude Code installed.
- (Optional, for Tensorlake target) a Tensorlake API key from <https://tensorlake.ai>.

> **Intel Mac and Linux note**: paths below assume `/opt/homebrew/bin/resonate` (Apple Silicon Homebrew). On Intel Mac use `/usr/local/bin/resonate`; on Linux, the path the package manager / installer chose.

### 1. Install the Resonate binary

```bash
brew install resonatehq/tap/resonate
resonate --version   # expect 0.9.7 or newer
```

A single binary serves both as the server (`resonate dev` / `resonate serve`) and as the MCP shim Claude talks to (`resonate mcp`).

### 2. Run the Resonate server with the bash transport enabled

Two options. Start with 2a to confirm everything works, then switch to 2b for a persistent install.

**2a. Foreground (temporary):**

```bash
# Only if you'll target bash://tensorlake/...
export TENSORLAKE_API_KEY="tl_apiKey_REPLACE_ME"

resonate dev \
  --server-port 8888 \
  --transports-bash-exec-enabled
```

Verify in another shell:

```bash
curl -s http://localhost:8888/health   # → 200 OK
```

**2b. Background via launchd (macOS persistent install):**

Write `~/Library/LaunchAgents/io.resonatehq.resonate.dev.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>io.resonatehq.resonate.dev</string>

    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/resonate</string>
        <string>dev</string>
        <string>--server-port</string>
        <string>8888</string>
        <string>--transports-bash-exec-enabled</string>
        <string>true</string>
    </array>

    <!-- Omit this block if you are not using bash://tensorlake/... -->
    <key>EnvironmentVariables</key>
    <dict>
        <key>TENSORLAKE_API_KEY</key>
        <string>tl_apiKey_REPLACE_ME</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/resonate-dev.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/resonate-dev.log</string>
</dict>
</plist>
```

Load it:

```bash
launchctl load ~/Library/LaunchAgents/io.resonatehq.resonate.dev.plist
launchctl list | grep resonate          # expect PID, exit code 0
curl -s http://localhost:8888/health    # → 200 OK
```

**Restart cheat-sheet:**

| When | Command |
|---|---|
| Changed the binary or want a clean restart | `launchctl kickstart -k gui/$(id -u)/io.resonatehq.resonate.dev` |
| Changed `EnvironmentVariables` or `ProgramArguments` in the plist | `launchctl unload <plist> && launchctl load <plist>` (kickstart will **not** pick up new env vars) |

Logs: `tail -f /tmp/resonate-dev.log`.

> The plist stores any API keys in plaintext under your home directory. For a hardened setup, source them from Keychain in a wrapper script and exec `resonate` from there.

### 3. Wire Claude Code to the Resonate MCP server

Edit `~/.claude.json` and add a `resonate` entry under `mcpServers`. Merge with existing entries — don't overwrite the file:

```json
{
  "mcpServers": {
    "resonate": {
      "type": "stdio",
      "command": "/opt/homebrew/bin/resonate",
      "args": ["mcp", "--server", "http://localhost:8888"],
      "env": {}
    }
  }
}
```

### 4. Enable the Claude Code preview channel

The `mcp__resonate__resonate-bash` tool ships behind a Claude Code preview channel. Add the flag to your `claude` alias in `~/.zshrc` (or `~/.bashrc`):

```bash
alias claude='claude --dangerously-load-development-channels server:resonate'
```

Reload the shell: `source ~/.zshrc`.

### 5. Verify

Open a fresh `claude` session and run:

```
/mcp
```

`resonate` should be listed and connected. Then exercise the tool with a real durable promise:

> Please watch my desktop for a file `Resonate.md` to appear, with resonate.

Claude should call `mcp__resonate__resonate-bash` with a poll-until-exists script. Touch the file in another shell — Claude will be notified the moment the promise resolves.

Follow-ups that exercise more of the surface:

> How did you do that? Show me the promise. And show me the script.

> What else can you use resonate for? Give me three examples.

> Run `echo hello from $RESONATE_PROMISE_ID` on tensorlake.

## Composing with the rest of the promise API

`resonate-bash` lives in the same promise namespace as the other MCP tools. Once a script is submitted you can:

- `promise-get { id }` — read state and value at any time, from any session.
- `promise-listen { id }` — block on resolution (useful for cross-session "wait for the thing the previous session started").
- `promise-search { tags: { project: "..." } }` — audit what fired and when.
- `promise-settle { id, state: "resolved" | "rejected", value }` — externally resolve a coordination promise. Pair a `resonate-bash` poll loop with an externally-settled promise for human-in-the-loop checkpoints.

A one-off durable script that earns reuse (3+ uses, or it encodes a hard-won gotcha) is the right candidate for promotion into a registered workflow via one of the per-SDK skills.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/mcp` doesn't show `resonate` | MCP entry missing from `~/.claude.json`, or the server isn't running | `curl http://localhost:8888/health`; check the JSON entry |
| `mcp__resonate__resonate-bash` tool absent in Claude | Missing `--dangerously-load-development-channels server:resonate` flag | Re-check the alias; restart `claude` |
| Promise resolves with `TENSORLAKE_API_KEY env var not set` | Env var not in resonate process env | For launchd: `unload` then `load` (`kickstart` won't pick up new env). Verify with `ps eww $(launchctl list \| awk '/resonate/{print $1}') \| tr ' ' '\n' \| grep TENSORLAKE` |
| Promises stuck `pending` forever | Bash exec transport not enabled | Confirm `--transports-bash-exec-enabled` is in the resonate launch args |
| Tensorlake sandbox creation hangs | Sandbox readiness timeout (120s) exceeded; bad image name | Check `/tmp/resonate-dev.log` for `tensorlake create` errors |

### Tensorlake-specific gotchas

- Sandbox lifetime is hardcoded server-side to 600s. Promises with longer timeouts will see a fresh sandbox per retry.
- `TENSORLAKE_API_KEY` is read directly from the resonate process env via `std::env::var` (not from `RESONATE_*` config). It must be visible to the **resonate process**, not just your interactive shell.

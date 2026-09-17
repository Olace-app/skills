---
name: olace-setup
description: Install and set up Olace on a computer so its local AI models (Ollama, LM Studio, llama.cpp) are reachable from the user's phone and other devices over an end-to-end encrypted connection, and served to local apps through an OpenAI-compatible endpoint. Use when a user asks to set up Olace, use their home GPU or local model from their phone or laptop, pair devices, or run a private AI host on a server or cloud VM. Works headless, no display needed.
---

# Set up Olace

Olace turns one computer into a private AI host. The daemon runs in the background, the `olace` command drives it, and every device the user pairs streams from the host's models over an end-to-end encrypted connection, on the LAN directly or through Olace's relay when away. The host only makes outbound connections: no port forwarding, no VPN, no public address. Olace never sees the prompts or responses of local and paired inference.

Everything below runs without a terminal on stdin. Two moments need the user: reading a 6-digit sign-in code from their email, and entering a pairing code on their other device. Relay both; never guess them.

## Before you start

- Supported hosts: Linux, macOS, Windows. Any of them may be headless (SSH-only, a server, a rented cloud VM).
- A local AI runtime: Ollama, LM Studio (or its headless `llmster`), or llama.cpp. Olace uses whichever is installed and can install one, but that step needs a terminal. If none is installed, install one yourself first (for example Ollama's own installer), then continue.
- A GPU is not required, but a model has to fit the machine. `olace setup` picks a starter model sized to the hardware it detects.
- Tell the user up front: an Olace account is free, local and paired AI never costs credits, and sign-in is a one-time email code with no password.

Docs, in agent-readable form: https://olace.app/docs/llms.txt (index) and https://olace.app/docs-md/<page>.md (any page as raw markdown). This flow is the `agents` page: https://olace.app/docs-md/agents.md

## Step 1: Install

Linux and macOS:

```bash
OLACE_NONINTERACTIVE=1 sh -c "$(curl -fsSL https://olace.app/install.sh)"
export PATH="$HOME/.olace/bin:$PATH"
olace version
```

Windows (PowerShell):

```powershell
$env:OLACE_NONINTERACTIVE = "1"
irm https://olace.app/install.ps1 | iex
& "$env:USERPROFILE\.olace\bin\olace.cmd" version
```

`OLACE_NONINTERACTIVE=1` makes the installer skip its two questions (PATH edit, handoff into setup) so it never blocks. It still installs the daemon, the `olace` command and the background service. Ask the user whether they want `~/.olace/bin` on their PATH permanently; the installer prints the exact line for their shell when you leave it out.

Re-running the installer is safe: an existing install is updated in place.

## Step 2: Set up local AI

```bash
olace setup < /dev/null
```

Four steps, each skipped when already done: runtime, starter model, Python sandbox (lets models run code on this machine for data files and charts), sign-in. With no terminal it prints what each step would do instead of asking.

- Exit `0` and "Setup complete": move on.
- Exit `2` and "Some steps need a terminal to confirm": read which step. A runtime install needs a terminal, so install the runtime yourself and re-run. The model and sandbox steps print their own pointer.
- A specific model instead of the starter pick: `olace pull qwen3.5:4b` (Ollama tags use `:`; LM Studio ids use `/`; `--provider ollama|lms|llamacpp` overrides). Pulling never prompts.

Sign-in is its own step below, because setup cannot read the email code without a terminal.

## Step 3: Sign in

Sign-in requests a 6-digit code by email, then reads it from stdin. The code does not exist when the command starts, so keep stdin open and append the code when the user gives it to you:

```bash
: > /tmp/olace-code.txt
(tail -f /tmp/olace-code.txt | olace signin --email USER@EXAMPLE.COM > /tmp/olace-signin.log 2>&1 &)
sleep 5 && cat /tmp/olace-signin.log
```

Expected so far:

```text
A 6-digit code was sent to USER@EXAMPLE.COM. It expires in 10 minutes.
Enter the code (or 'r' to resend, blank to cancel):
```

Ask the user for the code from their email. Then:

```bash
echo 575537 >> /tmp/olace-code.txt      # the user's code
sleep 10 && cat /tmp/olace-signin.log
pkill -f "tail -f /tmp/olace-code.txt"
```

Expected:

```text
Signed in as USER@EXAMPLE.COM.
The Olace background service is running.
```

Notes:

- A wrong code prints "That code is not correct or has expired" and waits again; append the corrected code the same way. A blank line cancels.
- If the log says "Start the background service later", the service was not installed (it normally is, by the installer). Run `olace service install`.
- The signed-in daemon picks the session up on its own within seconds; nothing to restart.
- On Windows, run `olace signin --email ...` in a terminal the user can type into instead of the stdin relay.

Verify:

```bash
olace whoami --json
```

## Step 4: Pair the user's other devices

The host shows a pairing code; the user enters it on the device that should use this computer's models. Run it in the background because it waits until claimed:

```bash
(olace pair --code --ascii > /tmp/olace-pair.log 2>&1 &)
sleep 3 && head -1 /tmp/olace-pair.log
```

Expected first line: `Pairing code: ABCD-1234`. Tell the user:

- On a phone: open Olace, sign in with the same account, go to **Settings › Paired devices**, and enter the code (or scan the QR in the log file).
- On another computer with Olace installed: `olace pair claim ABCD-1234`.

The command exits `0` and prints `Paired with <device name>.` when claimed. A code that expires is replaced with a fresh one in the same log; re-read the first `Pairing code:` line after "Code expired". Kill it with `pkill -f "olace pair --code"` only if the user stops.

Same-account computers can also pair without a code: `olace pair` on the consuming machine lists the account's hosts, but that menu needs a terminal.

Verify:

```bash
olace pair list --json      # pairs[] holds this computer's pairings
olace status --json         # pairs.active > 0
```

The free plan includes one pairing. `pairing_usage` in `pair list --json` shows used, cap, and the plan that raises it; report a full cap to the user rather than retrying.

## Step 5: Serve local apps (optional)

```bash
olace bridge on --require-key
olace bridge status --json
```

This starts the Olace Bridge, an OpenAI-compatible endpoint at `http://127.0.0.1:5578/v1` for every local and paired model. It listens on loopback only. `--require-key` mints a Bearer key so other local processes cannot use it unasked. The key is printed once, on the line after "Bridge API key (shown once, stored in daemon.json):"; capture it from that output, because `olace bridge key` only reports whether one is set (`--regenerate` mints a fresh one). See the `olace-bridge` skill for pointing tools at it.

## Verify the whole thing

```bash
olace status --json
```

Read: `account.signed_in`, `daemon.running`, `daemon.tunnel_connected` (the host is reachable from the user's other devices), `runtimes.<name>.alive`, `python.ready`, `pairs.active`, `bridge.listening`. `olace list` shows the models this computer can serve.

A first test that exercises the model: `olace chat "say hi in five words"`.

## Exit codes

`0` success, `1` failure, `2` usage error or a step that needs a terminal, `3` not signed in, `4` network error, `5` daemon not running, `6` update required (`olace update`).

## Do not

- Do not run `olace signout` unless the user asks. It signs out the whole computer, including the Olace desktop app, and clears its pairing and Bridge settings. It needs `--yes` without a terminal.
- Do not edit `~/.olace/daemon.json` by hand. The daemon and the app own it.
- Do not paste the sign-in code, the pairing code, or the Bridge key anywhere except the command that consumes it.
- Do not open firewall ports or set up a VPN for Olace. The host is reachable through outbound connections only.

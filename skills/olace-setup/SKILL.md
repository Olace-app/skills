---
name: olace-setup
description: Install and set up Olace on a computer so its local AI models (Ollama, LM Studio, llama.cpp) are reachable from the user's phone and other devices over an end-to-end encrypted connection, and served to local apps through an OpenAI-compatible endpoint. Use when a user asks to set up Olace, use their home GPU or local model from their phone or laptop, pair devices, or run a private AI host on a server or cloud VM. Works headless, no display needed.
---

# Set up Olace

Olace turns one computer into a private AI host. The daemon runs in the background, the `olace` command drives it, and every device the user pairs streams from the host's models over an end-to-end encrypted connection, on the LAN directly or through Olace's relay when away. The host only makes outbound connections: no port forwarding, no VPN, no public address. Olace never sees the prompts or responses of local and paired inference.

Everything below runs without a terminal on stdin. Two moments need the user: reading a 6-digit sign-in code from their email, and entering a pairing code on their other device. Relay both; never guess them.

## Before you start

- Supported hosts: Linux, macOS, Windows. Any of them may be headless (SSH-only, a server, a rented cloud VM).
- A local AI runtime: Ollama, LM Studio (or its headless `llmster`), or llama.cpp. Olace uses whichever is installed and can install one, but that step needs a terminal. If none is installed, install one yourself first (for example Ollama's own installer), then continue. Ollama's Linux installer needs `zstd`, which minimal images lack (`apt-get install -y zstd`). Olace starts Ollama itself when a model needs it, with or without systemd.
- A GPU is not required, but a model has to fit the machine. `olace setup` picks a starter model sized to the hardware it detects.
- Tell the user up front: an Olace account is free, local and paired AI never costs credits, and sign-in is a one-time email code with no password.
- A temporary machine (a cloud sandbox, a CI runner, a VM you will delete) joins the account as a device, and its pairings count against the plan's cap. Before it goes away, remove them with `olace pair remove <pair id> --yes`.

Docs, in agent-readable form: https://www.olace.app/docs/llms.txt (index) and https://www.olace.app/docs-md/<page>.md (any page as raw markdown). This flow is the `agents` page: https://www.olace.app/docs-md/agents.md

Fetch these on the `www.` host, as written. `olace.app` 308s to it, so a client that does not follow redirects (bare `curl -s` is one, `curl -sL` is not) reads a 15-byte "Redirecting..." body as the page.

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

`OLACE_NONINTERACTIVE=1` makes the installer skip its two questions (PATH edit, handoff into setup) so it never blocks. It still installs the daemon, the `olace` command and, where the system has a service manager, the background service. Ask the user whether they want `~/.olace/bin` on their PATH permanently; the installer prints the exact line for their shell when you leave it out.

The installer verifies the release signature and refuses to install unsigned. If it says it "could not fetch" the manifest or the signature, retry; a fetch can fail transiently.

Re-running the installer is safe: an existing install is updated in place.

### No systemd (containers)

On Linux without systemd, most containers included, there is no service manager to register with. The installer says so:

```text
No background service here: systemd is not running (a container?).
Start Olace with: /root/.olace/bin/olace-daemon run   (under your own supervisor to keep it up)
```

Start the daemon yourself and leave it running:

```bash
nohup olace run > ~/olace-run.log 2>&1 &
```

Everything else works the same. `olace run` restarts itself in place when it has to (sign-out, `olace update`, `olace service restart` from this or a paired computer), so it needs no supervisor for those. It does not outlive the machine's processes: after the container restarts, run it again. `olace service install` refuses here; that is expected.

## Step 2: Set up local AI

```bash
olace setup < /dev/null
```

Four steps, each skipped when already done: runtime, starter model, Python sandbox (lets models run code on this machine for data files and charts), sign-in. With no terminal it prints what each step would do instead of asking.

- Exit `0` and "Setup complete": move on.
- Exit `2` and "Some steps need a terminal to confirm": read which step. A runtime install needs a terminal, so install the runtime yourself and re-run. The model step prints `Get it with: olace pull <model>` with the download size; run that. The sandbox step prints `No terminal to confirm. Run: olace setup --sandbox --yes`; run exactly that (about 120 MB to download).
- The sandbox is offered only where models big enough to use it (above about 7B) can run. On hardware that only runs small models, setup says "Not needed yet" and moves on; that is not an error.
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
pkill -f '^tail -f /tmp/olace-code\.txt$'
```

Keep the `pkill` pattern anchored as written. An unanchored `pkill -f "tail -f /tmp/olace-code.txt"` also matches your own shell, whose command line contains that text, and kills it. Recent `tail` versions exit on their own once sign-in finishes, so the `pkill` is often a no-op.

Expected:

```text
Signed in as USER@EXAMPLE.COM.
The Olace background service is running.
```

Under `olace run` (no systemd), the second line is `Olace is running.` instead.

Notes:

- A wrong code prints "That code is not correct or has expired", says how many attempts are left, and waits again; append the corrected code the same way. `r` requests a fresh code; a blank line cancels.
- If the log says "Start the background service later with 'olace service install' or 'olace run'", no daemon is running. Run `olace service install`, or on a machine without systemd start `olace run` as in Step 1.
- The signed-in daemon picks the session up on its own within seconds; nothing to restart.
- On Windows, run `olace signin --email ...` in a terminal the user can type into instead of the stdin relay.

Verify:

```bash
olace whoami --json
```

## Step 4: Pair the user's other devices

Pairing is done from the device that will use this computer's models. There are three ways; the first two need no code.

**From the phone's own list.** In the Olace app, signed in to the same account, this computer appears among the account's computers once it has signed in. The user picks it there. Nothing to run on the host.

**From another computer on the same account.** On that computer, pair by this computer's name. No terminal needed:

```bash
olace pair --host <this computer's name>
```

It prints `Paired with <name>. N models available, including ...`. A name that matches nothing lists the account's computers to choose from. Plain `olace pair` shows the same list as a menu, but only in a terminal.

**By code**, for any device. The host shows a code; the user enters it on the other device. Run it in the background because it waits until claimed:

```bash
(olace pair --code --ascii > /tmp/olace-pair.log 2>&1 &)
sleep 3 && head -1 /tmp/olace-pair.log
```

Expected first line: `Pairing code: K7QM-3XPD`. Codes never contain 0, O, 1, I, L or U, so the user can read them back without guessing. Tell the user:

- On a phone: open Olace, sign in with the same account, go to **Settings › Paired devices**, and enter the code (or scan the QR in the log file).
- On another computer with Olace installed: `olace pair claim K7QM-3XPD`.

The command exits `0` and prints `Paired with <device name>.` when claimed. A code that expires is replaced with a fresh one in the same log; re-read the latest `Pairing code:` line after "Code expired". Kill it with `pkill -f '^[^ ]*olace pair --code'` only if the user stops. Asking for more than about five codes a minute gets "Too many requests"; wait and retry.

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

`identity.key_source` is `machine_id` when the system has no OS keystore (no gnome-keyring or KWallet), which is normal on servers and containers. `olace status` then says the identity key is "protected only by this machine's id". Tell the user; do not install a keyring unasked.

A first test that exercises the model: `olace chat "say hi in five words"`.

## Updating

```bash
olace update --check
olace update
```

`olace update` verifies and installs the latest release, then restarts the daemon, in place under `olace run` too, and prints `Olace <version> is running.` If it reports that the update was rolled back and the daemon runs under `olace run`, stop `olace run`, run `olace update` again, then start `olace run`.

## Exit codes

`0` success, `1` failure, `2` usage error or a step that needs a terminal, `3` not signed in, `4` network error, `5` daemon not running, `6` update required (`olace update`).

## Do not

- Do not run `olace signout` unless the user asks. It signs out the whole computer, including the Olace desktop app, and clears its pairing and file sharing (local models and the Bridge keep working, with their settings). Signing back in needs a new email code. It needs `--yes` without a terminal.
- Do not edit `~/.olace/daemon.json` by hand. The daemon and the app own it.
- Do not paste the sign-in code, the pairing code, or the Bridge key anywhere except the command that consumes it.
- Do not open firewall ports or set up a VPN for Olace. The host is reachable through outbound connections only.

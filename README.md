# Olace skills for AI agents

Agent Skills that let a coding agent (Claude Code, Codex CLI, Cursor, Gemini CLI, and any other client of the [Agent Skills](https://github.com/vercel-labs/skills) format) set up and use [Olace](https://olace.app) for a person, end to end, without a display.

| Skill | What it does |
| --- | --- |
| `olace-setup` | Install Olace on a computer, set up a local AI runtime and model, sign in, pair the user's phone or other computers, turn on the Bridge. Two human moments, both relayed by the agent: the sign-in code from email, and the pairing code on the other device. |
| `olace-bridge` | Point an OpenAI-compatible tool (Codex CLI, Continue, Cline, Aider, the openai SDK, and others) at the Olace Bridge so it runs on the user's own models or a paired computer's GPU. |

## Install

```bash
npx skills add Olace-app/skills                      # pick skills and agents interactively
npx skills add Olace-app/skills --skill olace-setup -a claude-code
npx skills add Olace-app/skills --all                # every skill, every agent, no prompts
```

Or copy a `skills/<name>/SKILL.md` into your agent's skills directory by hand.

## What Olace is

One computer runs the models, through Ollama, LM Studio, or llama.cpp. Every device the user pairs streams from it over an end-to-end encrypted connection, on the LAN directly or through Olace's relay when away, with no port forwarding, no VPN, and no public address. The Olace Bridge serves the same models to any OpenAI-compatible tool on the machine. Local and paired inference is free and never reaches Olace's servers. Docs for agents: https://www.olace.app/docs/llms.txt (the `www.` host: `olace.app` redirects to it, and a client that does not follow redirects reads the redirect body as the page).

## Contributing

These files are tested against the shipping `olace` CLI. If a command's output changes, the skill changes in the same release. Issues and pull requests welcome.

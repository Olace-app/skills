---
name: olace-bridge
description: Point any OpenAI-compatible tool (Codex CLI, Continue, Cline, Aider, OpenCode, Zed, Open WebUI, the openai SDK) at the Olace Bridge so it runs on the user's own local models or on the GPU of a paired computer, privately. Use when a user wants a coding agent or chat app to use their local model, their home GPU from a laptop, or a private OpenAI-compatible endpoint without a cloud API key.
---

# Use the Olace Bridge from other tools

The Olace Bridge is an OpenAI-compatible endpoint served by the Olace daemon at `http://127.0.0.1:5578/v1`. It exposes every model on this computer (Ollama, LM Studio, llama.cpp) and every model on the user's paired computers, reached over an end-to-end encrypted connection. The model id decides where inference runs: `ollama/qwen3.5:4b` runs here, `home-pc/ollama/qwen3.5:14b` runs on the paired host called "Home PC". Olace's servers never see the prompts or responses.

Olace must be installed and, for paired models, signed in and paired. If it is not, follow the `olace-setup` skill first.

## Turn it on

```bash
olace bridge on --require-key
olace bridge status --json
```

`state` is `listening` when ready. `--require-key` mints a Bearer key on first use and prints it once, on the line after "Bridge API key (shown once, stored in daemon.json):"; capture it there. `olace bridge key` only reports whether one is set; `--regenerate` mints a fresh one, `--no-key` removes the requirement (any process on the machine can then connect). The Bridge listens on loopback only; a different port is `olace bridge on --port <n>`.

## Pick a model

```bash
curl -s http://127.0.0.1:5578/v1/models -H "Authorization: Bearer $KEY" | jq -r '.data[].id'
```

Ids are provider-qualified, paired hosts carry their pairing label as a prefix, and unambiguous shorthand is accepted. Models on a host that is currently offline are listed too; a request to one fails until the host is back. `olace list all` shows the same table with sizes and capabilities.

## Endpoints

| Route | Notes |
| --- | --- |
| `GET /v1/models`, `GET /v1/models/{id}` | Local and paired, one list. |
| `POST /v1/chat/completions` | Streaming and non-streaming. Tool calls where the model supports them. |
| `POST /v1/completions` | Legacy text completions. |
| `POST /v1/embeddings` | Local and paired embedding models. |

No `/v1/responses`, no image generation. Tools that require the Responses API need their chat-completions wire mode (Codex CLI: `wire_api = "chat"`).

## Test

```bash
curl -s http://127.0.0.1:5578/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"ollama/qwen3.5:4b","messages":[{"role":"user","content":"Reply with the word ready."}]}'
```

## Configure the user's tool

Set base URL `http://127.0.0.1:5578/v1`, API key to the Bridge key (or any non-empty string when no key is required), and a model id from the list above.

| Tool | Where |
| --- | --- |
| openai SDK (Python, JS) | `OpenAI(base_url="http://127.0.0.1:5578/v1", api_key=KEY)` |
| Codex CLI | `~/.codex/config.toml`: a `[model_providers.olace]` with `base_url` and `wire_api = "chat"`, then `model_provider = "olace"` |
| Continue | `~/.continue/config.yaml`: provider `openai`, `apiBase` |
| Cline | Settings › API Provider › OpenAI Compatible |
| Aider | `OPENAI_API_BASE` and `OPENAI_API_KEY` environment variables, `--model openai/<id>` |
| OpenCode | `opencode.json`: provider with `baseURL` |
| Zed | `settings.json`: `openai_compatible` provider |
| Open WebUI | Admin Settings › Connections › OpenAI |
| Jan, Chatbox, Cherry Studio | Add provider, type "OpenAI API Compatible" |

Full table with the exact keys: https://olace.app/docs-md/bridge.md

## Troubleshooting

- Connection refused: `olace bridge status --json`; if `daemon_running` is false, `olace service restart`.
- 401: the tool is not sending the Bridge key as a Bearer token, or the key was regenerated.
- 404 with candidates: the model id was ambiguous; use one of the listed ids.
- A paired model errors with the host offline: `olace status --device <host>`; the host machine must be on and running its daemon.
- First request on a model is slow: the runtime is loading it into GPU memory. Later requests are fast until it idles out.

---
name: freellmapi
description: >
  Set up and use FreeLLMAPI (github.com/tashfeenahmed/freellmapi), a self-hosted gateway that
  pools the free tiers of many LLM providers behind one OpenAI- and Anthropic-compatible
  endpoint. Use when the user wants free LLM API access through it: installing or running the
  gateway, adding provider keys, calling it from code, pointing a coding agent (Claude Code,
  Codex, Cline, Aider, OpenCode...) at it, reading its usage, or fixing it.
---

# FreeLLMAPI

FreeLLMAPI is a local-first, single-user router (MIT, by Tashfeen Ahmed). The user adds API keys
for providers' free tiers (Google, Groq, Cerebras, Mistral, OpenRouter, Cloudflare, Cohere,
NVIDIA, Hugging Face and many more) in its dashboard. It stores them encrypted and gives apps one
unified bearer token (`freellmapi-...`) and one base URL. A router picks a model for each
request, fails over to the next one on a 429 or 5xx, and tracks each key's limits so requests
stay under every free tier's caps.

The project moves fast. Its README, its `docs/en/` folder and https://freellmapi.co/models.html
(the live model list) are the source of truth: check them when a detail here and the user's
install disagree.

## First, find out what the user has

1. Is a gateway running? Server installs (Docker, the one-liner, a source run) listen on
   `http://localhost:3001`. The desktop app shows its address and key in its tray popover (its
   dashboard is usually `http://127.0.0.1:31415`).
   `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3001/v1/models` prints an HTTP
   status when it is up (401 only means no key was sent); a connection error means it isn't.
2. Is the unified key in the user's environment (`FREELLMAPI_API_KEY`)? Never ask the user to
   paste a key into the chat.

## Install (ask which the user wants)

- **Desktop app**, the simplest: on Windows from the Microsoft Store (signed, and it updates
  itself) or the `.exe` on the project's GitHub Releases; on macOS the `.dmg` (arm64 or x64)
  from Releases. No account to make: the unified key is in the tray popover.
- **Docker one-liner**: sets up `~/freellmapi`, generates the encryption key, starts the
  container. Download the script and let the user read it before it runs:
  ```bash
  curl -fsSL https://freellmapi.co/install.sh -o freellmapi-install.sh
  less freellmapi-install.sh
  bash freellmapi-install.sh
  ```
  Running it again is safe: the `.env` and its encryption key are kept.
- **Docker Compose, from source (Node.js 20.18 or later), LAN access, backups**: follow
  `docs/en/install/01-install.md` in the repository.

Server installs have an email and password for the dashboard. A forgotten password: "Forgot
password?" on the login page prints a one-time code, valid 15 minutes, to the server log
(`docker compose logs -f freellmapi`).

## Provider keys

- The user adds them on the dashboard's **Keys** page. Each comes from that provider's own
  free-tier sign-up, under that provider's terms: one account per provider, never extra accounts
  to multiply a quota.
- From a terminal, with a **dashboard session token** (the unified key can't manage keys):
  ```bash
  export FREELLMAPI_URL=http://localhost:3001
  export FREELLMAPI_DASHBOARD_TOKEN='<dashboard session token>'  # the user sets this themselves
  npx freellmapi keys add groq       # asks for the key with a hidden prompt
  npx freellmapi keys list           # never prints credentials
  npx freellmapi keys test groq
  npx freellmapi keys remove groq --id 7
  ```
  Use the hidden prompt rather than `--key`: a key on a command line stays in shell history.
- A key never goes into the chat, into the arguments of a command you run, into a log, or into a
  committed file. If one is exposed, tell the user to revoke it at the provider and add a new one.

## Calling it from code

Base URL `http://localhost:3001/v1`; the bearer token is the unified key, read from the
environment.

```python
import os
from openai import OpenAI

client = OpenAI(base_url="http://localhost:3001/v1", api_key=os.environ["FREELLMAPI_API_KEY"])
reply = client.chat.completions.with_raw_response.create(
    model="auto",  # or "auto:fast", "auto:smart", "auto:<profile>", or a model id
    messages=[{"role": "user", "content": "One sentence on the fall of Rome."}],
)
print(reply.parse().choices[0].message.content)
print("served by", reply.headers.get("x-routed-via"))
```

```bash
curl -s http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer $FREELLMAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "auto", "messages": [{"role": "user", "content": "Hello"}]}'
```

What it serves: `/v1/chat/completions`, `/v1/responses`, `/v1/completions`, `/v1/embeddings`,
`/v1/images/generations`, `/v1/videos/generations`, `/v1/audio/speech`,
`/v1/audio/transcriptions` and `/v1/models`; Anthropic's Messages API at `/v1/messages`; Gemini's
`/v1beta`; an opt-in Ollama surface; and an MCP server at `/mcp`. Interactive API docs are at
`/v1/docs`. `model: "fusion"` sends a prompt to several free models at once and has a judge model
merge their drafts. Every response names the model that served it in `X-Routed-Via`.

## Pointing a coding agent at it

`npx freellmapi setup-<tool>` writes a tool's config from the models the gateway actually serves:
`setup-claude`, `setup-codex`, `setup-cline`, `setup-continue`, `setup-aider`, `setup-opencode`,
`setup-cursor`, `setup-generic` and more (`npx freellmapi list`). It merges into the existing
config and keeps a timestamped backup of it.

- Run it with `--dry-run` first and show the user the diff.
- Take the key from the user's environment (`FREELLMAPI_API_KEY`) rather than typing it into
  `--api-key`.
- `setup-claude` changes Claude Code's own settings, so that Claude Code itself runs on the free
  pool instead of Anthropic's models: much weaker models, and no SLA. Do it only when the user
  explicitly asks, after saying so. To try it, `npx freellmapi launch` starts one Claude Code
  session with the credentials in that process only, with nothing written to disk.

## Tell the user up front

- Free tiers have no SLA and no frontier models. Latency varies, and quality dips late in the day
  as the best models reach their daily caps (they reset at midnight UTC).
- Some free tiers may keep prompts or use them to train: no secrets, credentials or personal data
  through it unless every provider enabled allows it under its own terms.
- A free install's catalog is a monthly snapshot: a new model reaches it 30 days after the live
  feed. The live catalog is a paid option of the project.
- It is single-user and local-first: keep it on localhost unless the user deliberately turns on
  LAN access, and keep it updated (security fixes ship on the latest release line only).

## Troubleshooting

- **401**: a wrong or missing unified key. **429 from every route**: every key is cooling down;
  wait, or add providers. **An unexpected model answered**: check `X-Routed-Via` and the
  dashboard's fallback chain.
- **Logs**: `docker compose logs -f freellmapi` (Docker), the terminal (a source run), or
  `<data dir>/logs/freeapi.log` (desktop: tray menu, Open Logs Folder).
- **Uninstall**: remove the app (or `docker compose down -v`), then delete its data folder:
  `%APPDATA%\FreeLLMAPI\`, `~/Library/Application Support/FreeLLMAPI/` or
  `~/.config/FreeLLMAPI/`.
- A security problem in FreeLLMAPI itself goes to the project's GitHub Security tab as a private
  report, never a public issue.

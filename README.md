# FreeLLMAPI skill for Claude Code

A Claude Code skill for [FreeLLMAPI](https://github.com/tashfeenahmed/freellmapi), the
self-hosted gateway that pools LLM providers' free tiers behind one OpenAI- and
Anthropic-compatible endpoint. With it, Claude Code knows how to install and run the gateway, add
provider keys without exposing them, call it from code, point coding agents at it, and fix it when
it misbehaves.

Unofficial: not made by, or affiliated with, FreeLLMAPI's author.

## Install

As a plugin, so `claude plugin update` keeps it current:

```bash
claude plugin marketplace add BarfbagX2/freellmapi-skill
claude plugin install freellmapi@freellmapi-skill
```

Then run `/reload-plugins` in Claude Code, or start a new session.

Or copy just the skill into your personal skills: `skills/freellmapi` to
`~/.claude/skills/freellmapi`.

## Use

Ask Claude Code things like "set up FreeLLMAPI with Docker", "call my FreeLLMAPI gateway from
Python", or "point Aider at FreeLLMAPI". The skill loads when a request is about FreeLLMAPI.

## What it keeps Claude from doing

- Asking for your API keys in the chat, or putting them in commands, logs or files.
- Switching Claude Code itself onto the free pool (`npx freellmapi setup-claude`) without your
  explicit say-so.

## License

This skill: MIT (see `LICENSE`). FreeLLMAPI: MIT © Tashfeen Ahmed.

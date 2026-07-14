# Claude Code CLI → Telegram: Collapsible Thinking

A step-by-step Chinese guide to displaying structured `thinking` blocks from Claude Code CLI (`claude -p`) as collapsible Telegram messages.

## What this guide covers

- Verifying whether the installed Claude Code version exposes independent `thinking` blocks
- Switching from `--output-format json` to `--output-format stream-json --verbose`
- Parsing NDJSON without mixing thinking, debug events, stderr, and the final answer
- Keeping `{ text, thinking }` separate throughout the application
- Rendering thinking with Telegram HTML `<blockquote expandable>`
- Adding `/think auto`, `/think off`, and `/think status`
- Keeping thinking out of history, memory, summaries, and embeddings
- Testing fallback paths so the final answer is never lost or duplicated

## Read the guide

After GitHub Pages is enabled, the site will be available at:

**https://gwendolenmave.github.io/claudep-telegram-thinking/**

The complete self-contained tutorial is also available directly in [`index.html`](./index.html).

## Important terminology

This repository discusses **structured thinking blocks exposed by Claude Code CLI**. It does not claim to reveal a complete or permanently stable hidden chain of thought. Event shapes can change between Claude Code versions, models, and runtime modes, so capability detection is the first step in the guide.

## Scope

This guide focuses on:

```text
Telegram Bot
→ custom backend
→ Claude Code CLI / claude -p
→ stream-json parser
→ Telegram reply
```

An Anthropic Messages API integration can reuse the Telegram renderer, settings, fallback, and memory-isolation layers, but it needs a different upstream adapter.

## License

Documentation, prompts, and code samples in this repository are licensed under **Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)**. See [`LICENSE.md`](./LICENSE.md).

You may copy, adapt, translate, share, and provide the material to coding agents for personal, educational, research, and other non-commercial use, provided that attribution and ShareAlike requirements are followed.

Please also read [`RESPONSIBLE_USE.md`](./RESPONSIBLE_USE.md).

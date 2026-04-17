# ACP adapter for the Claude Agent SDK

[![npm](https://img.shields.io/npm/v/%40yoke233%2Fclaude-agent-acp)](https://www.npmjs.com/package/@yoke233/claude-agent-acp)

> **Fork note.** This fork of [zed-industries/claude-agent-acp](https://github.com/zed-industries/claude-agent-acp) adds a **`session/steer`** extension for mid-turn user-input injection. See the [Mid-turn steer](#mid-turn-steer-extension) section below. Everything else matches upstream.

Use [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview#branding-guidelines) from [ACP-compatible](https://agentclientprotocol.com) clients!

This tool implements an ACP agent by using the official [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview), supporting:

- Context @-mentions
- Images
- Tool calls (with permission requests)
- Following
- Edit review
- TODO lists
- Interactive (and background) terminals
- Custom [Slash commands](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- Client MCP servers
- **Mid-turn steer** (this fork) — inject additional user input into a running turn without cancelling it

Learn more about the [Agent Client Protocol](https://agentclientprotocol.com/).

## Installation

```
npm install -g @yoke233/claude-agent-acp
# or run directly
npx @yoke233/claude-agent-acp
```

## Mid-turn steer extension

ACP's `session/prompt` is a blocking request-response: once a prompt is active, its JSON-RPC channel is held open until the `PromptResponse` comes back, so a client can't send another prompt on the same session until the turn finishes. That rules out a common UX pattern — letting the user append a follow-up message *while the agent is still thinking or executing tools*.

This fork exposes the capability the Claude Agent SDK already supports underneath: writing an additional user message into the active query's input stream, which the SDK reads at the next tool boundary.

### Capability

Advertised in the `initialize` response:

```jsonc
{
  "agentCapabilities": {
    "_meta": {
      "claudeCode": { "promptQueueing": true },
      "steer": true    // <-- new
    }
  }
}
```

Clients that also advertise support for `session/steer` may send the notification below; clients that don't know about it can ignore the capability.

### Wire format

`session/steer` is a **notification** (no response). It is delivered via the ACP extension-notification channel, so the adapter accepts either of these method names on the wire — the `_`-prefixed form matches how the Rust ACP SDK encodes extension methods:

- `session/steer` (raw, as sent by the TypeScript ACP SDK)
- `_session/steer` (prefixed, as sent by the Rust ACP SDK)

```jsonc
// Client → Agent (notification — no id, no response)
{
  "jsonrpc": "2.0",
  "method": "session/steer",
  "params": {
    "sessionId": "uuid-xxx",
    "prompt": [
      { "type": "text", "text": "use try/catch instead of if/else" }
    ]
  }
}
```

### Semantics

- The notification is **silently ignored** when there is no active turn for the session. No error is returned (notifications can't return errors anyway).
- Injected content is delivered to the Claude Agent SDK's query input stream and is picked up at the next tool boundary within the running turn.
- The originating `session/prompt` is **not** affected: it keeps streaming `session/update` notifications and eventually returns its own `PromptResponse` with `stopReason: "end_turn"` (or `"cancelled"`).
- Parameters accept the same `ContentBlock[]` shape as `session/prompt`.

## Contribution Policy

This project does not require a Contributor License Agreement (CLA). Instead, contributions are accepted under the following terms:

> By contributing to this project, you agree that your contributions will be licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0). You affirm that you have the legal right to submit your work, that you are not including code you do not have rights to, and that you understand contributions are made without requiring a Contributor License Agreement (CLA).

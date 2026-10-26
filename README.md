# Micup Developer Documentation — For LLMs and AI Agents

A consolidated, single-file technical reference designed to help AI coding tools correctly use Micup's public **HTTP API**, **Gateway**, **webhooks**, **voice/media** capabilities, and Micup-specific platform features.

> **Version:** 1.0 · **Live verification:** October 26, 2026  
> **Live documentation:** <https://micup.gg/docs/> · **Discovery:** <https://micup.gg/.well-known/fluxer>

This repository adapts the approach of [jubbio-docs-for-llms](https://github.com/saintberat/jubbio-docs-for-llms) for Micup. It consolidates verified endpoints, authentication rules, Gateway opcodes, interaction flows, and Micup-specific features into a single reference to prevent models from inventing Discord API behavior when working with Micup.

The technical reference is written in English to preserve field names, headers, and error codes exactly as documented. The README and usage notes are also provided in English in this version. This repository is not an official Micup publication and does not replace the live documentation.

---

## Table of Contents

| File | Purpose |
| --- | --- |
| [MICUP_DEV_REFERENCE.md](./MICUP_DEV_REFERENCE.md) | Main technical reference covering bots, OAuth2, the HTTP API, Gateway, webhooks, voice, media, and Micup-specific features. |
| [llms.txt](./llms.txt) | Compact index for LLMs, containing links to the live documentation and local files. |
| [llms-full.txt](./llms-full.txt) | Plain-text concatenation of the README and the main reference. Suitable for Claude Projects and ChatGPT Knowledge. |
| [LICENSE](./LICENSE) | AGPL-3.0-or-later license text. |

---

## Why This Repository?

Micup is a relatively new platform, and general-purpose models may not have detailed information about its API in their training data. As a result, models frequently:

- Use Discord's `discord.com/api` URL or the `gateway.discord.gg` Gateway.
- Assume the existence of Gateway intents and privileged intent approval, neither of which is available in Micup.
- Attempt to receive interactions through an HTTP endpoint and implement Ed25519 signature verification.
- Request the `applications.commands` scope.
- Parse Snowflakes as JSON numbers.
- Attempt to call Micup-specific `MicupBOT`, feed, board game, and store routes using bot credentials.

This reference addresses most of these errors at their source.

---

## Quick Facts

| Component | Value |
| --- | --- |
| Discovery | `GET https://micup.gg/.well-known/fluxer` |
| HTTP API | `endpoints.api_public` + `/v1` (live: `https://micup.gg/api/v1`) |
| Gateway | `endpoints.gateway` + `?v=1&encoding=json` (live: `wss://micup.gg/gateway`) |
| Bot authentication | `Authorization: Bot <token>` |
| User session | `Authorization: <token>` (no scheme prefix) |
| OAuth2 | `Bearer <token>`, only on routes that explicitly support it |
| API version | `/v1` only |
| Gateway version | `v=1` only |
| Official Micup SDK | Not documented; use discord.js 14 / discord.py 2 |
| OpenAPI | `GET https://micup.gg/api/v1/openapi.json` |
| Bot catalog | `GET https://micup.gg/api/v1/instance/bot-catalog` |
| Live documentation | <https://micup.gg/docs/> |

**Important:** These values were obtained from the live discovery response on October 26, 2026. Clients must always read `/.well-known/fluxer` rather than hard-coding endpoint URLs.

---

## Using AI Tools

### Cursor / Windsurf / Antigravity / Claude Code

1. Clone or download this repository.
2. Add `MICUP_DEV_REFERENCE.md` to your project.
3. In Cursor, include it in the **Docs** context or project rules.
4. After publishing the repository, you can add the raw URL of `llms.txt` as a remote documentation index.

### Claude Projects / ChatGPT Custom GPT

- Upload `MICUP_DEV_REFERENCE.md` or `llms-full.txt` to the project's knowledge base.
- Use `llms-full.txt` when a large context window is available; use `MICUP_DEV_REFERENCE.md` when working with a shorter context window.

### System Prompt

You can add the following concise instruction to your system prompt when generating Micup-related code:

```text
Treat only the following sources as authoritative for the Micup API:
1. The discovery response at https://micup.gg/.well-known/fluxer
2. The live documentation at https://micup.gg/docs/
3. MICUP_DEV_REFERENCE.md in this project

Micup is not Discord. There are no Gateway intents; interactions are
received through the Gateway, not HTTP; there are no Ed25519 signatures
or PING events; and there is no applications.commands scope. All
Snowflakes are strings, and the API base URL must be read from discovery.
Never invent an undocumented endpoint or field.
```

### Direct API Request

```js
const discovery = await (await fetch('https://micup.gg/.well-known/fluxer')).json();
const api = discovery.endpoints.api_public.replace(/\/$/, '');

const response = await fetch(`${api}/v1/channels/${channelId}/messages`, {
  method: 'POST',
  headers: {
    Authorization: `Bot ${process.env.MICUP_TOKEN}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({content: 'Hello, Micup!'}),
});

console.log(response.status, await response.text());
```

For complete examples and error handling, see [MICUP_DEV_REFERENCE.md](./MICUP_DEV_REFERENCE.md), §16.

---

## Creating Your First Bot

1. Open **Settings → Applications** in Micup.
2. Select **Create Application** and provide a name. A bot account is created alongside the application.
3. Copy the **bot token** immediately. The full token is displayed only in the creation or regeneration response.
4. Invite the bot:

```text
https://micup.gg/oauth2/authorize?client_id=APPLICATION_ID&scope=bot&permissions=3072
```

`3072` is the sum of `VIEW_CHANNEL` (1024) and `SEND_MESSAGES` (2048).

5. Obtain the channel ID from a URL in the following format: `https://micup.gg/channels/SERVER_ID/CHANNEL_ID`.
6. Send a message:

```bash
curl -X POST "https://micup.gg/api/v1/channels/CHANNEL_ID/messages" \
  -H "Authorization: Bot $MICUP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Hello, Micup!"}'
```

For slash commands, buttons, menus, and modals, see **§8 Application Commands and Interactions** in the reference. For Gateway connections, see **§9 Gateway**.

---

## Common Mistakes

| Mistake | Correct Approach |
| --- | --- |
| Using `https://discord.com/api` | Use `endpoints.api_public` + `/v1`. |
| Using `wss://gateway.discord.gg` | Use `endpoints.gateway` + `?v=1&encoding=json`. |
| Sending Gateway intents | Micup has no intents; use `ignored_events` and `Lazy Request`. |
| Setting up an HTTP server to receive interactions | Interactions arrive through the Gateway as `INTERACTION_CREATE` events. |
| Verifying Ed25519 signatures | Micup does not use signatures; the interaction token serves as the credential. |
| Requesting `applications.commands` | `scope=bot` is sufficient. |
| Parsing a Snowflake with `Number` | Keep it as a string and compare it using exact integer-safe semantics. |
| Putting the bot token in a URL | Use `Authorization: Bot <token>`. |
| Calling MicupBOT/feed/store routes with bot credentials | These routes require a user session; bots receive `403 ACCESS_DENIED`. |
| Retrying a `403`/`401` response with the same request | Read the `code` field and correct the credentials or request. |
| Retrying a `429` response without waiting | Wait for the duration specified by `retry_after` / `Retry-After`. |

---

## Scope and Verification

**Verified on October 26, 2026:**

- `GET /.well-known/fluxer` → HTTP 200; live endpoint and feature values retrieved.
- `GET /api/v1/instance/bot-catalog` → HTTP 200; official Micup bot cards retrieved.
- `GET /api/v1/gateway/bot` without credentials → HTTP 401.
- The live documentation homepage, Gateway documentation, and HTTP API documentation were reviewed.
- OpenAPI documentation → 402 paths and 894 schemas.
- Documentation sources, permission registry, Gateway registry, and Micup-specific routes in the Micup source archive were reviewed.

**Not verified (requires credentials or a test account):**

- Logging in with an actual bot, sending messages, and responding to interactions.
- The complete OAuth2 authorization/code/token/refresh flow.
- Voice placement and LiveKit media.
- Board games, store payment flows, referrals, and interest selection.
- Admin API behavior.

These sections document the protocol; they are not an end-to-end live testing report.

---

## Repository Structure and Maintenance

```text
micup-docs-for-llms/
├── README.md
├── MICUP_DEV_REFERENCE.md
├── llms.txt
├── llms-full.txt
└── LICENSE
```

Maintenance guidelines:

- Re-read the `/.well-known/fluxer` response before changing any endpoints.
- Compare the reference against `GET /v1/openapi.json`.
- Regenerate `llms-full.txt` whenever `MICUP_DEV_REFERENCE.md` changes.
- Update the verification date and the live discovery snapshot.

To generate `llms-full.txt`:

```bash
cat README.md MICUP_DEV_REFERENCE.md > llms-full.txt
```

---

## License and Attribution

The documentation content in this repository is distributed under **AGPL-3.0-or-later**. Micup's documentation and source code are derived from the Fluxer project (<https://github.com/fluxerapp/fluxer>) and are distributed under the same license. See [LICENSE](./LICENSE) for details.

The names and logos of "Micup" and "Fluxer" are not covered by the license; the license does not grant trademark rights.

Protocol identifiers such as `X-Fluxer-*` headers, the `/.well-known/fluxer` path, and the `flx_` token prefix have been preserved for wire compatibility. Their presence does not imply an affiliation with Fluxer.

---

## English Summary

This repository is an LLM-oriented technical reference for **Micup** (`micup.gg`), a Turkish chat and voice community platform built on the Fluxer protocol. It consolidates Micup's public HTTP API, Gateway, OAuth2, webhooks, voice, media, and Micup-specific features into a single verified document.

Micup is **Discord-shaped but not Discord**: there are no Gateway intents, no HTTP interaction receiver, no Ed25519 signatures, and no `applications.commands` scope. Always read base URLs from `https://micup.gg/.well-known/fluxer`; never hard-code Discord domains.

Files:

- [`MICUP_DEV_REFERENCE.md`](./MICUP_DEV_REFERENCE.md) — the complete technical reference.
- [`llms.txt`](./llms.txt) — a compact index and links to the live documentation.
- [`llms-full.txt`](./llms-full.txt) — the README and complete reference concatenated as plain text.
- [`LICENSE`](./LICENSE) — AGPL-3.0-or-later.

The reference was verified against Micup's live discovery endpoint and documentation on **October 26, 2026**. It is not an official Micup publication and does not replace the live documentation at <https://micup.gg/docs/>.

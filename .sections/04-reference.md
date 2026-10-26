---

## 14. Discord compatibility and migration

### 14.1 Surface mapping

| Discord concept | Micup equivalent | Change required |
| --- | --- | --- |
| `https://discord.com/api/v10` | `endpoints.api_public` + `/v1` | Discover the base |
| `wss://gateway.discord.gg` | `endpoints.gateway` + `?v=1&encoding=json` | Discover the base and set `v=1` |
| `Authorization: Bot ...` | Same | Keep |
| Intents and privileged intents | None | Remove intent assumptions |
| HTTP interactions endpoint + Ed25519 | Gateway `INTERACTION_CREATE` | Replace the web server |
| `applications.commands` scope | None | Use `scope=bot` |
| User-installable apps | None | Remove `integration_types` |
| Discord CDN | Media Proxy | Use returned URLs |
| `#1234` discriminators | `tag_style`-dependent | Read discovery |
| Message components | Same type numbers | Works, subject to feature keys |
| Modals | Same type numbers | Works, subject to feature keys |
| Components v2 | Feature-gated off by default | Do not rely on it |
| Premium buttons | Not offered | Remove |
| Files in interaction responses | Not offered | Use followups or a normal message |
| Attachment command option | Not offered | Use a string URL or upload flow |

### 14.2 Migration checklist

1. Replace every Discord host with a value from `GET /.well-known/fluxer`.
2. Set REST version to `1` and Gateway `v` to `1`.
3. Set the bot `Authorization` header to `Bot <token>`.
4. For discord.py, replace `DiscordWebSocket.DEFAULT_GATEWAY`; it does not discover.
5. Remove all `intents`/privileged-intent logic. Use `ignored_events` and `Lazy Request` instead.
6. Remove the HTTP interactions listener, public-key verification, and `PING` response.
7. Register commands with `scope=1` and no `applications.commands`.
8. Replace Discord CDN URLs with the Media Proxy URLs returned by the API.
9. Treat every snowflake as a string.
10. Re-check permissions against Micup's bit table. `STREAM` is `1 << 9`; `VIEW_CHANNEL_MEMBERS` is `1 << 54` and feature-gated.
11. Re-check message length and attachment limits from discovery.
12. Re-check interaction timing: 3 seconds, 15 minutes, 20 followups.
13. Re-check component availability: buttons and selects are open; Components v2 is not.
14. Test 401, 403, 404, 429, 4004, 4011, and `INTERACTION_EXPIRED` paths.
15. Remove any token from source, logs, and URLs. Rotate it if it was exposed.

### 14.3 Library quirks

| Library | Quirk | Fix |
| --- | --- | --- |
| discord.js 14 | `rest.api` must be the base **without** `/v1` | Pass `discovery.endpoints.api_public` |
| discord.js 14 | It fetches the Gateway URL from `/v1/gateway/bot` | Ensure the token is valid |
| discord.js 14 | It still requires client-side intents | Provide the bits for the events you handle; Micup ignores the field |
| discord.py 2 | It does not discover the Gateway URL | Set `DiscordWebSocket.DEFAULT_GATEWAY` |
| discord.py 2 | It uses `Route.BASE` for REST | Set it to `{api_public}/v1` |
| discord.py 2 | It may send `with_response=1` | Micup supports it |
| Any library | It may assume Discord CDN | Replace asset URL builders |
| Any library | It may assume `applications.commands` | Remove that scope from invites |

### 14.4 What works unchanged

- Slash command payload shapes for types 1–8 and 10.
- Buttons, link buttons, string/user/role/mentionable/channel selects.
- Modals with text inputs.
- Ephemeral responses via `flags: 64`.
- Message embeds, attachments, reactions, pins, and replies.
- Guild, channel, role, member, invite, and webhook concepts.
- Discord.js and discord.py command builders.

### 14.5 What does not work

- Gateway intents and privileged-intent approval.
- Receiving interactions over HTTP.
- Ed25519 request verification.
- `PING` interactions.
- User-installable applications.
- `applications.commands` scope.
- Context menu (right-click) commands.
- Mentionable and attachment command option types.
- Components v2 by default.
- Premium buttons.
- Files in interaction responses.
- Files or select menus inside modals.

---

## 15. Error and troubleshooting guide

### 15.1 Status semantics

| Status | Meaning | Safe action |
| --- | --- | --- |
| `200` / `201` / `204` | Success | Continue |
| `304` | Not modified | Use the cached representation |
| `400` | Bad request or validation failure | Fix the request; do not retry unchanged |
| `401` | Credential missing, malformed, expired, or revoked | Verify/rotate the credential; do not loop |
| `403` | Valid credential but not allowed | Fix permissions/credential type; do not loop |
| `404` | Unknown route or resource | Fix the ID or route; do not loop |
| `409` | State conflict, duplicate, or consumed token | Read `code`; reconcile state |
| `410` | Expired interaction or resource | Stop answering; send a new interaction |
| `412` | Precondition failed | Re-read the version/ETag and retry once |
| `413` | Payload too large | Reduce files/content/body |
| `429` | Rate limited | Wait `retry_after` and retry once |
| `500` | Internal error | Backoff; report if persistent |
| `502` / `503` / `504` | Upstream or capacity failure | Backoff with jitter |
| `403 GLOBAL_IP_TEMPORARILY_BANNED` | Temporary IP ban | Stop; do not retry |

### 15.2 Common API codes

| Code | Status | Meaning |
| --- | --- | --- |
| `UNAUTHORIZED` | 401 | Missing, malformed, unknown, expired, or revoked credential |
| `INVALID_TOKEN` | 401 | A bot token or bearer does not resolve |
| `MISSING_AUTHORIZATION` | 401 | No credential on a protected route |
| `ACCESS_DENIED` | 403 | Credential type or identity not allowed |
| `MISSING_ACCESS` | 403 | No access to the guild or resource |
| `MISSING_PERMISSIONS` | 403 | Missing permission or hierarchy |
| `MISSING_OAUTH_SCOPE` | 403 | Bearer lacks the required scope |
| `COMMUNICATION_DISABLED` | 403 | Member timed out |
| `ACCOUNT_LIMITED` | 403 | Account limitation blocks the action |
| `TWO_FACTOR_REQUIRED` | 400 | Elevated action needs MFA |
| `SUDO_MODE_REQUIRED` | 403 | Sensitive operation needs a sudo proof |
| `CONTENT_BLOCKED` | 403 | Content policy blocked the value |
| `FEATURE_TEMPORARILY_DISABLED` | 403 | Deployment feature is off |
| `APPLICATION_COMMAND_FEATURE_DISABLED` | 403 | Bot interaction key is off |
| `UNKNOWN_GUILD` | 404 | Guild not found |
| `UNKNOWN_CHANNEL` | 404 | Channel not found or not visible |
| `UNKNOWN_MESSAGE` | 404 | Message not found |
| `UNKNOWN_MEMBER` | 404 | Member not found |
| `UNKNOWN_WEBHOOK` | 404 | Webhook/token pair not found |
| `INVALID_FORM_BODY` | 400 | Field validation failure; inspect `errors` |
| `CANNOT_SEND_EMPTY_MESSAGE` | 400 | Message has nothing sendable |
| `CANNOT_SEND_MESSAGES_TO_USER` | 400 | Recipient DM policy |
| `CANNOT_EXECUTE_ON_DM` | 400 | Route is guild-only |
| `RATE_LIMITED` | 429 | Route or global bucket |
| `GLOBAL_IP_BANNED` | 403 | Permanent IP ban |
| `GLOBAL_IP_TEMPORARILY_BANNED` | 403 | Temporary IP ban |
| `INTERACTION_EXPIRED` | 410 | 3-second or 15-minute window passed |
| `INTERACTION_TOKEN_CONSUMED` | 409 | Interaction already answered |
| `INTERACTION_NONCE_CONFLICT` | 409 | Nonce/idempotency key reused with another body |
| `INTERACTION_OUTCOME_UNKNOWN` | 503 | Retry with the same key |
| `UNSUPPORTED_COMPONENT` | 400 | Component type not offered |
| `INVALID_COMPONENT_VALUE` | 400 | Select value not on the component |
| `MODAL_NOT_OPEN` | 409 | Modal submit without an open modal |
| `APPLICATION_COMMAND_STALE` | 409 | Command changed since load |
| `APPLICATION_COMMAND_BOT_UNAVAILABLE` | 503 | No bot session took the interaction |
| `TABLE_GAME_RANKED_IN_PROGRESS` | 409 | Ranked table blocks a theme/channel change |
| `DISCOVERY_DISABLED` | 400 | Discovery is off on the instance |

### 15.3 Troubleshooting matrix

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `401` on every REST call | Token reset, wrong token, or Discord URL | Re-copy the token; use discovered API base; use `Bot` prefix |
| `403 ACCESS_DENIED` on a normal bot call | The route is user-only | Do not call feed/shop/table/voice-seat routes with a bot |
| `403 MISSING_PERMISSIONS` | Bot lacks `VIEW_CHANNEL`/`SEND_MESSAGES`, or its role is too low | Grant permissions and move the bot role up |
| `404 UNKNOWN_CHANNEL` | Wrong channel ID, wrong guild, or no visibility | Verify IDs from the guild's channel list |
| Commands work but interactions never arrive | Bot has no live Gateway session, or wrong Gateway URL | Check `READY`; verify `DiscordWebSocket.DEFAULT_GATEWAY` in discord.py |
| `410 INTERACTION_EXPIRED` | First response took more than 3 seconds | Defer within 3 seconds, then edit |
| `409 INTERACTION_TOKEN_CONSUMED` | Two code paths answered | Answer once; guard against duplicate handlers |
| Components render but clicks do nothing | `feature_bot_components` off, or bot not the message author | Check discovery limits; write components on the bot's own message |
| Link button opens nothing | URL invalid or blocked | Use `https`, at most 512 chars, no credentials |
| Modal opens but submit fails | Unsupported component or no open modal | Use text inputs only; track modal state |
| Gateway closes immediately with `4012` | Missing `v=1` | Add `?v=1&encoding=json` |
| Gateway closes with `4004` | Invalid token or token mismatch | Re-copy the bot token; do not use a user token |
| Gateway closes with `4011` | More than 2,500 guilds on a shard | Identify with a larger `shard_count` |
| Gateway closes with `4009` | Heartbeats late | Send heartbeats on the advertised interval and answer server requests |
| Gateway closes with `4008` | Payload/connection budget exceeded | Reduce frequency; one session per shard; backoff |
| `429` repeatedly | No backoff or too many shards | Centralize rate limiting; honor `retry_after` |
| Embeds missing | Missing `EMBED_LINKS` | Grant the bit |
| Attachments fail | Missing `ATTACH_FILES` or size limit | Grant the bit; check discovery limits |
| Media URL 403 | Signed URL expired | Re-read the message; refresh URLs |
| `503 SERVICE_UNAVAILABLE` | Instance overloaded | Wait `Retry-After: 1` and retry with backoff |
| discord.js hits Discord | `rest.api` still Discord | Set `rest: {api, version: '1'}` |
| discord.py hits Discord | Gateway URL not replaced | Set `DEFAULT_GATEWAY` from `/v1/gateway/bot` |

### 15.4 Retry policy

- Retry `429` once after `retry_after` / `Retry-After`.
- Retry `500`, `502`, `503`, and `504` with exponential backoff and jitter, at most 3 times.
- Retry `503 INTERACTION_OUTCOME_UNKNOWN` with the **same** `Idempotency-Key`.
- Do not retry `400`, `401`, `403`, `404`, `410`, or a `409` that represents a consumed token.
- On `412`, re-read the current ETag/version and retry once.
- Never rotate a token automatically on `401` unless the operator explicitly configured a rotation flow.
- Never retry a `GLOBAL_IP_TEMPORARILY_BANNED` response.

---

## 16. Minimal recipes

### 16.1 Send a message with Python and the standard library

```python
import getpass
import json
import urllib.error
import urllib.request

token = getpass.getpass("Micup bot token: ").strip()
channel_id = input("Channel ID: ").strip()

if not token or not channel_id.isascii() or not channel_id.isdigit():
    raise SystemExit("A bot token and a numeric channel ID are required.")

try:
    with urllib.request.urlopen(
        "https://micup.gg/.well-known/fluxer", timeout=15
    ) as response:
        discovery = json.load(response)
    api = discovery["endpoints"]["api_public"].rstrip("/")

    request = urllib.request.Request(
        f"{api}/v1/channels/{channel_id}/messages",
        data=json.dumps({"content": "Merhaba, Micup!"}).encode("utf-8"),
        headers={
            "Authorization": f"Bot {token}",
            "Content-Type": "application/json",
        },
        method="POST",
    )
    with urllib.request.urlopen(request, timeout=15) as response:
        message = json.load(response)
    print("Sent message:", message["id"])
except urllib.error.HTTPError as error:
    body = error.read().decode("utf-8", "replace")
    raise SystemExit(f"HTTP {error.code}: {body}")
except urllib.error.URLError:
    raise SystemExit("Could not reach Micup.")
```

### 16.2 Send a message with Node.js

```js
const token = process.env.MICUP_TOKEN;
const channelId = process.env.MICUP_CHANNEL_ID;

const discovery = await (await fetch('https://micup.gg/.well-known/fluxer')).json();
const api = discovery.endpoints.api_public.replace(/\/$/, '');

const response = await fetch(`${api}/v1/channels/${channelId}/messages`, {
  method: 'POST',
  headers: {
    Authorization: `Bot ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({content: 'Merhaba, Micup!'}),
});

if (!response.ok) {
  console.error(response.status, await response.text());
  process.exit(1);
}

console.log(await response.json());
```

### 16.3 Minimal discord.js slash command

```js
import {
  Client,
  Events,
  GatewayIntentBits,
  MessageFlags,
  SlashCommandBuilder,
} from 'discord.js';

const discovery = await (await fetch('https://micup.gg/.well-known/fluxer')).json();
const api = discovery.endpoints.api_public.replace(/\/$/, '');

const client = new Client({
  intents: [GatewayIntentBits.Guilds],
  rest: {api, version: '1'},
  ws: {version: 1},
});

client.once(Events.ClientReady, async (readyClient) => {
  await readyClient.application.commands.create(
    new SlashCommandBuilder()
      .setName('ping')
      .setDescription('Replies with Pong!')
      .toJSON(),
  );
  console.log(`Logged in as ${readyClient.user.username}`);
});

client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isChatInputCommand()) return;
  if (interaction.commandName === 'ping') {
    await interaction.reply({content: '🏓 Pong!', flags: MessageFlags.Ephemeral});
  }
});

client.login(process.env.MICUP_TOKEN);
```

Global command propagation can take time. For immediate testing, register a guild command with `guilds/{guild_id}/commands` or `applicationGuildCommands`.

### 16.4 Minimal discord.py slash command

```python
import json
import os
import urllib.request

import discord
import discord.gateway
import discord.http
import yarl
from discord import app_commands

TOKEN = os.environ["MICUP_TOKEN"]

with urllib.request.urlopen("https://micup.gg/.well-known/fluxer", timeout=15) as response:
    API = json.load(response)["endpoints"]["api_public"].rstrip("/")

discord.http.INTERNAL_API_VERSION = 1
discord.http.Route.BASE = f"{API}/v1"

request = urllib.request.Request(
    f"{API}/v1/gateway/bot",
    headers={"Authorization": f"Bot {TOKEN}"},
)
with urllib.request.urlopen(request, timeout=15) as response:
    gateway = json.load(response)["url"]
discord.gateway.DiscordWebSocket.DEFAULT_GATEWAY = yarl.URL(gateway)

client = discord.Client(intents=discord.Intents.default())
tree = app_commands.CommandTree(client)


@tree.command(name="ping", description="Replies with Pong!")
async def ping(interaction: discord.Interaction):
    await interaction.response.send_message("🏓 Pong!", ephemeral=True)


@client.event
async def on_ready():
    await tree.sync()
    print(f"Logged in as {client.user}")


client.run(TOKEN)
```

### 16.5 Execute a webhook

```bash
curl -X POST \
  "https://micup.gg/api/v1/webhooks/$WEBHOOK_ID/$WEBHOOK_TOKEN?wait=true" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Build #128 passed",
    "username": "CI",
    "allowed_mentions": {"parse": []}
  }'
```

### 16.6 Register a global command with fetch

```js
const token = process.env.MICUP_TOKEN;
const applicationId = process.env.MICUP_APPLICATION_ID;

const discovery = await (await fetch('https://micup.gg/.well-known/fluxer')).json();
const api = discovery.endpoints.api_public.replace(/\/$/, '');

const response = await fetch(`${api}/v1/applications/${applicationId}/commands`, {
  method: 'PUT',
  headers: {
    Authorization: `Bot ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify([
    {
      name: 'ping',
      description: 'Replies with Pong!',
      dm_permission: false,
    },
  ]),
});

console.log(response.status, await response.text());
```

### 16.7 Read a guild's channels

```js
const response = await fetch(`${api}/v1/guilds/${guildId}/channels`, {
  headers: {Authorization: `Bot ${token}`},
});
if (!response.ok) throw new Error(`${response.status} ${await response.text()}`);
const channels = await response.json();
console.log(channels.filter((channel) => channel.type === 0)); // text channels
```

---

## 17. Source map and verification

### 17.1 Live documentation

| Topic | URL |
| --- | --- |
| Documentation home | <https://micup.gg/docs/> |
| Authentication | <https://micup.gg/docs/authentication/> |
| HTTP API | <https://micup.gg/docs/http-api/> |
| Instance discovery and limits | <https://micup.gg/docs/http-api/instance/> |
| OAuth2 | <https://micup.gg/docs/http-api/oauth2/> |
| Applications | <https://micup.gg/docs/http-api/applications/> |
| Permissions | <https://micup.gg/docs/http-api/permissions/> |
| Messages | <https://micup.gg/docs/http-api/messages/> |
| Application commands | <https://micup.gg/docs/http-api/application-commands/> |
| Interactions | <https://micup.gg/docs/http-api/interactions/> |
| Webhooks | <https://micup.gg/docs/http-api/webhooks/> |
| Gateway overview | <https://micup.gg/docs/gateway/overview/> |
| Gateway commands | <https://micup.gg/docs/gateway/commands/> |
| Gateway events | <https://micup.gg/docs/gateway/events/> |
| Gateway opcodes and close codes | <https://micup.gg/docs/gateway/opcodes-and-close-codes/> |
| Voice | <https://micup.gg/docs/voice/> |
| Voice seats | <https://micup.gg/docs/http-api/voice-seats/> |
| Media Proxy | <https://micup.gg/docs/media-proxy/overview/> |
| Uploads | <https://micup.gg/docs/topics/uploads/> |
| Turkish bot quick start | <https://micup.gg/docs/bots/hizli-baslangic/> |
| Turkish interactions guide | <https://micup.gg/docs/bots/etkilesimler/> |
| MicupBOT | <https://micup.gg/docs/http-api/guild-micupbot/> |
| Micup Roster | <https://micup.gg/docs/http-api/guild-roster-events/> |
| Feed | <https://micup.gg/docs/http-api/feed/> |
| Table games | <https://micup.gg/docs/http-api/table-games/> |
| Shop | <https://micup.gg/docs/http-api/shop/> |
| Badges | <https://micup.gg/docs/http-api/users/badges/> |
| Referral | <https://micup.gg/docs/http-api/referral/> |
| Interest onboarding | <https://micup.gg/docs/http-api/interest-onboarding/> |
| Admin API | <https://micup.gg/docs/admin-api/> |

### 17.2 Machine-readable endpoints

```text
GET https://micup.gg/.well-known/fluxer
GET https://micup.gg/api/v1/openapi.json
GET https://micup.gg/api/v1/instance/bot-catalog
GET https://micup.gg/api/v1/gateway/bot
```

### 17.3 What was verified

On **26 October 2026**:

- `GET https://micup.gg/.well-known/fluxer` returned HTTP 200 with the endpoint values and feature flags in §1.2.
- `GET https://micup.gg/api/v1/instance/bot-catalog` returned HTTP 200 with the official Micup bot cards.
- `GET https://micup.gg/api/v1/gateway/bot` without a credential returned HTTP 401, confirming the route is protected.
- The live documentation returned HTTP 200 for the home page and the Gateway overview.
- The deployment's OpenAPI document was reachable and described 402 paths and 893 schemas.
- The Micup source archive contained the documentation source, the permission registry, the Gateway registry, the OpenAPI document, and the Micup-specific routes referenced here.

### 17.4 What was not verified

These require credentials, test accounts, or a media client and were **not** executed:

- Logging in a real bot and sending a message.
- Receiving a real `INTERACTION_CREATE` and answering a slash command.
- A complete OAuth2 authorize/consent/token/refresh cycle.
- Voice placement and LiveKit media.
- Table games, shop checkout, referral rewards, and interest onboarding.
- Admin API behavior.

Treat the relevant sections as protocol documentation, not as a live end-to-end test report.

### 17.5 Maintenance

- Re-read `/.well-known/fluxer` before changing any endpoint.
- Diff `GET /v1/openapi.json` against this document when routes change.
- Regenerate `llms-full.txt` after editing `MICUP_DEV_REFERENCE.md`.
- Update the verification date and §1.2 snapshot when the live instance changes.

---

## 18. License and attribution

This reference describes Micup's public developer API. Micup's documentation and source are derived from the Fluxer project (<https://github.com/fluxerapp/fluxer>) under the **GNU Affero General Public License v3.0 or later** (AGPL-3.0-or-later).

The documentation content in this repository is distributed under the same license. See [LICENSE](./LICENSE).

"Micup" and "Fluxer" names and logos are not covered by the license; the license grants no trademark rights.

Protocol identifiers such as `X-Fluxer-*` headers, `/.well-known/fluxer`, and the `flx_` token prefix are retained for wire compatibility. They are not a claim of affiliation with Fluxer.

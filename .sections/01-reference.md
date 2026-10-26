# Micup Developer Reference for LLMs and AI Agents

> **Version:** 1.0
> **Verified against live Micup services:** 26 October 2026
> **Live documentation:** <https://micup.gg/docs/>
> **Instance discovery:** <https://micup.gg/.well-known/fluxer>
> **Scope:** public bot, application, OAuth2, HTTP API, Gateway, webhook, voice, media, and Micup-specific platform surfaces.
> **Not covered:** Admin API internals. See <https://micup.gg/docs/admin-api/>.

This document is a compact, machine-readable ground truth for AI coding assistants that write Micup bots, integrations, and clients. It is derived from Micup's live developer documentation and the Micup source archive. Where this document and the live documentation disagree, the live documentation and the deployment's discovery document win.

Normative keywords **MUST**, **MUST NOT**, **SHOULD**, and **MAY** describe client requirements. Field names, type numbers, routes, and error codes are literal. All identifiers are decimal strings unless stated otherwise.

---

## 0. Read this first

Micup is **not Discord**, even though its API is deliberately Discord-shaped and is built on the Fluxer protocol. A bot copied from a Discord tutorial usually needs four changes:

1. Point the REST base at `endpoints.api_public` from discovery and use `/v1`.
2. Set the Gateway version to `v=1` at the discovered `endpoints.gateway`.
3. Use `Authorization: Bot <token>` for bot REST calls.
4. Stop relying on Discord-only features: there are **no Gateway intents**, **no HTTP endpoint that receives interactions**, **no Ed25519 interaction signatures**, **no `PING` interaction**, and **no `applications.commands` OAuth2 scope**.

Other rules that prevent most hallucinations:

- **Never hard-code `discord.com`, `discordapp.com`, `cdn.discordapp.com`, or `gateway. discord.gg`.** Read the five public base URLs from `GET /.well-known/fluxer`.
- **Never parse a snowflake as a JSON number.** IDs are 64-bit decimal strings.
- **Never match on the human-readable `message` field.** Match on `code`.
- **Never put a bot token, webhook token, interaction token, or signed media URL in logs, source code, URLs, analytics, or error reports.**
- **Do not invent Micup SDK packages.** No official Micup SDK is documented. Use a Discord-compatible library (discord.js 14 or discord.py 2) or the raw HTTP/WebSocket protocol.
- **Do not assume a feature is enabled.** Read `features`, `limits`, and per-route feature flags. Several Micup features are deployment-gated or temporarily disabled.
- **The deployment is one instance.** Micup's live instance is at `micup.gg`, but the protocol is designed for multiple deployments. A value that is true on one deployment can be false on another.

If you only remember one thing: **discover first, then call**. The discovery document is the entry point of the entire API.

---

## 1. Platform overview

Micup (micup.gg) is a chat and voice community platform with a Turkish-first product experience. Users join communities (called **guilds** in the API), talk in text and voice channels, send direct messages, publish feed posts, play table games in 3D voice rooms, and buy cosmetic and premium products.

### 1.1 Protocol surfaces

| Surface | Purpose | Base |
| --- | --- | --- |
| HTTP API | Read and mutate resources | `endpoints.api_public` + `/v1` |
| Gateway | Persistent WebSocket for events and a small command set | `endpoints.gateway` with `?v=1&encoding=json` |
| Media Proxy | Attachments, images, themes, signed external media | `endpoints.media` |
| Upload relay | Capability-authorised uploads | `endpoints.upload_relay` |
| Admin API | Privileged instance management | `endpoints.admin` + `/v1/admin` |
| Developer docs | Human documentation | <https://micup.gg/docs/> |
| OpenAPI | Machine-readable HTTP schema | `GET /v1/openapi.json` |

The Media Proxy and the Admin API each have their own authentication. Nothing negotiated on one surface applies to another.

### 1.2 Live discovery snapshot

These are the values Micup published on **26 October 2026**. Treat them as examples, not constants.

| Field | Value |
| --- | --- |
| `api_code_version` | `1` |
| `endpoints.api_public` | `https://micup.gg/api` |
| `endpoints.api_client` | `https://micup.gg/api` |
| `endpoints.gateway` | `wss://micup.gg/gateway` |
| `endpoints.media` | `https://media.micup.gg` |
| `endpoints.upload_relay` | `https://micup.gg/media` |
| `endpoints.static_cdn` | `https://static.micup.gg` |
| `endpoints.webapp` | `https://micup.gg` |
| `endpoints.invite` | `https://micup.gg/invite` |
| `endpoints.gift` | `https://micup.gg/gift` |
| `endpoints.admin` | `https://admin.micup.gg` |
| `features.voice_enabled` | `true` |
| `features.premium_enabled` | `true` |
| `features.stripe_enabled` | `false` |
| `features.self_hosted` | `true` |
| `features.presigned_attachment_uploads` | `true` |
| `features.emails_enabled` | `true` |
| `features.gift_codes_enabled` | `true` |
| `registration.mode` | `open` |
| `features.account_identity` | `email` |
| `features.tag_style` | `none` |
| `captcha.provider` | `turnstile` |

A deployment can change any of these. The live discovery document may also include fields that this snapshot omits.

### 1.3 What "Discord-compatible" means here

| Area | Compatible? | Notes |
| --- | --- | --- |
| Route shapes and field names | Mostly | The API follows Discord's naming closely. |
| Application command types 1–8, 10 | Yes | Types 9 (mentionable) and 11 (attachment) are rejected. |
| Message components (buttons, selects, modals) | Yes | Type numbers match Discord's. |
| Discord.js 14 / discord.py 2 | Yes, with configuration | Set the API base and version, and the Gateway URL for discord.py. |
| Gateway intents | **No** | Micup has no intents field, no privileged intents, and no intent close code. |
| Receiving interactions over HTTP | **No** | Interactions arrive over the Gateway. |
| Interaction signing / `PING` | **No** | No Ed25519, no public key, no `PING`. |
| `applications.commands` scope | **No** | Installing a bot uses `scope=bot`; commands need no separate scope. |
| User-installable apps | **No** | `integration_types` is accepted and discarded. |
| Discord CDN URLs | **No** | Attachments and assets are served by the Media Proxy. |

### 1.4 The bot account model

- A **user account** owns an **application**.
- Creating an application creates exactly **one bot account**.
- The bot's user ID **equals** its application ID.
- The bot token is shown **once** at creation or reset. It cannot be read back later.
- A bot is authenticated on REST with `Authorization: Bot <token>`.
- A bot is authenticated on the Gateway by sending the raw token in `Identify`, with no `Bot ` prefix.
- A bot must be a **member of a guild** before it can act there.

---

## 2. Shared conventions

### 2.1 Snowflakes

A snowflake is a 64-bit identifier for a user, guild, channel, message, role, attachment, application, or other resource. It is unique within one deployment and is never reused after deletion. Separate deployments can issue the same numeric value, so scope every snowflake to its deployment.

| Field | Bits | Meaning |
| --- | --- | --- |
| Timestamp | 62–22 | Milliseconds since `2015-01-01T00:00:00.000Z` |
| Worker ID | 21–12 | Unsigned allocator worker, `0`–`1023` |
| Sequence | 11–0 | Unsigned per-worker sequence, `0`–`4095` |

Extract the approximate creation time:

```text
timestamp_ms = (snowflake >> 22) + 1420070400000
```

Wire rules:

- A snowflake is an **unsigned decimal string** in every JSON body, path parameter, query parameter, and Gateway payload.
- Never parse it as a JSON number. Values exceed the exact integer range of a JSON double.
- Compare snowflakes as integers, not as text. Decimal strings of different lengths do not sort correctly as text.
- An HTTP request accepts `0`, or a non-zero digit followed by further digits. Leading zeros, signs, decimal points, and non-digit characters are rejected.
- A Gateway command accepts a non-zero digit followed by further digits, as a string or as a positive JSON integer. `0`, leading zeros, signed values, negative values, and values above `9223372036854775807` are rejected.

Snowflake-ordered collections accept `before`, `after`, or `around` cursors. `before` selects IDs lower than the cursor; `after` selects IDs higher; `around` selects a window centred on and including the cursor.

### 2.2 Field notation

| Field | Type | Meaning |
| --- | --- | --- |
| `name` | `string` | Required; cannot be `null` |
| `name?` | `string` | Optional; cannot be `null` when present |
| `name` | `?string` | Required; can be `null` |
| `name?` | `?string` | Can be omitted or set to `null` |

`array[type]` is an array of the named type. `map[key, value]` is a JSON object keyed by the first type with values of the second type. An omitted field is not the same as a field set to `null`; each operation defines the difference.

### 2.3 Requests

- Send JSON bodies with `Content-Type: application/json`.
- OAuth2 token operations accept `application/x-www-form-urlencoded` or `multipart/form-data`.
- Message creation, message modification, and webhook execution also accept multipart bodies.
- Responses are UTF-8 JSON unless an operation states otherwise.
- Trailing slashes are significant.
- An unknown path or unsupported method returns `404 NOT_FOUND` with no `Allow` header.
- Every `GET` route supports `HEAD`, returning the same status and headers with no body. Routes without `GET` do not support `HEAD`.
- Empty values are normalised before validation: an empty string becomes `null`, and an empty nested object becomes `null`. Message creation, message modification, and webhook execution are exceptions and preserve empty values.
- An overloaded instance returns `503 SERVICE_UNAVAILABLE` with `Retry-After: 1`. Wait before retrying.

### 2.4 Standard request headers

| Header | Purpose |
| --- | --- |
| `Authorization` | The single credential: `Bot <token>`, `Bearer <token>`, a raw user session token, or `Admin <key>` |
| `Content-Type` | Body media type; include the multipart boundary |
| `Accept-Language` | Selects the locale for an error `message` when the caller has no account locale |
| `X-Audit-Log-Reason` | Free-text reason recorded on the audit log entry; at most 512 characters after normalisation |
| `X-Fluxer-Sudo-Mode-JWT` | A short-lived sudo proof for sensitive account operations |
| `X-Captcha-Token` | A solved CAPTCHA challenge on gated account operations |
| `X-Request-ID` | Client-chosen correlation ID, echoed unchanged; never send a secret |
| `X-Fluxer-Features` | Comma-separated feature declarations for feature-gated permission bits |
| `Origin` | Browser origin, used for CORS and the same-host origin check on mutations |

The `X-Fluxer-*` header prefix is a protocol identifier inherited from Fluxer. It is not branding and must not be renamed.

### 2.5 Standard response headers

| Header | Meaning |
| --- | --- |
| `X-Fluxer-Version` | Build version of the serving instance; `dev` when unstamped |
| `X-Request-ID` | The request ID, generated or echoed |
| `X-Fluxer-Sudo-Mode-JWT` | A new sudo token after a successful sudo proof |
| `Content-Type` | Response media type; absent when there is no body |
| `Cache-Control` | `no-cache` unless the operation sets its own directive |
| `Retry-After` | Whole seconds to wait on a rate-limit, slowmode, lock, or in-flight denial |
| `X-RateLimit-Limit` | Maximum requests the route bucket admits at once |
| `X-RateLimit-Remaining` | Requests still admitted by the route bucket |
| `X-RateLimit-Reset` | Unix seconds when the whole route bucket is available again |
| `X-RateLimit-Reset-After` | Seconds until the whole route bucket is available |
| `X-RateLimit-Bucket` | Stable 16-character bucket identifier |
| `X-RateLimit-Scope` | `user`, `shared`, or `global` on a denial |
| `X-RateLimit-Global` | `true` on a global denial |

Successful responses include rate-limit headers only when the credential resolves to a bot, or when the request has no account and the route is a token webhook route. Do not depend on their presence.

### 2.6 Errors

Every HTTP API and Admin API failure uses one envelope:

```json
{
  "code": "MISSING_PERMISSIONS",
  "message": "You do not have permission to do this."
}
```

Rules:

- **Match `code`, never `message`.** `message` is localised and can change wording without notice.
- An HTTP status alone is not enough; many codes share one status.
- A validation failure uses status `400` and adds an `errors` array. Each element has `path`, `message`, and optionally a stable `code`. `path` is dot-joined, such as `embeds.0.title`; a top-level failure uses `root`.
- A rate-limit denial adds `global` and `retry_after`.
- A missing OAuth2 scope adds `required_scope`.
- An OAuth2 protocol failure at the OAuth2 resource uses the RFC 6749 shape `{ "error": "...", "error_description": "..." }` instead of this envelope.

Status fallbacks when no specific code is registered:

| Status | Fallback code |
| --- | --- |
| 400 | `BAD_REQUEST` |
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 405 | `METHOD_NOT_ALLOWED` |
| 409 | `CONFLICT` |
| 410 | `GONE` |
| 500 | `INTERNAL_SERVER_ERROR` |
| 501 | `NOT_IMPLEMENTED` |
| 502 | `BAD_GATEWAY` |
| 503 | `SERVICE_UNAVAILABLE` |
| 504 | `GATEWAY_TIMEOUT` |
| other | `GENERAL_ERROR` |

Authentication rules:

- `401 UNAUTHORIZED` means the credential is missing, malformed, unknown, expired, or revoked.
- `403 ACCESS_DENIED` means the credential is valid but the operation is not allowed for that identity, or the operation does not accept that credential type.
- A bot cannot use a user-only operation. It receives `403 ACCESS_DENIED`.
- A user-only Authentication operation that resolves a bot account returns `403 BOT_USER_AUTH_ENDPOINT_ACCESS_DENIED`.
- Repeated bad credentials or malformed requests can trigger an IP ban. A temporary ban returns `403 GLOBAL_IP_TEMPORARILY_BANNED` for 24 hours. Stop retrying a rejected credential.

### 2.7 Rate limits

Micup uses a per-route leaky bucket plus one global bucket.

- Authenticated requests are keyed by account and credential kind. A bot token, a user session, an OAuth2 bearer application, and an Admin API key each draw separate allowances.
- Unauthenticated requests are keyed by client IP, exactly for IPv4 and by `/64` for IPv6.
- The default global allowance is **50 requests per second**. Accounts with `HIGH_GLOBAL_RATE_LIMIT` get 1,200 requests per second. `RATE_LIMIT_BYPASS` exempts an account from route and global buckets.
- Some operations enforce additional in-handler limits that `RATE_LIMIT_BYPASS` does not exempt.
- Every bucket drains continuously; a client can retry as soon as enough capacity has refilled.
- A denial is `429` with:

```json
{
  "code": "RATE_LIMITED",
  "message": "You're being rate limited.",
  "global": false,
  "retry_after": 0.428
}
```

- `retry_after` is the delay before one further request is admitted. `X-RateLimit-Reset` reports when the bucket is completely full again; the two instants differ.
- On a **global** denial of a non-bot user session, Micup revokes the session token before writing the 429. Bot tokens, OAuth2 access tokens, and Admin API keys are not revoked this way. A route-bucket denial never revokes a credential.
- A 429 can hide a 401 or 403; check `code` before refreshing credentials.
- Read `Retry-After` or `retry_after` and wait. Do not retry a denied request in a loop.

### 2.8 Deployment limits

Limits are **not constants**. Read the `limits` object in discovery and evaluate its rules for the account and guild context.

Important defaults and floors:

| Limit | Default / floor |
| --- | --- |
| `max_message_length` | 2,000 characters; a bot or webhook author gets at least 4,000 |
| `max_embeds_per_message` | 10 |
| `max_attachments_per_message` | 10 |
| Application commands per scope | 100 |
| Options per command or subcommand | 25 |
| Choices per option | 25 |
| Serialised command size | 32,768 bytes |
| Serialised command set | 524,288 bytes |
| Bulk delete messages | 1–100 per request |
| Action rows per message | 5 |
| Buttons per action row | 5 |
| Stickers per message | 3 |

`limits.rules` is an ordered list of rules. Clients start from each key's default, evaluate rules from least specific to most specific, and replace or raise values as the document describes. The server still applies its own authoritative bound on each request.

### 2.9 OpenAPI

`GET /v1/openapi.json` returns the deployment's OpenAPI 3.1 document. It had **402 paths** and **893 component schemas** at verification time. Its server URL is the deployment's `api_client` endpoint followed by `/v1`; every path is relative to that URL.

Use the OpenAPI document as the final authority for field names, request bodies, status codes, and operation IDs. This reference explains the protocol; OpenAPI enumerates the routes.

---

## 3. Authentication and credentials

### 3.1 Credential schemes

| Value | Name | Accepted on |
| --- | --- | --- |
| `Bot <token>` | Bot token | Bot REST operations |
| `Bearer <token>` | OAuth2 access token | Operations that explicitly opt into OAuth2, with the required scope |
| `<token>` | User session token | User operations; sent with **no** scheme prefix |
| `Admin <token>` | Admin API key | Routes below `/v1/admin` only |

Credential namespaces are distinct. A bot token cannot be used as an OAuth2 bearer, a session token cannot be used as a bot token, and an upload capability or signed media path is not an API credential.

Scheme prefixes are case-sensitive, with two exceptions: `GET /v1/gateway/bot` and `GET /v1/applications/@me` accept case-insensitive prefixes. `GET /v1/applications/@me` requires `Bot` specifically.

### 3.2 Bot tokens

Format:

```text
<application_id>.<secret>
```

- `application_id` is a decimal snowflake.
- `secret` is 32 random bytes encoded as base64url.
- The full token is returned only by application creation and bot-token reset.
- A preview cannot authenticate.
- Rotating a bot token immediately invalidates the old token and **ends every Gateway session the bot holds**.
- A bot token is **not** bounded by the 100-session-per-user limit; it is bounded by the guild ceiling and Gateway admission rules.

Use it on REST:

```http
Authorization: Bot 1557753609842589699.D3f...
```

Use it on the Gateway **without** the prefix:

```json
{
  "op": 2,
  "d": {
    "token": "1557753609842589699.D3f...",
    "properties": {"os": "Linux", "browser": "my-bot", "device": "server"}
  }
}
```

### 3.3 User session tokens

A user session token authenticates an ordinary account. It is issued by login, registration, or session exchange and is sent **raw**:

```http
Authorization: flx_...
```

A value of `Bearer flx_` followed by 36 alphanumerics is also treated as a user session token. This is an exception, not the general rule.

User session tokens are for user-only routes. A bot must not use one, and a bot token cannot call a user-only route.

### 3.4 OAuth2 access tokens

An OAuth2 access token belongs to an account and is limited by its scopes. Use `Bearer <token>`.

- Only operations that explicitly support OAuth2 accept a bearer token.
- A missing scope returns `403 MISSING_OAUTH_SCOPE` with `required_scope`.
- A bearer-only operation rejects a session token, bot token, or Admin key with `401 UNAUTHORIZED`.
- Access tokens live **7 days** (604,800 seconds). Refresh tokens live **30 days**.
- A refresh consumes the presented refresh token and rotates the pair. If the response is lost, restart the authorization code grant.
- Only `authorization_code` and `refresh_token` grants exist. There is no `client_credentials` grant and no token unbound from a user.

### 3.5 Admin API keys

Admin API keys authenticate as their creator on routes below `/v1/admin`. Permissions are the intersection of the key's ACLs and the creator's ACLs. An admin request also requires the account's `admin:authenticate` ACL or a wildcard.

Bots cannot use the Admin API. This reference does not document its routes.

### 3.6 Sudo mode

Sudo mode is a short-lived proof that the account recently re-verified a credential.

- The proof is valid for **5 minutes**.
- Present it in `X-Fluxer-Sudo-Mode-JWT`.
- An invalid, expired, or account-mismatched token behaves like a missing one.
- Micup issues a token only to an account with a TOTP secret or a registered WebAuthn credential. A password-only account re-verifies per operation.
- A bot account satisfies sudo mode immediately.
- Revoking the session that obtained a proof does not revoke the proof before it expires.

Sudo mode is irrelevant to ordinary bot messaging. It matters for sensitive application and account operations.

### 3.7 Gateway authentication

- Send the raw bot or user token in `Identify`.
- Do **not** send an `Authorization` header on the Gateway.
- An invalid or revoked token closes with `4004` and reason `Invalid token`.
- A missing `token` closes with `4002` and reason `Invalid identify payload`.
- `GET /v1/gateway/bot` checks only the token's shape. A 200 does not prove the token is valid.

### 3.8 Credential safety

- Treat every bot token, client secret, webhook token, interaction token, upload capability, and signed media URL as a bearer credential.
- Never log, echo, commit, or send these values to third parties.
- Never put an `Authorization` credential in a URL.
- A webhook execution URL is a secret for its whole lifetime.
- An interaction token can answer, edit, and delete that interaction's responses for 15 minutes with no other credential.
- A signed media URL can be read by anyone who holds it until it expires.
- Use TLS. Rotate a bot token immediately after any suspected exposure.

---

## 4. Applications, bots, and OAuth2

### 4.1 Create an application

In the Micup web app:

1. Open **Ayarlar → Uygulamalar** (Settings → Applications).
2. Choose **Uygulama oluştur** (Create application).
3. Enter a name. Micup creates the application and its bot account together.
4. Copy the **bot token** immediately. It is shown once.
5. Optionally set the bot's username, avatar, banner, and bio under the application's bot settings.
6. Use the **OAuth2 URL builder** to create an invite link with `scope=bot` and the permissions you need.

The same operation over REST:

```http
POST /v1/oauth2/applications
Authorization: <user session token>
Content-Type: application/json

{
  "name": "My Micup Bot",
  "redirect_uris": ["https://example.com/callback"],
  "bot_public": true,
  "bot_require_code_grant": false
}
```

Rules:

- The caller must be a claimed account: it needs a password, a linked SSO identity, or a non-bot account.
- A user can own at most **25 applications**.
- The route is limited to **10 creations per hour per user**.
- A CAPTCHA may be required.
- `redirect_uris` accepts at most **10** absolute URIs. HTTPS is required except for `localhost`, `.localhost`, IPv4 literals, and bracketed IPv6 literals.
- The response contains the application, the initial `client_secret`, and the bot object with its `token`.
- A blocked name or redirect URI returns `403 CONTENT_BLOCKED`.
- No bot username may be allocatable; that returns `500 BOT_USER_GENERATION_FAILED`.

The bot account:

- ID equals the application ID.
- Username is 1–32 ASCII letters, digits, or underscores.
- Discriminator is always four decimal digits.
- Starts with no avatar, banner, bio, or global display name.
- Gets the animated avatar and animated banner entitlements automatically.
- Supports two bot flags:

| Value | Flag | Meaning |
| --- | --- | --- |
| `16` (`1 << 4`) | `FRIENDLY_BOT` | Accepts friend requests |
| `32` (`1 << 5`) | `FRIENDLY_BOT_MANUAL_APPROVAL` | Requires manual approval for friend requests |

### 4.2 Application and bot routes

| Method and path | Credential | Purpose |
| --- | --- | --- |
| `GET /v1/users/@me/applications` | User | List owned applications |
| `POST /v1/oauth2/applications` | User | Create application and bot |
| `GET /v1/oauth2/applications/{id}` | Owner user | Read one application |
| `PATCH /v1/oauth2/applications/{id}` | Owner user | Update name, redirect URIs, public-bot settings |
| `DELETE /v1/oauth2/applications/{id}` | Owner user + sudo | Delete the application |
| `GET /v1/oauth2/applications/@me` | Bot | Read the application that owns the token |
| `GET /v1/applications/@me` | Bot (`Bot` required) | Read the bot application |
| `GET /v1/oauth2/applications/{id}/public` | None | Read the public application card |
| `PATCH /v1/oauth2/applications/{id}/bot` | Owner user | Update bot profile |
| `POST /v1/oauth2/applications/{id}/bot/reset-token` | Owner user + MFA/sudo | Rotate the bot token |
| `POST /v1/oauth2/applications/{id}/client-secret/reset` | Owner user + MFA/sudo | Rotate the client secret |
| `GET /v1/instance/bot-catalog` | None | List official and verified community bots |

`GET /v1/instance/bot-catalog` returns two sections:

- `micup`: official bots, in the same order and with the same invite permissions as `features.official_apps` in discovery.
- `community`: public, verified, non-deleted, non-disabled, non-banned community bots, in the operator's order.

The route supports `ETag` / `If-None-Match` and is limited to 60 requests per minute.

### 4.3 Invite a bot

Browser invite URL:

```text
https://micup.gg/oauth2/authorize?client_id=APPLICATION_ID&scope=bot&permissions=3072
```

Pre-select a guild and hide the guild picker:

```text
https://micup.gg/oauth2/authorize?client_id=APPLICATION_ID&scope=bot&permissions=3072&guild_id=GUILD_ID&disable_guild_select=true
```

- `permissions` is a **decimal** permission mask, not a hex value.
- `scope=bot` installs the bot. There is no `applications.commands` scope.
- The inviter needs permission to manage the guild and grant the requested permissions.
- The bot must be a guild member before it can read or write there.
- Do not put the bot token in an invite URL.
- `bot_public=true` lets any eligible user install the bot. `bot_public=false` restricts installation to the owner.
- `bot_require_code_grant=true` requires the OAuth2 authorization code flow before installation.

A minimal invite needs only:

| Permission | Bit | Decimal |
| --- | --- | --- |
| `VIEW_CHANNEL` | `1 << 10` | 1,024 |
| `SEND_MESSAGES` | `1 << 11` | 2,048 |
| Combined | | **3,072** |

### 4.4 OAuth2 scopes

| Scope | Grants |
| --- | --- |
| `identify` | Read the authorized user's identity; required for `GET /v1/oauth2/userinfo` and the reduced bearer form of `GET /v1/users/@me` |
| `email` | Include the user's email address and verification state; only meaningful together with `identify` |
| `guilds` | Read the user's guild list and guild details |
| `connections` | Read the user's connections |
| `bot` | Install the application's bot into a guild or group DM; grants no bearer access by itself |

An unrecognized scope is rejected when the authorization request is completed. A repeated scope is granted once. Compare scopes as a set, never by order.

### 4.5 Authorization code grant

1. Send the user to the authorization endpoint:

```text
GET https://micup.gg/api/v1/oauth2/authorize
  ?client_id=APPLICATION_ID
  &response_type=code
  &scope=identify%20guilds
  &redirect_uri=https%3A%2F%2Fexample.com%2Fcallback
  &state=RANDOM_STATE
  &code_challenge=PKCE_CHALLENGE
  &code_challenge_method=S256
```

2. The user approves. Micup redirects to the registered `redirect_uri` with `code` and the unchanged `state`.
3. Exchange the code:

```http
POST https://micup.gg/api/v1/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTHORIZATION_CODE
&redirect_uri=https%3A%2F%2Fexample.com%2Fcallback
&client_id=APPLICATION_ID
&client_secret=CLIENT_SECRET
&code_verifier=PKCE_VERIFIER
```

4. The response is:

```json
{
  "access_token": "...",
  "token_type": "Bearer",
  "expires_in": 604800,
  "refresh_token": "...",
  "scope": "identify guilds"
}
```

5. Refresh:

```http
POST https://micup.gg/api/v1/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=REFRESH_TOKEN
&client_id=APPLICATION_ID
&client_secret=CLIENT_SECRET
```

Details that matter:

- Authorization codes are single-use and expire after **10 minutes**.
- `redirect_uri` must match the URI recorded on the code exactly, including trailing slash and encoding.
- Client authentication uses HTTP Basic or the `client_id` / `client_secret` form fields. Supplying both is accepted on the token route.
- PKCE is supported with `S256` or `plain`. If the code has a challenge, the verifier is required.
- A refresh consumes the presented refresh token and rotates the pair. Old access tokens remain valid until they expire.
- A replayed code, an expired code, a wrong redirect URI, or a wrong verifier returns an OAuth2 `invalid_grant`.
- An unknown application or wrong secret returns `invalid_client`.
- `client_credentials` is not supported.

Read the user:

```http
GET /v1/oauth2/userinfo
Authorization: Bearer ACCESS_TOKEN
```

Requires `identify`. The `email` scope adds `email` and `verified`. The response includes `sub` and `id` (both the user's snowflake), `username`, `discriminator`, `global_name`, `avatar`, `flags`, `avatar_color`, `bot`, and `system`.

Read the current authorization:

```http
GET /v1/oauth2/@me
Authorization: Bearer ACCESS_TOKEN
```

List or revoke the user's authorized applications:

```text
GET    /v1/oauth2/@me/authorizations
DELETE /v1/oauth2/@me/authorizations/{applicationId}
POST   /v1/oauth2/@me/authorizations/revoke
```

Revoke a token:

```text
POST /v1/oauth2/token/revoke
```

Introspect a token (client credentials required):

```text
POST /v1/oauth2/introspect
```

### 4.6 OAuth2 error shape

An OAuth2 protocol failure uses RFC 6749 fields, not the ordinary API error envelope:

```json
{
  "error": "invalid_grant",
  "error_description": "The authorization code is invalid or expired."
}
```

Common values: `invalid_request`, `invalid_client`, `invalid_grant`, `unauthorized_client`, `unsupported_grant_type`, `invalid_scope`.

### 4.7 Application error quick reference

| Status | Code | Cause |
| --- | --- | --- |
| 400 | `CAPTCHA_REQUIRED` | No CAPTCHA token on a gated operation |
| 400 | `INVALID_CAPTCHA` | Rejected CAPTCHA token |
| 400 | `MAX_APPLICATIONS` | The owner already has 25 applications |
| 400 | `UNCLAIMED_ACCOUNT_CANNOT_CREATE_APPLICATIONS` | The account is not claimed |
| 401 | `INVALID_TOKEN` | The bot token does not resolve to an application |
| 403 | `ACCESS_DENIED` | Wrong credential type or not the owner |
| 403 | `CONTENT_BLOCKED` | The name, redirect URI, or derived bot name is blocked |
| 404 | `UNKNOWN_APPLICATION` | No such application |
| 500 | `BOT_USER_GENERATION_FAILED` | No bot username could be allocated |

---

## 5. HTTP API usage

### 5.1 Base URL and version

```text
GET https://micup.gg/.well-known/fluxer
```

Read `endpoints.api_public`. Then call:

```text
{endpoints.api_public}/v1/...
```

`/v1` is the only HTTP API version. `endpoints.api` and `endpoints.api_client` are aliases for the first-party client; bots and third-party clients use `api_public`.

Do not derive a base URL from the web app origin. A deployment can serve the API from a different host, scheme, port, or path prefix.

### 5.2 Request bodies

- JSON: `Content-Type: application/json`.
- OAuth2 forms: `application/x-www-form-urlencoded` or `multipart/form-data`.
- Message and webhook multipart: include `payload_json` and `files[n]`.

Multipart message fields:

| Field | Meaning |
| --- | --- |
| `payload_json` | JSON object with the complete message request payload; absent means `{}` |
| `files[n]` | A file at zero-based index `n`; `n` is also the attachment `id` in `payload_json.attachments` |
| `content`, `nonce`, `tts`, `flags`, `favorite_meme_id`, `sticker_ids` | Plain form fields on a multipart message; each overrides the same member in `payload_json` |

Rules:

- The index `n` is at most 10,000 and below the deployment's `max_attachments_per_message`.
- `files[n]` indices are attachment IDs, not part order.
- A metadata entry with a `filename` but no matching file fails with `NO_FILE_FOR_ATTACHMENT_METADATA`.
- Two metadata entries claiming one index fail with `DUPLICATE_ATTACHMENT_IDS_NOT_ALLOWED`.
- A supplied file that no metadata entry claims is still attached, using its index as the attachment ID.
- The legacy field names `file` and `fileN` are accepted.

### 5.3 Messages require a visible payload

A message must contain at least one of:

- visible `content`,
- an embed,
- an attachment,
- a favourite meme,
- a sticker.

Otherwise the request fails with `400 CANNOT_SEND_EMPTY_MESSAGE`.

### 5.4 Reusing a nonce

A message `nonce` is echoed on the created message and its `MESSAGE_CREATE` Dispatch. Reusing the same nonce with the same body returns the original result instead of creating a duplicate. Reusing a nonce with a different body is a conflict. Use a nonce for idempotent sends.

### 5.5 Error handling recipe

```text
if response.status == 401:
    # Credential is missing, malformed, expired, or revoked.
    # Do not retry blindly. Verify the token and rotate if needed.
elif response.status == 403:
    # Read body.code:
    # ACCESS_DENIED / MISSING_PERMISSIONS / MISSING_ACCESS /
    # COMMUNICATION_DISABLED / ACCOUNT_LIMITED / ...
elif response.status == 404:
    # Unknown path or resource. Check the route and the channel/guild ID.
elif response.status == 429:
    wait(response.body.retry_after or Retry-After)
    retry once
elif response.status >= 500:
    wait(Retry-After or exponential backoff)
    retry with jitter, at most a few times
else:
    surface body.code and body.message to the operator
```

Never retry `400`, `401`, `403`, or `404` unchanged.

### 5.6 Selected HTTP route map

This is a bot-oriented map, not the complete route list. The full set is in the OpenAPI document and the live documentation.

**Applications and OAuth2**

```text
POST   /v1/oauth2/applications
GET    /v1/users/@me/applications
GET    /v1/oauth2/applications/{id}
PATCH  /v1/oauth2/applications/{id}
DELETE /v1/oauth2/applications/{id}
GET    /v1/oauth2/applications/{id}/public
PATCH  /v1/oauth2/applications/{id}/bot
POST   /v1/oauth2/applications/{id}/bot/reset-token
POST   /v1/oauth2/applications/{id}/client-secret/reset
GET    /v1/applications/@me
GET    /v1/gateway/bot
GET    /v1/oauth2/authorize
POST   /v1/oauth2/authorize/consent
POST   /v1/oauth2/token
POST   /v1/oauth2/token/revoke
POST   /v1/oauth2/introspect
GET    /v1/oauth2/userinfo
GET    /v1/oauth2/@me
```

**Guilds, channels, members, roles**

```text
GET    /v1/users/@me/guilds
GET    /v1/guilds/{guild_id}
PATCH  /v1/guilds/{guild_id}
DELETE /v1/users/@me/guilds/{guild_id}
GET    /v1/guilds/{guild_id}/channels
POST   /v1/guilds/{guild_id}/channels
PATCH  /v1/guilds/{guild_id}/channels
GET    /v1/channels/{channel_id}
PATCH  /v1/channels/{channel_id}
DELETE /v1/channels/{channel_id}
PUT    /v1/channels/{channel_id}/permissions/{overwrite_id}
DELETE /v1/channels/{channel_id}/permissions/{overwrite_id}
GET    /v1/guilds/{guild_id}/members
GET    /v1/guilds/{guild_id}/members/{user_id}
PATCH  /v1/guilds/{guild_id}/members/{user_id}
DELETE /v1/guilds/{guild_id}/members/{user_id}
PUT    /v1/guilds/{guild_id}/members/{user_id}/roles/{role_id}
DELETE /v1/guilds/{guild_id}/members/{user_id}/roles/{role_id}
GET    /v1/guilds/{guild_id}/roles
POST   /v1/guilds/{guild_id}/roles
PATCH  /v1/guilds/{guild_id}/roles/{role_id}
DELETE /v1/guilds/{guild_id}/roles/{role_id}
GET    /v1/guilds/{guild_id}/bans
PUT    /v1/guilds/{guild_id}/bans/{user_id}
DELETE /v1/guilds/{guild_id}/bans/{user_id}
GET    /v1/guilds/{guild_id}/invites
POST   /v1/channels/{channel_id}/invites
DELETE /v1/invites/{invite_code}
GET    /v1/guilds/{guild_id}/audit-logs
```

**Messages**

```text
GET    /v1/channels/{channel_id}/messages
POST   /v1/channels/{channel_id}/messages
GET    /v1/channels/{channel_id}/messages/{message_id}
PATCH  /v1/channels/{channel_id}/messages/{message_id}
DELETE /v1/channels/{channel_id}/messages/{message_id}
POST   /v1/channels/{channel_id}/messages/bulk-delete
POST   /v1/channels/{channel_id}/messages/{message_id}/crosspost
POST   /v1/channels/{channel_id}/typing
GET    /v1/channels/{channel_id}/messages/pins
PUT    /v1/channels/{channel_id}/pins/{message_id}
DELETE /v1/channels/{channel_id}/pins/{message_id}
PUT    /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions
POST   /v1/channels/{channel_id}/attachments
POST   /v1/channels/{channel_id}/attachments/complete
POST   /v1/attachments/refresh-urls
```

**Application commands and interactions**

```text
GET    /v1/applications/{application_id}/commands
PUT    /v1/applications/{application_id}/commands
POST   /v1/applications/{application_id}/commands
DELETE /v1/applications/{application_id}/commands
GET    /v1/applications/{application_id}/commands/{command_id}
PATCH  /v1/applications/{application_id}/commands/{command_id}
DELETE /v1/applications/{application_id}/commands/{command_id}
GET    /v1/applications/{application_id}/guilds/{guild_id}/commands
PUT    /v1/applications/{application_id}/guilds/{guild_id}/commands
POST   /v1/applications/{application_id}/guilds/{guild_id}/commands
DELETE /v1/applications/{application_id}/guilds/{guild_id}/commands
POST   /v1/interactions/{interaction_id}/{interaction_token}/callback
POST   /v1/interactions/{interaction_id}/followups
GET    /v1/interactions/{interaction_id}
GET    /v1/interactions/{interaction_id}/response
POST   /v1/interactions
```

`POST /v1/interactions` invokes a bot command as a **user**; it is not how a bot receives interactions. Bots receive interactions on the Gateway.

**Webhooks**

```text
GET    /v1/guilds/{guild_id}/webhooks
GET    /v1/channels/{channel_id}/webhooks
POST   /v1/channels/{channel_id}/webhooks
GET    /v1/webhooks/{webhook_id}
PATCH  /v1/webhooks/{webhook_id}
DELETE /v1/webhooks/{webhook_id}
GET    /v1/webhooks/{webhook_id}/{token}
PATCH  /v1/webhooks/{webhook_id}/{token}
DELETE /v1/webhooks/{webhook_id}/{token}
POST   /v1/webhooks/{webhook_id}/{token}
GET    /v1/webhooks/{webhook_id}/{token}/messages/{message_id}
PATCH  /v1/webhooks/{webhook_id}/{token}/messages/{message_id}
DELETE /v1/webhooks/{webhook_id}/{token}/messages/{message_id}
POST   /v1/webhooks/{webhook_id}/{token}/github
POST   /v1/webhooks/{webhook_id}/{token}/slack
POST   /v1/webhooks/{webhook_id}/{token}/instatus
```

### 5.7 Pagination

Collection endpoints that page over snowflake-ordered resources accept `before`, `after`, or `around`. The operation defines which cursors it supports, whether they are mutually exclusive, the result order, and how `limit` applies. Use the returned IDs as the next cursor; do not compute cursors from timestamps unless the endpoint documents that use.

### 5.8 Admin API

The Admin API lives below `/v1/admin` and uses `Admin <key>`. It is for instance operators only. Bot tokens are rejected. Its ACLs, audit requirements, and routes are outside this reference; see <https://micup.gg/docs/admin-api/>.
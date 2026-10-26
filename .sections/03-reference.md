---

## 9. Gateway

The Gateway is one persistent WebSocket. It sends account and guild events and accepts a small set of bounded client commands. Every other read or mutation is an HTTP API operation.

### 9.1 Connecting

1. Read `endpoints.gateway` from `GET /.well-known/fluxer`.
2. Open a WebSocket with:

```text
wss://micup.gg/gateway?v=1&encoding=json
```

Optional compression:

```text
wss://micup.gg/gateway?v=1&encoding=json&compress=zstd-stream&stream=1
```

3. Read Opcode `10` `Hello` and take `heartbeat_interval`.
4. Send Opcode `2` `Identify` with the raw token and `properties`.
5. Read `READY`, keep `session_id`, and use its sequence for heartbeats.
6. Send Opcode `1` `Heartbeat` with the last processed Dispatch sequence every `heartbeat_interval` milliseconds.

Connection parameters:

| Parameter | Meaning |
| --- | --- |
| `v` | Gateway protocol version; must be `1` |
| `encoding` | Only `json` is implemented; absent selects JSON |
| `compress` | `zstd-stream` or `none` |
| `stream` | `1` or `true` enables the compression stream |

A missing or non-`1` `v` closes with `4012` **before Hello**. Unknown parameters are ignored. An unrecognized `compress` or `stream` value selects no compression and leaves the socket open.

With `compress=zstd-stream&stream=1`, every server frame is binary and every client frame must use the same compressor. Hello is already compressed, so the first inbound frame is binary. Compression is negotiated once per connection; changing it requires a new socket and a new Identify or Resume.

### 9.2 Payload envelope

```json
{
  "op": 0,
  "t": "MESSAGE_CREATE",
  "s": 42,
  "d": {}
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `op` | integer | Opcode |
| `d` | any | Payload; required on client-to-server payloads; opcode-dependent on server-to-client payloads |
| `s` | integer | Dispatch sequence; only on Dispatch |
| `t` | string | Event name; only on Dispatch |

A session starts at sequence `0`, so `READY` is sequence `1`. Every live Dispatch advances the sequence. A replayed Dispatch keeps its original sequence and a replay can contain gaps. `RESUMED` carries the current sequence and does not advance it.

### 9.3 Opcodes

| Op | Name | Direction | Purpose |
| ---: | --- | --- | --- |
| `0` | Dispatch | Server → client | Named event |
| `1` | Heartbeat | Bidirectional | Keepalive; server may request an immediate beat with `d: null` |
| `2` | Identify | Client → server | Start an authenticated session |
| `3` | Presence Update | Client → server | Replace session presence |
| `4` | Voice State Update | Client → server | Join, move, update, or leave voice |
| `5` | Voice Server Ping | Reserved | Never sent; sending it closes the socket |
| `6` | Resume | Client → server | Restore a retained session |
| `7` | Reconnect | Server → client | Open a new socket; the current one closes with `4000` |
| `8` | Request Guild Members | Client → server | Request bounded member chunks |
| `9` | Invalid Session | Server → client | `d: false`; the session cannot be resumed |
| `10` | Hello | Server → client | `heartbeat_interval` |
| `11` | Heartbeat ACK | Server → client | Acknowledges a heartbeat |
| `12` | Gateway Error | Reserved | Never sent; errors arrive as Dispatches or close frames |
| `14` | Lazy Request | Client → server | Replace bounded guild subscriptions |
| `15` | Request Guild Counts | Client → server | Request guild count records |
| `16` | Request Channel Member Counts | Client → server | Request channel count records |

Opcode `13` and every value above `16` are undefined. An undefined opcode, a server-only opcode sent by a client, or a payload with no `d` closes with `4001` once a session is attached, and with `4003` before authentication. `Identify` with no `d` closes with `4005`.

### 9.4 Identify

```json
{
  "op": 2,
  "d": {
    "token": "1557753609842589699.D3f...",
    "properties": {
      "os": "Linux",
      "browser": "my-bot",
      "device": "server"
    },
    "presence": {
      "status": "online",
      "afk": false,
      "mobile": false
    },
    "ignored_events": ["TYPING_START"],
    "flags": 2,
    "shard": [0, 1]
  }
}
```

| Field | Required? | Meaning |
| --- | --- | --- |
| `token` | Yes | Raw bot or user token, no `Bot ` prefix |
| `properties` | Yes | Object with `os`, `browser`, and `device` strings |
| `presence` | No | Initial presence; null or omitted uses saved status |
| `ignored_events` | No | Up to 256 Dispatch event names to suppress |
| `flags` | No | Non-negative session flags bitfield |
| `initial_guild_id` | No | One guild to join as active without a Lazy Request |
| `shard` | No | `[shard_id, shard_count]` |

There is **no `intents` field and no intent system**. `Identify` ignores unknown fields. The following close with `4002` and reason `Invalid identify payload`:

- missing `token` or `properties`;
- `properties` not an object;
- missing or non-string `os`, `browser`, or `device`;
- `ignored_events` not an array of strings, or more than 256 entries;
- `flags` not a non-negative integer.

A malformed `shard` closes with `4010` and reason `Invalid shard`. A bot session that resolves to more than 2,500 guilds after the shard filter closes with `4011` and reason `Sharding required`. A non-bot account with 100 live sessions closes with `4008` and reason `Too many sessions`. A bot credential is not bounded by that count.

Session flags:

| Value | Name | Effect |
| ---: | --- | --- |
| `1 << 1` (2) | `DEBOUNCE_MESSAGE_REACTIONS` | Merge runs of reaction additions in a DM/group DM into `MESSAGE_REACTION_ADD_MANY` |

Undefined session-flag bits are accepted and ignored.

`properties` optional hints:

- `e2ee_capable`: whether the client can join end-to-end encrypted voice. A session without it is refused from such a channel with `VOICE_E2EE_REQUIRED`.
- `mobile`: used when `presence` is absent.
- `latitude`, `longitude`: decimal strings; used to order `rtc_regions` by distance. Send both or neither. A number fails validation; send `"52.52"`, not `52.52`.

### 9.5 Heartbeats

Hello advertises `heartbeat_interval`, normally **41,250 ms**.

```json
{"op": 1, "d": 42}
```

- Before a session exists, the payload is ignored but still acknowledged.
- After a session exists, `d` is the last processed Dispatch sequence, or `null` before any Dispatch.
- A `d` that is neither `null` nor an integer closes with `4007` and reason `Invalid sequence`.
- The server answers with Opcode `11` `Heartbeat ACK`.
- The server requests an immediate heartbeat with Opcode `1` and `d: null` once 90% of the interval has elapsed since the last acknowledgement. Answer it immediately.
- If no new acknowledgement is accepted within **45,000 ms**, the socket closes with `4009` and reason `Heartbeat timeout`.

A heartbeat with a sequence trims every retained Dispatch at or below that sequence and records it as acknowledged. A client MUST send the sequence it has actually processed. A later Resume from a lower sequence fails with `4007`.

### 9.6 Resume

```json
{
  "op": 6,
  "d": {
    "token": "1557753609842589699.D3f...",
    "session_id": "6f1d0b7c9a2e4f83b5c1d9e7a4f20b13",
    "seq": 42
  }
}
```

- All fields are required.
- A missing field, a non-string `token` or `session_id`, or a non-integer `seq` closes with `4002` and reason `Invalid resume payload`.
- A successful Resume replays every retained Dispatch above `seq`, in order, then sends `RESUMED`.
- A disconnected session is retained for **60,000 ms**.
- An accepted `seq` is at most the current sequence and at least the acknowledged sequence. Outside those bounds closes with `4007`.
- An unknown or expired session, or a `seq` below the replay floor, sends Opcode `9` with `d: false` and leaves the socket unauthenticated.
- A token that does not own the session closes with `4004` and reason `Invalid token`.
- Resume is accepted in every open state. A socket that already holds a session still processes Resume and attaches the named session in its place.
- A socket displaced by a Resume receives Opcode `7` and closes with `4000`.

There is no separate resume URL. Reconnect to the discovered Gateway endpoint.

### 9.7 Sharding

`shard` is `[shard_id, shard_count]`:

- `shard_count` is 1–16,384.
- `shard_id` is 0 through `shard_count - 1`.
- An omitted or null value applies no sharding.
- Guild shard: `((guild_id >> 22) % shard_count)`, computed on the integer value of the guild snowflake.
- At Identify, the Gateway filters the account's guild list to the shard's guilds.
- Bot shard with more than 2,500 guilds closes with `4011` and reason `Sharding required`.
- A user session is never refused for guild count.
- Shard `0` also receives account-level traffic. A non-zero shard drops every Dispatch that names no guild, except `RATE_LIMITED`, `GUILD_COUNTS_UPDATE`, and `CHANNEL_MEMBER_COUNTS_UPDATE`.

`GET /v1/gateway/bot` reports `shards: 1` as a recommendation, but Identify accepts any valid shard count. Micup has no large-bot tier, no shard alignment requirement, and no Identify concurrency buckets.

### 9.8 Close codes

| Code | Name | Meaning |
| ---: | --- | --- |
| `4000` | Unknown error | Drain, unclassified session error, unreachable retained session, or `RESUMED` failure |
| `4001` | Unknown opcode | Undefined opcode, server opcode from client, or missing `d` |
| `4002` | Decode error | Bad JSON, bad payload, bad compression, bad command fields, oversized payload |
| `4003` | Not authenticated | Authenticated command before a session, or Identify timeout |
| `4004` | Authentication failed | Invalid token, or token does not own the named session |
| `4005` | Already authenticated | Identify while a session is attached |
| `4007` | Invalid sequence | Heartbeat or Resume sequence outside accepted bounds |
| `4008` | Rate limited | Connection, payload, Identify, or session budget exceeded |
| `4009` | Session timeout | No heartbeat acknowledgement within 45,000 ms |
| `4010` | Invalid shard | Malformed `[shard_id, shard_count]` |
| `4011` | Sharding required | More than 2,500 guilds on one bot session |
| `4012` | Invalid API version | `v` absent or not `1` |

`4006` is unassigned; no code above `4012` is defined. A reconnect with the same token, shard pair, and version reproduces the same failure.

### 9.9 Dispatch events

Event names are literal strings in the `t` field. The live registry:

**Session and current user**

`READY`, `RESUMED`, `SESSIONS_REPLACE`, `AUTH_SESSION_CHANGE`, `RATE_LIMITED`, `USER_UPDATE`, `USER_SETTINGS_UPDATE`, `USER_GUILD_SETTINGS_UPDATE`, `USER_NOTE_UPDATE`, `USER_PINNED_DMS_UPDATE`, `USER_CONNECTIONS_UPDATE`, `WEBAUTHN_CREDENTIALS_UPDATE`, `RELATIONSHIP_ADD`, `RELATIONSHIP_UPDATE`, `RELATIONSHIP_REMOVE`, `SAVED_MESSAGE_CREATE`, `SAVED_MESSAGE_DELETE`, `RECENT_MENTION_DELETE`, `FAVORITE_MEME_CREATE`, `FAVORITE_MEME_UPDATE`, `FAVORITE_MEME_DELETE`, `FEED_NOTIFICATION_ACK`, `FEED_NOTIFICATION_CREATE`, `FOLLOW_CREATE`, `FOLLOW_DELETE`.

**Guilds and channels**

`GUILD_CREATE`, `GUILD_SYNC`, `GUILD_UPDATE`, `GUILD_DELETE`, `GUILD_ROLE_CREATE`, `GUILD_ROLE_UPDATE`, `GUILD_ROLE_UPDATE_BULK`, `GUILD_ROLE_DELETE`, `GUILD_EMOJIS_UPDATE`, `GUILD_STICKERS_UPDATE`, `GUILD_SOUNDBOARD_SOUNDS_UPDATE`, `GUILD_SOUNDBOARD_SOUND_PLAY`, `CHANNEL_CREATE`, `CHANNEL_UPDATE`, `CHANNEL_UPDATE_BULK`, `CHANNEL_DELETE`, `CHANNEL_RECIPIENT_ADD`, `CHANNEL_RECIPIENT_REMOVE`, `WEBHOOKS_UPDATE`, `INVITE_CREATE`, `INVITE_DELETE`.

**Members and moderation**

`GUILD_MEMBER_ADD`, `GUILD_MEMBER_UPDATE`, `GUILD_MEMBER_REMOVE`, `GUILD_MEMBERS_CHUNK`, `GUILD_MEMBER_LIST_UPDATE`, `GUILD_AUDIT_LOG_ENTRY_CREATE`, `GUILD_BAN_ADD`, `GUILD_BAN_REMOVE`, `PRESENCE_UPDATE`, `PRESENCE_UPDATE_BULK`, `PASSIVE_UPDATES`.

**Messages and reactions**

`MESSAGE_CREATE`, `MESSAGE_UPDATE`, `MESSAGE_DELETE`, `MESSAGE_DELETE_BULK`, `MESSAGE_ACK`, `MESSAGE_REACTION_ADD`, `MESSAGE_REACTION_ADD_MANY`, `MESSAGE_REACTION_REMOVE`, `MESSAGE_REACTION_REMOVE_ALL`, `MESSAGE_REACTION_REMOVE_EMOJI`, `TYPING_START`, `CHANNEL_PINS_UPDATE`, `CHANNEL_PINS_ACK`.

**Voice and calls**

`VOICE_STATE_UPDATE`, `VOICE_SERVER_UPDATE`, `ENTRANCE_SOUND_PLAY`, `TABLE_GAME_UPDATE`, `CALL_CREATE`, `CALL_UPDATE`, `CALL_DELETE`.

**Count responses**

`GUILD_COUNTS_UPDATE`, `CHANNEL_MEMBER_COUNTS_UPDATE`.

**Applications and interactions**

`APPLICATION_COMMANDS_UPDATE`, `INTERACTION_CREATE`, `INTERACTION_RESPONSE_CREATE`, `INTERACTION_RESPONSE_UPDATE`, `INTERACTION_RESPONSE_DELETE`.

`READY` key fields:

| Field | Meaning |
| --- | --- |
| `session_id` | Resume identifier |
| `version` | Gateway version, always `1` |
| `user` | Authenticated account in its private representation |
| `guilds` | Guilds for the session; bot guilds are unavailable stubs |
| `private_channels` | DM and group DM channels |
| `relationships` | The account's relationships |
| `presences` | Visible presences at connect time |
| `users` | Referenced users |
| `sessions` | The account's other live sessions |
| `read_states` | Per-channel read state |
| `user_settings` | Account-wide settings |
| `user_guild_settings` | Per-guild notification settings |
| `notes`, `pinned_dms`, `favorite_memes` | Account data |
| `shard` | The accepted shard pair, when one was sent |
| `application` | The bot application, for a bot session |
| `interest_onboarding_required?` | `true` when the account must finish the interest picker |

For a bot session, each guild in `READY.guilds` is an unavailable stub, followed by one `GUILD_CREATE` per guild. `GUILD_CREATE` and `GUILD_SYNC` carry the complete roles, channels, emojis, stickers, and voice states for that guild; replace stored lists with them. Every other guild Dispatch changes part of that state.

### 9.10 Event filtering

Micup has **no intents**. Replace an intent mask with these mechanisms:

- `Identify.ignored_events`: suppress up to 256 named Dispatches.
- `Lazy Request` (op `14`): subscribe to guild member lists and mark guilds active.
- Guild availability and permissions: the server only sends events the session may see.
- Sharding: a non-zero shard receives only its guilds' guild-scoped events.

Gates, in order:

1. **Guild availability.** An unavailable guild sends nothing except `GUILD_UPDATE`.
2. **Permission and visibility.** Channel-scoped and message-access-filtered events go only to sessions that can see the channel or message.
3. **Guild subscription state.** A passive user session in a guild with more than 250 members receives a reduced set. Bots are never passive.
4. **Session filters.** The shard filter, then `ignored_events`.

A passive user session in a large guild receives:

- `GUILD_UPDATE`, `GUILD_DELETE`
- `GUILD_ROLE_UPDATE`, `GUILD_ROLE_UPDATE_BULK`
- `CHANNEL_CREATE`, `CHANNEL_UPDATE`, `CHANNEL_UPDATE_BULK`, `CHANNEL_DELETE`
- `GUILD_AUDIT_LOG_ENTRY_CREATE`
- `PASSIVE_UPDATES`
- `MESSAGE_CREATE` only when the message mentions the session's user
- `GUILD_MEMBER_UPDATE`, `GUILD_MEMBER_REMOVE` only for the session's own user

`VOICE_STATE_UPDATE` follows its own path: sessions that can view the channel. `TYPING_START` is decided by the guild's typing subscription.

`Lazy Request` (op `14`) replaces bounded guild subscriptions. It accepts member-list ranges and explicit member IDs; ranges are at most 100,000 with a span of at most 99, and explicit IDs are at most 1,000 per guild. Up to 10 ranges per channel.

### 9.11 Gateway limits

| Limit | Value |
| --- | --- |
| Inbound WebSocket message | 4,096 bytes on the wire, and 4,096 bytes after decompression |
| Compressed message decompressed size | 10 MiB; beyond it closes `4002` |
| Heartbeat interval | 41,250 ms advertised |
| Heartbeat timeout | 45,000 ms since last acknowledgement |
| Session retention after socket drop | 60,000 ms |
| Presence publish after drop | `offline` after 5,000 ms |
| Client payloads per socket | 600 in a rolling 60 s |
| Client payloads per authenticated session | 600 in a fixed 60 s bucket |
| Client payloads per source IP | 6,000 in a fixed 60 s bucket |
| Concurrent WebSockets per source IP | 256 |
| Identify attempts per source IP | 300 per fixed 60 s |
| Sessions per user | 100 |
| Bot guilds per shard | 2,500 |
| Replay buffer | 4,096 Dispatches or 16 MiB, whichever fills first |
| Single replayed event ceiling | 2 MiB; larger events are delivered but not retained |
| Presence updates | 5 per rolling 20 s per socket |
| Voice state updates | 2 per rolling 1 s immediate; later ones queue up to 64 and drain one per 500 ms |
| Bounded requests in flight | 4 |
| Bounded request deadline | 10,000 ms |
| Request Guild Members complete list | One accepted request per guild per 30 s |

Behavior on excess:

- Socket/session/IP payload budget: close `4008` and reason `Rate limited`.
- More than 256 concurrent sockets from one IP: close `4008` and reason `Too many connections` before Hello.
- Identify budget exhausted: the payload is discarded with **no reply and no close**. The client must send a new Identify after the budget recovers, before the heartbeat deadline.
- Presence over budget: discarded without closing.
- Bounded request with all four slots occupied: dropped with no result and no close.
- `GUILD_SYNC`, `GUILD_MEMBER_LIST_UPDATE`, and `GUILD_MEMBERS_CHUNK` are never retained for Resume.

### 9.12 Minimal raw Gateway handshake

```js
const ws = new WebSocket('wss://micup.gg/gateway?v=1&encoding=json');

ws.addEventListener('message', (event) => {
  const payload = JSON.parse(event.data);
  if (payload.op === 10) {
    ws.send(JSON.stringify({
      op: 2,
      d: {
        token: process.env.MICUP_TOKEN,
        properties: {os: 'linux', browser: 'my-bot', device: 'server'},
      },
    }));
    setInterval(() => ws.send(JSON.stringify({op: 1, d: null})), payload.d.heartbeat_interval);
  }
  if (payload.op === 0 && payload.t === 'READY') {
    console.log('session', payload.d.session_id);
  }
  if (payload.op === 0 && payload.t === 'INTERACTION_CREATE') {
    // Answer within 3 seconds with the interaction token.
  }
});
```

This is illustrative. A production client must track `s`, send the last processed sequence, handle `RECONNECT` and `INVALID_SESSION`, and resume within 60 seconds.

---

## 10. Webhooks

### 10.1 Webhook types

| Value | Type | Meaning |
| ---: | --- | --- |
| `1` | `INCOMING` | Posts the messages its token holder sends |
| `2` | `CHANNEL_FOLLOWER` | Posts copies of messages published in a followed announcement channel |

An incoming webhook belongs to one guild and posts into one guild text, voice, or announcement channel. Its execution token is generated once at creation and **never rotates**. The ID and token pair is a bearer credential for the webhook's lifetime.

A channel follower webhook has no usable token. Every token route returns `404 UNKNOWN_WEBHOOK` for it, so nothing can post through it except Micup itself.

### 10.2 Authorisation

- Execution and token management use the token in the path, not an `Authorization` header.
- Management routes (`GET/PATCH/DELETE /v1/webhooks/{webhook_id}`) use a bot token and `MANAGE_WEBHOOKS`.
- Create a webhook with `POST /v1/channels/{channel_id}/webhooks` and `MANAGE_WEBHOOKS`.
- A webhook can target a guild text, voice, or announcement channel.
- The webhook token is a secret. Anyone who has the full URL can post, edit, and delete the webhook's messages.
- A first-party web client `Origin` is refused with `403 INVALID_API_ORIGIN`. Execute webhooks from a server, not from a browser.

### 10.3 Execute a webhook

```http
POST /v1/webhooks/{webhook_id}/{token}?wait=true
Content-Type: application/json

{
  "content": "Build #128 passed",
  "username": "CI",
  "allowed_mentions": {"parse": []}
}
```

- `wait=true` returns the created message. The exact trimmed values `true`, `True`, and `1` are true; everything else is false.
- Without `wait`, the response is `204`.
- JSON and multipart bodies are accepted.
- A JSON body cannot attach a file. Use `files[n]` in a multipart body.
- `username` overrides the author name for that message.
- `avatar_url` overrides the avatar for that message; if it cannot be fetched, the message is created without the override.
- The stored webhook name and avatar do not change.
- A webhook message can have `content`, embeds, attachments, stickers, a favourite meme, or a reference, but not none.
- An **omitted** `allowed_mentions` in a webhook body suppresses every mention, which is the opposite of `Create message`.
- `nonce` is remembered per webhook for five minutes. A repeat within that window creates no second message and returns the first.
- Rate limit: 60 requests per minute per caller and webhook, exempt from the global bucket.

### 10.4 Webhook message routes

```text
GET    /v1/webhooks/{webhook_id}/{token}/messages/{message_id}
PATCH  /v1/webhooks/{webhook_id}/{token}/messages/{message_id}
DELETE /v1/webhooks/{webhook_id}/{token}/messages/{message_id}
```

The token can edit or delete messages the webhook authored. It cannot edit or delete another author's message.

### 10.5 Slack, GitHub, and Instatus

```text
POST /v1/webhooks/{webhook_id}/{token}/slack
POST /v1/webhooks/{webhook_id}/{token}/github
POST /v1/webhooks/{webhook_id}/{token}/instatus
```

These routes accept the payload shapes of the named service and render them as Micup messages. They are not general-purpose message endpoints; use the base execute route for custom payloads.

### 10.6 Webhook errors

| Status | Code | Cause |
| --- | --- | --- |
| 400 | `CANNOT_SEND_EMPTY_MESSAGE` | No visible content, embed, or attachment |
| 400 | `INVALID_FORM_BODY` | Validation failed; inspect `errors` |
| 401 | `MISSING_AUTHORIZATION` | Management route without a credential |
| 403 | `MISSING_PERMISSIONS` | Missing `MANAGE_WEBHOOKS` or channel permission |
| 403 | `INVALID_API_ORIGIN` | First-party web client origin |
| 403 | `CONTENT_BLOCKED` | Blocked content |
| 404 | `UNKNOWN_WEBHOOK` | Webhook/token pair not found |
| 404 | `UNKNOWN_CHANNEL` | Target channel missing or wrong type |
| 404 | `UNKNOWN_MESSAGE` | Webhook message not found |
| 429 | `RATE_LIMITED` | Webhook execution allowance exceeded |

---

## 11. Voice

### 11.1 Transport

Micup runs voice over **LiveKit**. The main Gateway places a session into a voice channel and returns one media credential. The client presents that credential to the LiveKit endpoint and sends every microphone, camera, and screen-share track over that connection.

There is **no voice WebSocket**, **no voice opcode set**, **no UDP discovery step**, and **no separate encryption handshake** published by Micup. Do not invent one.

Room names:

| Surface | Room name |
| --- | --- |
| Guild voice channel | `guild_{guild_id}_channel_{channel_id}` |
| Private call | `dm_channel_{channel_id}` |
| Participant identity | `user_{user_id}_{connection_id}` |

The grant lives for **600 seconds**.

### 11.2 Placement

Send Opcode `4` `Voice State Update`:

```json
{
  "op": 4,
  "d": {
    "guild_id": "1501314428688990000",
    "channel_id": "1501314428688991111",
    "connection_id": "1730000000000000000",
    "self_mute": false,
    "self_deaf": false,
    "self_video": false,
    "self_stream": false
  }
}
```

| Shape | Result |
| --- | --- |
| `channel_id` and no `connection_id` | Open a new connection |
| `channel_id` and a `connection_id` | Update or move that connection |
| `channel_id: null` and a `connection_id` in a guild | Drop that connection |
| `channel_id: null` and no `connection_id` in a guild | Refused with `VOICE_MISSING_CONNECTION_ID` |
| `channel_id: null` and no `connection_id` in a call | Drop every voice membership in the session |

The server answers with `VOICE_STATE_UPDATE` for the resulting membership to every session that can view the channel, and `VOICE_SERVER_UPDATE` to the requesting session with:

- `token`
- `endpoint`
- `channel_id`
- `connection_id`
- `guild_id` (present for a guild channel, omitted for a call)
- `e2ee_key` when the channel is end-to-end encrypted

A refused placement produces **no Dispatch and no error event**. The client observes only the absence of a grant.

### 11.3 Voice permissions and capacity

| Permission | Effect |
| --- | --- |
| `VIEW_CHANNEL` + `CONNECT` | Hold a voice connection |
| `SPEAK` | Publish audio |
| `STREAM` | Publish camera and screen share |
| `MUTE_MEMBERS` | Apply/clear moderator mute |
| `DEAFEN_MEMBERS` | Apply/clear moderator deafen |
| `MOVE_MEMBERS` | Move or disconnect a member |
| `UPDATE_RTC_REGION` | Change the channel region |

Capacity:

| Bound | Refusal |
| --- | --- |
| `user_limit` (`0` = unlimited) | `VOICE_CHANNEL_FULL` |
| `voice_connection_limit` (default 5, max 100) | `VOICE_CONNECTION_LIMIT_REACHED` |
| 25 members with camera on | `VOICE_CAMERA_USER_LIMIT` |

A timed-out member is refused with `VOICE_MEMBER_TIMED_OUT`. An unclaimed account is refused for a one-to-one DM call and for a guild voice channel it does not own, except a group DM call. A session without `e2ee_capable` is refused from an E2EE channel when every existing connection is capable; a bot is exempt from that check.

### 11.4 Private calls, Go Live, entrance sounds

- A private call is the DM/group DM counterpart of a voice channel. Its room name is `dm_channel_{channel_id}`.
- Go Live publishes a screen-share track on the voice connection the member already holds. It opens no second connection and issues no second credential.
- An entrance sound is a short clip announced to everyone already in the channel. It publishes no media track.
- Voice activity sharing is a user setting that tells friends which voice channel the account is in.

### 11.5 3D rooms, voice seats, and table games

A guild voice channel can have a `room_theme`:

| Theme | Layout |
| --- | --- |
| `gece-lounge` | Night lounge, one table `T01`, four watch seats |
| `kahvehane` | Coffee house, tables `T01`–`T03`, two watch seats each |
| `oyun-odasi` | Game room, tables `T01`–`T02`, four watch seats each |

The server owns seat assignment. A connection is seated on join, moves or stands when it leaves, and carries its seat as a LiveKit participant attribute.

Voice-seat routes are **user-only**; a bot or OAuth2 bearer credential is refused with `403 ACCESS_DENIED`. 3D rooms and table games are feature-gated and can be off.

Table games:

| Kind | Description |
| --- | --- |
| `okey` | Classic Okey, four players, one hand |
| `okey101` | 101 Okey, four players, one hand, scored |
| `chess` | Chess, two players at positions 1 and 3 |
| `dama` | Türk daması, two players |
| `dama_normal` | Normal dama (Brazilian draughts), two players |
| `tavla` | Backgammon, no doubling cube, two players |

The server deals, rolls, validates moves, runs clocks, and sends each player only what that player may see. Spectators receive public views with `rack: null`.

Table game routes:

```text
GET    /v1/channels/{channel_id}/table-games
GET    /v1/channels/{channel_id}/table-games/{slot}
POST   /v1/channels/{channel_id}/table-games/{slot}
PUT    /v1/channels/{channel_id}/table-games/{slot}/players/@me
DELETE /v1/channels/{channel_id}/table-games/{slot}/players/@me
POST   /v1/channels/{channel_id}/table-games/{slot}/actions
PUT    /v1/channels/{channel_id}/table-games/{slot}/spectators/@me
DELETE /v1/channels/{channel_id}/table-games/{slot}/spectators/@me
GET    /v1/table-games/rewards
GET    /v1/table-games/leaderboards/{game}
GET    /v1/guilds/{guild_id}/table-games/leaderboard
GET    /v1/users/@me/table-game-ratings
```

They are user-only. A running ranked game has additional protection: changing the room theme or deleting the channel can return `409 TABLE_GAME_RANKED_IN_PROGRESS`.

### 11.6 Bot voice

A bot is exempt from the E2EE capability check, so a bot session can be placed in voice under the normal permission and capacity rules. Micup does **not** document a bot voice SDK, a server-side audio mixer, or a REST route that publishes audio. A bot that needs voice must implement the LiveKit client side itself using the grant from `VOICE_SERVER_UPDATE`, and must handle Opcode `4` and the `VOICE_STATE_UPDATE` / `VOICE_SERVER_UPDATE` events.

Do not claim that a bot can "join voice" with a single REST call. Placement is a Gateway operation, and media is a LiveKit connection.

---

## 12. Media and uploads

### 12.1 Media Proxy

Take `endpoints.media` from discovery. Live value: `https://media.micup.gg`. Read paths have **no `/v1` prefix**.

| Family | Paths |
| --- | --- |
| Attachment | `/attachments/{path}` |
| Signed external media | `/external/{signature}/{target}` |
| Theme CSS | `/themes/{path}.css` |
| Entrance sound | `/entrance-sounds/{user_id}/{filename}` |
| Image assets | `/avatars`, `/icons`, `/branding`, `/banners`, `/splashes`, `/embed-splashes`, `/emojis`, `/stickers` |
| Guild member images | `/guilds/{guild_id}/users/{user_id}/avatars`, `/banners` |
| Static object | `/{key}` on a `static` endpoint |
| Upload relay | `/v1/relay/{key}` on the relay base |

Public reads do **not** read the `Authorization` header. A stored object is authorised by its path, external media by its signature, and an upload by the capability in its URL.

A signed media URL is a bearer capability. Anyone holding it can read the object until it expires. Do not expose it through redirects, traces, or referrers.

### 12.2 Signed attachment URLs

- Attachment URLs returned in messages, embeds, and feed posts are signed.
- They expire. The exact lifetime is at most one day from signing for an ordinary signed attachment URL.
- Re-read the message, post, or attachment to obtain a fresh URL.
- `POST /v1/attachments/refresh-urls` refreshes a batch of attachment URLs.
- The `url` and `proxy_url` of a returned attachment are the same signed URL.
- A data-package URL works until the signing secret is removed.

### 12.3 Upload flows

**Inline multipart** is the simplest path:

```http
POST /v1/channels/{channel_id}/messages
Authorization: Bot <token>
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="payload_json"
Content-Type: application/json

{"content":"Screenshot","attachments":[{"id":0,"filename":"shot.png"}]}
--boundary
Content-Disposition: form-data; name="files[0]"; filename="shot.png"
Content-Type: image/png

<bytes>
--boundary--
```

**Pre-upload**:

1. `POST /v1/channels/{channel_id}/attachments` with up to 10 file plans. Each plan declares an ID, filename, exact byte count, and content type.
2. If `upload_mode` is `singlepart` (at most 10,485,760 bytes), `PUT` the bytes to `upload_url`.
3. If `upload_mode` is `multipart`, upload each part to its own URL using the returned `part_size` and part numbers, then `POST /v1/channels/{channel_id}/attachments/complete` with each `upload_filename` and `upload_id`.
4. Send the `upload_filename` in the message's `attachments` array.

Rules:

- Do not send `Authorization` on the upload URL. The URL's capability is the credential.
- Do not rewrite, decode, reorder, or re-encode the URL.
- Singlepart URL lifetime: 5 minutes. Multipart part URL: 1 hour. Relay URL: 900 seconds by default.
- There is no resume. If a capability expires, request a new plan and restart.
- An upload is bound to the identity and channel that planned it, and its key is single-use.
- `presigned_attachment_uploads` must be true. If false, use inline multipart.
- Stream previews use the user-only Streams API, not the attachment flow.

### 12.4 Media errors

The Media Proxy returns a short `text/plain` body, not the API error envelope. Common statuses:

| Status | Meaning |
| --- | --- |
| 200 | Object returned |
| 206 | Partial content for a valid `Range` |
| 401 | Missing/invalid/expired upload capability |
| 403 | Signature or origin refused |
| 404 | Object, route, or mode not found |
| 413 | Uploaded body exceeds the limit |
| 416 | Unsatisfiable range |
| 500 | Upstream storage failure |

`HEAD` is supported on read routes and returns the same status and representation headers with no body.

---

## 13. Micup-specific platform features

These are features of the Micup instance rather than generic Fluxer protocol surfaces. Almost all are **user-only**: they reject bot and OAuth2 bearer credentials with `403 ACCESS_DENIED`. Do not design a third-party bot around calling them.

### 13.1 Official community bots

Micup ships two official bot applications:

| Bot | Purpose | Invite permission mask |
| --- | --- | --- |
| **MicupBOT** | Welcome/goodbye, autorole, reaction roles, message/voice XP, levels, moderation commands, automod | `1099780156480` |
| **Micup Roster** | Hourly event cards, Join/Leave buttons, voice checks, main/sub rosters | `17910848` |

Discover them through:

- `features.official_apps` in instance discovery
- `GET /v1/instance/bot-catalog`
- `GET /v1/oauth2/applications/{application_id}/public`

Both have their own application IDs. They run inside Micup's services without a public bot Gateway session and can appear offline. Their tokens are not available to third parties and are refused for bot authentication.

**MicupBOT configuration** is user-only. Routes:

```text
GET    /v1/guilds/{guild_id}/micupbot/settings
PATCH  /v1/guilds/{guild_id}/micupbot/settings
GET    /v1/guilds/{guild_id}/micupbot/automod
PUT    /v1/guilds/{guild_id}/micupbot/automod
GET    /v1/guilds/{guild_id}/micupbot/reaction-roles
POST   /v1/guilds/{guild_id}/micupbot/reaction-roles
DELETE /v1/guilds/{guild_id}/micupbot/reaction-roles/{message_id}
GET    /v1/guilds/{guild_id}/micupbot/level-rewards
PUT    /v1/guilds/{guild_id}/micupbot/level-rewards
GET    /v1/guilds/{guild_id}/micupbot/leaderboard
GET    /v1/guilds/{guild_id}/micupbot/warnings/{user_id}
DELETE /v1/guilds/{guild_id}/micupbot/warnings/{user_id}
DELETE /v1/guilds/{guild_id}/micupbot/warnings/{user_id}/{warning_id}
```

All require a signed-in user and guild membership. The caller needs the documented guild permissions (`MANAGE_GUILD`, sometimes `MANAGE_ROLES`, `MANAGE_MESSAGES`, `MODERATE_MEMBERS`). The instance flag `feature_micupbot` must be enabled; otherwise writes return `403 FEATURE_TEMPORARILY_DISABLED` while reads keep working.

**Micup Roster configuration** is also user-only:

```text
GET    /v1/guilds/{guild_id}/roster-events
POST   /v1/guilds/{guild_id}/roster-events
PATCH  /v1/guilds/{guild_id}/roster-events/{event_id}
DELETE /v1/guilds/{guild_id}/roster-events/{event_id}
```

The feature flag is `feature_guild_roster`. Creating or updating an event checks the caller's channel permissions, not just `MANAGE_GUILD`.

### 13.2 Feed

The feed is a social timeline: posts with text, up to four images, a video, or a poll; a home feed of followed accounts; Discover; categories; topics; likes; comments; replies; and notifications.

- Every feed route except the Bunny Stream webhook requires a **user session**.
- A bot or OAuth2 bearer is refused with `403 ACCESS_DENIED`.
- `feature_feed` must be enabled; otherwise session routes return `403 FEATURE_TEMPORARILY_DISABLED`.
- New posts are **not** pushed over the Gateway. Clients read the home feed.
- A public post can be read by any signed-in account while the author's profile privacy is `EVERYONE`.
- Video posts use Bunny Stream, TUS upload, encoding, and content scanning. A video becomes a post only after it passes.
- Polls allow one vote per account; votes cannot be changed.

Key routes:

```text
GET    /v1/feed/home
GET    /v1/feed/discover
GET    /v1/feed/categories
GET    /v1/feed/topics
GET    /v1/feed/notifications
POST   /v1/feed/posts
GET    /v1/feed/posts/{post_id}
DELETE /v1/feed/posts/{post_id}
PUT    /v1/feed/posts/{post_id}/like
POST   /v1/feed/posts/{post_id}/comments
GET    /v1/feed/posts/{post_id}/comments
POST   /v1/feed/posts/{post_id}/comments/{comment_id}/replies
POST   /v1/feed/views
```

### 13.3 Shop and premium

The shop sells, through the Payriff payment page, in Azerbaijani manat (AZN):

| SKU | Product |
| --- | --- |
| `cosmetic:<item_id>` | Owned cosmetic item |
| `bundle:<bundle_id>` | Bundle of 2–10 owned items |
| `premium:1`, `premium:3`, `premium:12` | Premium time |
| `visionary` | Lifetime premium with a Visionary number |
| `boost:30` | One boost slot for 30 days |
| `username:1`, `username:2`, `username:3` | Username card for a 1–3 character username |

- All shop routes are **user-only**, except code previews and payment callbacks.
- Catalog, quote, and order routes work only on a self-hosted instance in the `mirror` premium mode.
- Sales switches start off. Both read false while Payriff is unconfigured.
- The server computes every amount; a client sends only the product.
- Prices are set in US cents, converted to qəpik at an admin rate, and displayed with an informational TL estimate.
- Premium has monthly/yearly subscriptions, gifts, and lifetime Visionary.
- Do not build a bot that buys products or reads another account's orders.

### 13.4 Badges and cosmetics

- Badge inventory is user-only (`GET /v1/users/@me/badges`).
- A badge is built-in (staff, partner, bug hunter, premium, premium package, premium sequence) or from a catalog.
- Hiding a badge only changes what other readers see; the account keeps it.
- Cosmetics include avatar decorations, ultra decorations, profile effects, nameplates, and 3D room character parts.
- Cosmetic routes are user-only and require no permission.

### 13.5 Referral

- Every account has one personal code: 10 characters from `a`–`z` and `2`–`7`.
- A referral link is `/r/<code>`.
- A new account registers with `referral_code`.
- A referral is `pending`, `valid`, or `rejected`.
- Rewards are cosmetic items, premium days, or off-site rewards delivered by staff.
- The programme starts closed. While closed, the route reports `enabled: false` and returns no code.
- The route is user-only.

### 13.6 Interest onboarding

- A one-time picker of interest cards, grouped by region and category.
- Choosing an option joins the account to the guilds behind the cards.
- While an account must finish it, `READY` carries `interest_onboarding_required: true`.
- The account cannot skip it.
- Bot and system accounts never get the picker.
- The routes are user-only:

```text
GET  /v1/users/@me/interest-onboarding
POST /v1/users/@me/interest-onboarding
```

### 13.7 Usernames and tags

- Micup's live deployment uses `tag_style: "none"`.
- On a `none` deployment, usernames are unique and shown without a tag; every non-bot account has discriminator `0000`.
- Bot accounts always have a four-digit discriminator.
- The instance may run a username migration. Discovery publishes `username_migration` with `state` (`legacy`, `preparing`, `selecting`), `epoch`, and deadlines.
- After `forced_at`, an account without a selected username must choose one before the actions listed in the user object.
- Do not assume Discord's `#1234` tag model. Read `tag_style`.

### 13.8 Guild discovery

Guilds can be listed in Micup's public discovery directory when they have the `DISCOVERABLE` feature. The discovery API is public and returns guild cards with approximate member and presence counts. Discovery listing management is a guild-owner operation, not a bot operation.

### 13.9 What bots cannot do

The following are user-only and return `403 ACCESS_DENIED` to a bot token or OAuth2 bearer:

- MicupBOT and Micup Roster settings
- Feed reads and writes
- Table games and voice seats
- Shop, orders, premium self-service
- Badges, cosmetics, referrals, interest onboarding
- User profile, relationships, DMs, settings, read states
- Calling other bots' interactions (`POST /v1/interactions`)
- Admin API

A bot can:

- send, edit, delete, pin, and react to messages where it has permission;
- manage guild resources where it has permission and hierarchy;
- create and manage webhooks;
- register commands and answer interactions;
- read guilds, channels, members, roles, invites, and audit logs it can access;
- join voice under the normal placement rules, if it implements LiveKit media itself.
---

## 6. Guilds, channels, roles, and permissions

### 6.1 Guilds

A **guild** is a community with channels, roles, members, and configuration. The Micup UI often calls it a **community** or **server**; the API field is always `guild`.

A guild object can contain:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | snowflake | Guild ID |
| `name` | string | Guild name, 1–100 characters |
| `icon` | ?string | Icon hash |
| `banner` | ?string | Banner hash; null without the `BANNER` feature |
| `banner_width`, `banner_height` | ?integer | Banner dimensions |
| `splash`, `embed_splash` | ?string | Invite splash hashes |
| `vanity_url_code` | ?string | Custom invite code |
| `owner_id` | snowflake | Owner user ID |
| `system_channel_id` | ?snowflake | Channel for system messages |
| `system_channel_flags` | integer | System message switches |
| `rules_channel_id` | ?snowflake | Rules channel |
| `afk_channel_id` | ?snowflake | AFK voice channel |
| `afk_timeout` | integer | AFK timeout in seconds, 60–3600 |
| `features` | array[string] | Guild feature names |
| `verification_level` | integer | Membership verification level |
| `mfa_level` | integer | MFA requirement for elevated actions |
| `nsfw_level` | integer | Derived from `nsfw` |
| `nsfw` | boolean | Adult-content guild |
| `content_warning_level` | integer | Guild content-warning level |
| `content_warning_text` | ?string | Custom warning text, max 200 characters |
| `explicit_content_filter` | integer | Explicit-content filter level |
| `default_message_notifications` | integer | Notification default |
| `message_history_cutoff` | ?ISO8601 timestamp | Earliest message visible without `READ_MESSAGE_HISTORY` |
| `permissions` | decimal string | Caller's computed guild permissions, when returned |
| `roles` | array[role] | Roles, when returned |
| `channels` | array[channel] | Visible channels, when returned |
| `member_count`, `online_count` | integer | Counts held by the Gateway |
| `premium_tier` | integer | Boost level, 0–3 |
| `premium_subscription_count` | integer | Active boosts |

Guild features that affect behavior include `DISCOVERABLE`, `PARTNERED`, `VERIFIED`, `VIP_VOICE`, `ANIMATED_ICON`, `ANIMATED_BANNER`, `BANNER`, `INVITE_SPLASH`, `RAID_DETECTED`, `VISIONARY`, `EXPRESSION_PURGE_ALLOWED`, `UNAVAILABLE_HIDDEN`, `UNAVAILABLE_FOR_EVERYONE`, and `UNAVAILABLE_FOR_EVERYONE_BUT_STAFF`.

### 6.2 Channel types

| Value | Type | Meaning |
| --- | --- | --- |
| `0` | `GUILD_TEXT` | Guild text channel |
| `1` | `DM` | One-to-one direct message |
| `2` | `GUILD_VOICE` | Guild voice channel; also has text messages, pins, and slowmode |
| `3` | `GROUP_DM` | Group direct message |
| `4` | `GUILD_CATEGORY` | Category; owns no parent and no messages |
| `5` | `GUILD_ANNOUNCEMENT` | Announcement channel whose messages can be published to followers |
| `998` | `GUILD_LINK` | Link channel with a destination URL and no messages |
| `999` | `DM_PERSONAL_NOTES` | Personal notes channel |

A message can be created in a text-bearing channel: `GUILD_TEXT`, `GUILD_VOICE`, `GUILD_ANNOUNCEMENT`, `DM`, `GROUP_DM`, and personal notes. A category and a link channel cannot hold messages.

A guild channel object has `permission_overwrites`; a DM has `recipients`; a group DM can have `name`, `icon`, `owner_id`, and `nicks`. A guild voice channel has `bitrate`, `user_limit`, `voice_connection_limit`, `rtc_region`, and optional `room_theme`.

### 6.3 Permission bits

Permissions are an unsigned 64-bit mask. On the wire, every permission field is a **decimal string**. Request fields also accept a JSON number that is a safe integer. The maximum accepted value is `9223372036854775807`.

| Bit | Name | Decimal | Grants |
| --- | --- | ---: | --- |
| `1 << 0` | `CREATE_INSTANT_INVITE` | 1 | Create guild channel invites |
| `1 << 1` | `KICK_MEMBERS` | 2 | Remove members |
| `1 << 2` | `BAN_MEMBERS` | 4 | Create and revoke bans |
| `1 << 3` | `ADMINISTRATOR` | 8 | Every permission; bypasses channel overwrites |
| `1 << 4` | `MANAGE_CHANNELS` | 16 | Create, modify, reorder, and delete channels |
| `1 << 5` | `MANAGE_GUILD` | 32 | Modify guild settings and guild-level resources |
| `1 << 6` | `ADD_REACTIONS` | 64 | Add reactions |
| `1 << 7` | `VIEW_AUDIT_LOG` | 128 | Read the audit log |
| `1 << 8` | `PRIORITY_SPEAKER` | 256 | Priority speaker; client-evaluated |
| `1 << 9` | `STREAM` | 512 | Publish camera and screen share |
| `1 << 10` | `VIEW_CHANNEL` | 1,024 | View a channel |
| `1 << 11` | `SEND_MESSAGES` | 2,048 | Send messages and publish own announcements |
| `1 << 12` | `SEND_TTS_MESSAGES` | 4,096 | Request TTS playback |
| `1 << 13` | `MANAGE_MESSAGES` | 8,192 | Delete/edit others' messages, bulk delete, remove reactions |
| `1 << 14` | `EMBED_LINKS` | 16,384 | Have links expanded into embeds |
| `1 << 15` | `ATTACH_FILES` | 32,768 | Attach files |
| `1 << 16` | `READ_MESSAGE_HISTORY` | 65,536 | Read messages before the history cutoff |
| `1 << 17` | `MENTION_EVERYONE` | 131,072 | Use `@everyone` and `@here` |
| `1 << 18` | `USE_EXTERNAL_EMOJIS` | 262,144 | Use another guild's emoji |
| `1 << 20` | `CONNECT` | 1,048,576 | Connect to voice |
| `1 << 21` | `SPEAK` | 2,097,152 | Transmit audio |
| `1 << 22` | `MUTE_MEMBERS` | 4,194,304 | Moderator-mute members |
| `1 << 23` | `DEAFEN_MEMBERS` | 8,388,608 | Moderator-deafen members |
| `1 << 24` | `MOVE_MEMBERS` | 16,777,216 | Move or disconnect voice members |
| `1 << 25` | `USE_VAD` | 33,554,432 | Use voice activity detection; client-evaluated |
| `1 << 26` | `CHANGE_NICKNAME` | 67,108,864 | Change own nickname |
| `1 << 27` | `MANAGE_NICKNAMES` | 134,217,728 | Change others' nicknames |
| `1 << 28` | `MANAGE_ROLES` | 268,435,456 | Manage roles and channel overwrites |
| `1 << 29` | `MANAGE_WEBHOOKS` | 536,870,912 | Manage webhooks and follow announcements |
| `1 << 30` | `MANAGE_EXPRESSIONS` | 1,073,741,824 | Modify/delete others' emojis and stickers |
| `1 << 37` | `USE_EXTERNAL_STICKERS` | 137,438,953,472 | Use another guild's stickers |
| `1 << 40` | `MODERATE_MEMBERS` | 1,099,511,627,776 | Apply/clear communication timeouts |
| `1 << 43` | `CREATE_EXPRESSIONS` | 8,796,093,022,208 | Create expressions; manage own |
| `1 << 51` | `PIN_MESSAGES` | 2,251,799,813,685,248 | Pin and unpin messages |
| `1 << 52` | `BYPASS_SLOWMODE` | 4,503,599,627,370,496 | Send without waiting for slowmode |
| `1 << 53` | `UPDATE_RTC_REGION` | 9,007,199,254,740,992 | Change a voice channel's region |
| `1 << 54` | `VIEW_CHANNEL_MEMBERS` | 18,014,398,509,481,984 | View a channel's member list |

Bits `19`, `31`–`36`, `38`, `39`, `41`, `42`, `44`–`50`, and `55`–`63` are unassigned and are discarded from writes.

`ADMINISTRATOR` grants the complete 64-bit mask and skips channel overwrites. Removing `ADMINISTRATOR` from every role is the only way to restrict such a member.

`VIEW_CHANNEL_MEMBERS` is feature-gated. A write must declare `view_channel_members_permission` in the `X-Fluxer-Features` header; otherwise the bit keeps its stored value. On role creation, supplying `permissions` without the feature always clears the bit.

### 6.4 Permission computation

1. The guild owner receives the complete mask.
2. A non-member receives an empty mask.
3. Start with the `@everyone` role's permissions. The everyone role ID equals the guild ID.
4. Union the permissions of every role assigned to the member.
5. If `ADMINISTRATOR` is present, return the complete mask and stop.
6. If a channel was named, apply its overwrites in this order:
   - `@everyone` overwrite: remove denied bits, then add allowed bits.
   - Union of the member's role overwrites: remove all denied bits, then add all allowed bits. An allow on one role defeats a deny on another.
   - Member overwrite: remove denied bits, then add allowed bits.

A category is visible when at least one child channel is visible. A live voice connection grants **virtual access** to its channel when a role or overwrite change would otherwise hide it; virtual access also grants `SPEAK` and `STREAM` there and ends when the connection ends.

### 6.5 Communication timeouts

A communication timeout does **not** change the permission mask. It is a separate precondition. While `communication_disabled_until` is in the future, these fail with `403 COMMUNICATION_DISABLED`:

- sending a message,
- editing the caller's own message,
- publishing the caller's own announcement,
- adding a reaction,
- requesting an attachment upload,
- starting a typing indicator,
- changing the caller's own nickname.

A timeout has no effect in a private channel. A client that gates its UI on the permission mask must also compare `communication_disabled_until` to the current time.

### 6.6 Role hierarchy

- Roles are ranked by `position`; higher ranks first.
- Ties are broken by snowflake; the lower snowflake ranks first.
- A member's rank is their highest assigned role.
- A member with no role ranks below every role.
- A caller manages a role only when the caller's rank is **strictly above** the target role's rank.
- The guild owner manages every role.
- The `@everyone` role is always position `0` and is never a valid position or hoist-position target.
- `MANAGE_ROLES` alone is not enough; hierarchy is enforced separately.
- A non-owner cannot grant a permission they do not hold. The whole request fails with `MISSING_PERMISSIONS`.
- A role with `managed_kind: "booster"` is managed by the system; deleting it or assigning it by hand returns `MISSING_PERMISSIONS`.

Role object key fields: `id`, `name`, `color` (RGB integer), `color_secondary?`, `icon_emoji_id?`, `position`, `hoist_position?`, `permissions` (decimal string), `hoist`, `mentionable`, `unicode_emoji?`, `managed_kind?`.

### 6.7 Elevated permissions and MFA

These are elevated:

`KICK_MEMBERS`, `BAN_MEMBERS`, `ADMINISTRATOR`, `MANAGE_CHANNELS`, `MANAGE_GUILD`, `MANAGE_MESSAGES`, `MANAGE_ROLES`, `MANAGE_WEBHOOKS`, `MODERATE_MEMBERS`.

In a guild whose `mfa_level` is elevated, a non-owner without an enrolled authenticator receives `400 TWO_FACTOR_REQUIRED` after the permission check passes. The reported mask still contains the elevated bit; hide the action in the UI based on the guild MFA level and the account's enrollment state.

### 6.8 Guild and channel routes

| Method and path | Purpose |
| --- | --- |
| `GET /v1/users/@me/guilds` | List the caller's guilds |
| `GET /v1/guilds/{guild_id}` | Get a guild |
| `PATCH /v1/guilds/{guild_id}` | Modify a guild |
| `DELETE /v1/users/@me/guilds/{guild_id}` | Leave a guild |
| `POST /v1/guilds/{guild_id}/delete` | Delete a guild (owner) |
| `GET /v1/guilds/{guild_id}/channels` | List channels |
| `POST /v1/guilds/{guild_id}/channels` | Create a channel |
| `PATCH /v1/guilds/{guild_id}/channels` | Reorder channels |
| `GET /v1/channels/{channel_id}` | Get a channel |
| `PATCH /v1/channels/{channel_id}` | Modify a channel |
| `DELETE /v1/channels/{channel_id}` | Delete or leave a channel |
| `PUT /v1/channels/{channel_id}/permissions/{overwrite_id}` | Create/update an overwrite |
| `DELETE /v1/channels/{channel_id}/permissions/{overwrite_id}` | Delete an overwrite |
| `GET /v1/guilds/{guild_id}/members` | List members |
| `GET /v1/guilds/{guild_id}/members/{user_id}` | Get a member |
| `PATCH /v1/guilds/{guild_id}/members/{user_id}` | Modify a member, including timeout and voice move |
| `DELETE /v1/guilds/{guild_id}/members/{user_id}` | Kick a member |
| `PUT /v1/guilds/{guild_id}/members/{user_id}/roles/{role_id}` | Add a role |
| `DELETE /v1/guilds/{guild_id}/members/{user_id}/roles/{role_id}` | Remove a role |
| `GET /v1/guilds/{guild_id}/roles` | List roles |
| `POST /v1/guilds/{guild_id}/roles` | Create a role |
| `PATCH /v1/guilds/{guild_id}/roles/{role_id}` | Modify a role |
| `DELETE /v1/guilds/{guild_id}/roles/{role_id}` | Delete a role |
| `GET /v1/guilds/{guild_id}/bans` | List bans |
| `PUT /v1/guilds/{guild_id}/bans/{user_id}` | Ban a user |
| `DELETE /v1/guilds/{guild_id}/bans/{user_id}` | Revoke a ban |
| `GET /v1/guilds/{guild_id}/invites` | List guild invites |
| `POST /v1/channels/{channel_id}/invites` | Create an invite |
| `GET /v1/invites/{invite_code}` | Get an invite |
| `DELETE /v1/invites/{invite_code}` | Delete an invite |
| `GET /v1/guilds/{guild_id}/audit-logs` | Read the audit log |

Moderation notes:

- A kick requires `KICK_MEMBERS`.
- A ban requires `BAN_MEMBERS`. The ban body can include `delete_message_seconds` where the route supports it.
- A timeout is `PATCH /v1/guilds/{guild_id}/members/{user_id}` with `communication_disabled_until` set to an ISO timestamp or `null`. It requires `MODERATE_MEMBERS`.
- A voice move or disconnect uses the same member PATCH with `channel_id`, and requires `MOVE_MEMBERS`.
- Moderation requests accept `X-Audit-Log-Reason` where the operation records an audit entry.
- The bot needs its own role hierarchy and permissions; the caller's permission does not substitute for the bot's.

### 6.9 Common guild and channel errors

| Status | Code | Meaning |
| --- | --- | --- |
| 403 | `MISSING_PERMISSIONS` | Caller lacks the required permission |
| 403 | `MISSING_ACCESS` | Caller cannot access the guild or resource |
| 403 | `ACCESS_DENIED` | Credential type is not allowed |
| 403 | `COMMUNICATION_DISABLED` | Member is timed out |
| 403 | `UNKNOWN_GUILD` disguised as 404 | Guild does not exist |
| 404 | `UNKNOWN_CHANNEL` | Channel does not exist or is not visible |
| 404 | `UNKNOWN_USER` | User does not exist |
| 400 | `INVALID_CHANNEL_TYPE` | Operation does not apply to this channel type |
| 400 | `TWO_FACTOR_REQUIRED` | Elevated action requires MFA |

---

## 7. Messages

### 7.1 Message object

A message object includes at least:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | snowflake | Message ID |
| `channel_id` | snowflake | Channel ID |
| `guild_id?` | snowflake | Guild ID, absent in a DM |
| `author` | user | Author |
| `member?` | guild member | Author's guild member state |
| `content` | string | Text |
| `timestamp` | ISO8601 | Creation time |
| `edited_timestamp` | ?ISO8601 | Last edit time |
| `tts` | boolean | TTS requested |
| `mention_everyone` | boolean | Active `@everyone`/`@here` |
| `mentions` | array[user] | Mentioned users |
| `mention_roles` | array[snowflake] | Mentioned role IDs |
| `attachments` | array[attachment] | Files |
| `embeds` | array[embed] | Embeds |
| `reactions?` | array[reaction] | Reaction summaries |
| `nonce?` | string | Caller nonce |
| `pinned` | boolean | Pinned |
| `webhook_id?` | snowflake | Webhook author |
| `type` | integer | Message type |
| `flags` | integer | Message flags |
| `components?` | array[component] | Buttons and selects |
| `message_reference?` | object | Reply/forward reference |
| `referenced_message?` | ?message | Replied-to message |
| `message_snapshots?` | array | Forward snapshots |
| `sticker_items?` | array | Stickers |
| `thread?` | channel | Associated thread |

A message may have `content`, embeds, attachments, stickers, or a favourite meme, but not none.

### 7.2 Message types

| Value | Type | Notes |
| --- | --- | --- |
| `0` | `DEFAULT` | Ordinary message |
| `1` | `RECIPIENT_ADD` | Group DM recipient added |
| `2` | `RECIPIENT_REMOVE` | Group DM recipient removed |
| `3` | `CALL` | Call system message |
| `4` | `CHANNEL_NAME_CHANGE` | Channel renamed |
| `5` | `CHANNEL_ICON_CHANGE` | Channel icon changed |
| `6` | `CHANNEL_PINNED_MESSAGE` | Pin system message |
| `7` | `USER_JOIN` | User joined |
| `12` | `CHANNEL_FOLLOW_ADD` | Channel started following an announcement channel |
| `19` | `REPLY` | Reply message |

Only `DEFAULT` and `REPLY` can be modified, pinned, unpinned, or used as a reply target. Only `DEFAULT`, `CHANNEL_PINNED_MESSAGE`, `USER_JOIN`, `CHANNEL_FOLLOW_ADD`, and `REPLY` can be deleted.

### 7.3 Message flags

| Bit | Value | Name | Sendable? |
| --- | ---: | --- | --- |
| `1 << 0` | 1 | `CROSSPOSTED` | Server-set |
| `1 << 1` | 2 | `IS_CROSSPOST` | Server-set |
| `1 << 2` | 4 | `SUPPRESS_EMBEDS` | Yes |
| `1 << 3` | 8 | `SOURCE_MESSAGE_DELETED` | Server-set |
| `1 << 12` | 4,096 | `SUPPRESS_NOTIFICATIONS` | Yes |
| `1 << 13` | 8,192 | `VOICE_MESSAGE` | Yes |
| `1 << 15` | 32,768 | `IS_COMPONENTS_V2` | Bots only; currently feature-gated off |

`EPHEMERAL` (`1 << 6`, 64) is **not** a message flag. It exists only on an interaction response's `flags` field.

A create masks the supplied flags down to the sendable set and silently drops every other bit. A modify replaces only the sendable bits and leaves other stored bits untouched.

### 7.4 Create a message

```http
POST /v1/channels/{channel_id}/messages
Authorization: Bot <token>
Content-Type: application/json

{
  "content": "Merhaba, Micup!",
  "nonce": "req-001",
  "tts": false,
  "embeds": [],
  "allowed_mentions": {"parse": []}
}
```

Body fields:

| Field | Type | Notes |
| --- | --- | --- |
| `content?` | string | Text; effective limit from discovery |
| `nonce?` | string \| integer | Idempotency key, 1–32 characters or a safe non-negative integer |
| `tts?` | boolean | Forced to false without `SEND_TTS_MESSAGES` |
| `embeds?` | array[embed] | At most `max_embeds_per_message`, default 10 |
| `attachments?` | array[attachment] | Pre-uploaded attachments |
| `message_reference?` | object | Reply or forward |
| `allowed_mentions?` | object | Mention policy |
| `flags?` | integer | Sendable flags only |
| `favorite_meme_id?` | snowflake | Copied into a new attachment |
| `sticker_ids?` | array[snowflake] | At most 3 |
| `components?` | array[component] | Bots only; gated by feature keys |

Permissions and preconditions:

- Guild channel: `VIEW_CHANNEL` and `SEND_MESSAGES`.
- Embeds: `EMBED_LINKS`.
- Attachments: `ATTACH_FILES`.
- Active everyone mention: `MENTION_EVERYONE`.
- Favourite meme: both `EMBED_LINKS` and `ATTACH_FILES`.
- Slowmode applies to non-bots unless the author has `BYPASS_SLOWMODE`.
- A timed-out member gets `403 COMMUNICATION_DISABLED`.
- A one-to-one DM may fail with `400 CANNOT_SEND_MESSAGES_TO_USER` due to the recipient's DM policy or relationship state.
- A new account under a new-conversation limit can get `403 NEW_CONVERSATIONS_LIMITED`.
- An unclaimed account can send only to its personal notes; otherwise `400 UNCLAIMED_ACCOUNT_CANNOT_SEND_MESSAGES`.
- A non-bot must have started a session; otherwise `MUST_START_SESSION_BEFORE_SENDING`.

A bot or webhook author gets `max(resolved_max_message_length, 4000)`, so a bot can send at least 4,000 characters even on a deployment whose normal limit is 2,000.

### 7.5 Mentions

`allowed_mentions` controls which mentions written in the text become active:

| Field | Type | Meaning |
| --- | --- | --- |
| `parse?` | array[string] | `users`, `roles`, `everyone` |
| `users?` | array[snowflake] | At most 100 user IDs |
| `roles?` | array[snowflake] | At most 100 role IDs |
| `replied_user?` | boolean | Whether a reply mentions the referenced author; default true |

Rules:

- An absent object parses every category.
- `{}` parses **nothing**. This is the safest default for bot output.
- A non-empty `parse` cannot be combined with non-empty `users` or `roles`.
- `mention_everyone` is forced to false when the caller lacks `MENTION_EVERYONE`.
- `replied_user` is ignored in a one-to-one DM, in personal notes, and when the reply target has the same author.

### 7.6 Embeds

A rich embed input supports:

```json
{
  "title": "Release 2.4.0",
  "description": "Voice reconnects are now instant.",
  "url": "https://example.com/releases/2.4.0",
  "timestamp": "2026-10-26T12:00:00.000Z",
  "color": 5793266,
  "author": {"name": "Micup", "url": "https://micup.gg", "icon_url": "https://example.com/icon.png"},
  "thumbnail": {"url": "https://example.com/thumb.png"},
  "image": {"url": "https://example.com/image.png"},
  "footer": {"text": "example.com", "icon_url": "https://example.com/icon.png"},
  "fields": [
    {"name": "Platform", "value": "Desktop", "inline": true}
  ]
}
```

- A message can hold at most 10 embeds by default.
- An embed requires `EMBED_LINKS` in a guild.
- Embed media URLs from the instance are signed by the Media Proxy when returned.
- A rich embed is stored as type `rich`. Link, image, video, audio, gifv, article, and Bluesky embeds are produced by the unfurler, not accepted as rich input.
- Do not put an attachment token or temporary signed URL in an embed if you need it to keep working; re-read the message to obtain fresh signed URLs.

### 7.7 Attachments and uploads

There are three attachment paths:

1. **Direct multipart** on message create. Send `payload_json` plus `files[n]`. The file index is the attachment ID.
2. **Pre-upload then reference.** Request an upload with `POST /v1/channels/{channel_id}/attachments`, upload the bytes, complete with `POST /v1/channels/{channel_id}/attachments/complete`, then send the returned attachment metadata in `attachments`.
3. **Presigned upload URLs**, when discovery reports `features.presigned_attachment_uploads: true`. Request the upload, PUT the bytes to the returned URL, then reference the attachment.

Attachment rules:

- The deployment's `max_attachments_per_message` defaults to 10.
- The deployment's `max_attachment_file_size` is in bytes.
- Every returned attachment URL is a signed Media Proxy URL. It is a bearer capability and expires.
- Refresh expired signed URLs with `POST /v1/attachments/refresh-urls`.
- Use the uploaded filename; do not invent a `Content-Type` that contradicts the file.
- A bot that cannot attach files receives a permission error, not a silent drop.

### 7.8 Reactions

Emoji path parameter forms:

- Unicode emoji: percent-encode the emoji, such as `%F0%9F%91%8D`.
- Custom emoji: `name:id`, percent-encoded, such as `party%3A1501314428688998182`.

Routes:

```text
PUT    /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}/{target_id}
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}
DELETE /v1/channels/{channel_id}/messages/{message_id}/reactions
GET    /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}
GET    /v1/channels/{channel_id}/messages/{message_id}/reactions/{emoji}/users
```

Adding a reaction requires `ADD_REACTIONS` and access to the message. Removing another user's reaction or all reactions requires `MANAGE_MESSAGES`. A reaction is idempotent for its author: adding the same reaction twice does not create a second count.

### 7.9 Pins, replies, forwards, and bulk deletion

- Pins:
  - `GET /v1/channels/{channel_id}/messages/pins`
  - `PUT /v1/channels/{channel_id}/pins/{message_id}`
  - `DELETE /v1/channels/{channel_id}/pins/{message_id}`
  - Pinning requires `PIN_MESSAGES`; only `DEFAULT` and `REPLY` messages can be pinned.
- Reply:
  - Set `message_reference: {"message_id": "...", "channel_id": "...", "type": 0}`.
  - A reply mentions its author by default unless `allowed_mentions.replied_user` is false.
- Forward:
  - Set `message_reference.type` to `1` and include `channel_id`.
  - The created message has `message_snapshots` and no `referenced_message`.
- Bulk delete:
  - `POST /v1/channels/{channel_id}/messages/bulk-delete` with `message_ids` (1–100).
  - Requires `MANAGE_MESSAGES` and elevated-MFA compliance.
  - Deletes matching messages only; unknown IDs are skipped.
  - Emits one `MESSAGE_DELETE_BULK`.

### 7.10 Common message errors

| Status | Code | Cause |
| --- | --- | --- |
| 400 | `CANNOT_SEND_EMPTY_MESSAGE` | No content, embed, attachment, meme, or sticker |
| 400 | `INVALID_MESSAGE_DATA` | Malformed JSON message body |
| 400 | `INVALID_FORM_BODY` | Field validation failed; inspect `errors` |
| 400 | `CANNOT_SEND_MESSAGES_TO_USER` | Recipient's DM policy blocks the send |
| 400 | `CANNOT_EXECUTE_ON_DM` | Route is guild-only |
| 403 | `COMMUNICATION_DISABLED` | Author is timed out |
| 403 | `MISSING_PERMISSIONS` | Missing channel or guild permission |
| 403 | `NEW_CONVERSATIONS_LIMITED` | New-account DM limit |
| 403 | `NSFW_CONTENT_AGE_RESTRICTED` | Age check required |
| 404 | `UNKNOWN_CHANNEL` | Channel not found or not visible |
| 404 | `UNKNOWN_MESSAGE` | Message not found |
| 413 | `FILE_SIZE_TOO_LARGE` | Upload exceeds the deployment limit |
| 429 | `RATE_LIMITED` | Route or global bucket denial |

---

## 8. Application commands and interactions

### 8.1 The interaction model

Micup's interaction model is Discord-shaped but has one important transport difference:

- A person runs a slash command, clicks a button or select menu, types into an autocomplete option, or submits a modal.
- Micup sends **one** `INTERACTION_CREATE` Dispatch over the Gateway to **exactly one** session of the bot.
- The bot answers within **3 seconds** by POSTing to the callback route with the interaction token.
- The bot may edit the original response and send up to **20 followups** for **15 minutes**.
- There is **no HTTP endpoint that receives interactions**, **no `PING` interaction**, and **no Ed25519 signature verification**.

Delivery is not retried. If no session of the bot is connected, the interaction is not delivered. If the bot misses the 3-second window, the user sees no answer.

### 8.2 Register commands

A command is either **global** or **guild-scoped**.

- A global command works in every guild the bot is in, and in a one-to-one DM when `dm_permission` is true.
- A guild command works only in that guild and never in a DM.
- When a guild command and a global command share a name, the guild command hides the global one in that guild.
- Each scope holds one command set with a version. Every write replaces the set atomically and returns a new `ETag`.

Registration routes:

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
GET    /v1/applications/{application_id}/guilds/{guild_id}/commands/{command_id}
PATCH  /v1/applications/{application_id}/guilds/{guild_id}/commands/{command_id}
DELETE /v1/applications/{application_id}/guilds/{guild_id}/commands/{command_id}
```

`PUT` is a **bulk overwrite** and replaces the whole set. `POST` creates one command. The command routes require the application's own bot token; any other credential is refused.

Registration rules:

- A guild command requires the bot to be a member of that guild.
- `feature_bot_interactions` must be `1` and `feature_bot_interactions_blocked` must be `0`; otherwise every registration route returns `403 APPLICATION_COMMAND_FEATURE_DISABLED`.
- A request body with a declared length over 540,672 bytes is refused with `413 FILE_SIZE_TOO_LARGE`.
- `If-Match` is optional. A stale ETag returns `412 APPLICATION_COMMAND_VERSION_CONFLICT`.
- discord.js and discord.py send neither precondition header, so concurrent writers can conflict.

### 8.3 Command object

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | snowflake | Command ID, kept while the name stays the same |
| `application_id` | snowflake | Owning application |
| `version` | snowflake | Changes when the command changes |
| `guild_id?` | snowflake | Guild for a guild command |
| `type` | integer | Always `1` (chat input) |
| `name` | string | Command name |
| `description` | string | 1–100 characters |
| `options` | array[option] | Up to 25 |
| `default_member_permissions` | ?string | Decimal permission mask; null for none |
| `dm_permission` | boolean | Whether a global command works in a DM with the bot |
| `contexts?` | array[integer] | `0` guild, `1` DM with the bot, `2` private channels (no effect) |

A command's request body accepts `type`, `name`, `description`, `options`, `default_member_permissions`, and `dm_permission`.

Accepted but discarded for Discord-library compatibility:

- `name_localizations`
- `description_localizations`
- `default_permission`
- `nsfw: false`
- `integration_types`

Rejected or unsupported:

- `nsfw: true` → `400 INVALID_FORM_BODY`
- Option type `9` (mentionable) → rejected
- Option type `11` (attachment) → rejected
- User-installed applications → not supported
- Right-click message/user commands → not offered
- Premium buttons in interaction responses → not offered

`default_member_permissions` only **hides** the command from members who lack the bits. It does not authorize the command. Enforce authorization in the bot.

### 8.4 Command names

- 1–32 Unicode code points after NFC normalization.
- Lowercase letters of any script, digits, `-`, and `_`.
- `yardım` and `seviye-ödül` are valid.
- Uppercase letters and other characters are rejected.
- Option names, subcommand names, and group names follow the same rules and must be unique among siblings.

### 8.5 Option types

| Value | Type | Notes |
| --- | --- | --- |
| `1` | `SUB_COMMAND` | Leaf command |
| `2` | `SUB_COMMAND_GROUP` | Group of subcommands |
| `3` | `STRING` | Text; max 6,000 code points |
| `4` | `INTEGER` | Whole number |
| `5` | `BOOLEAN` | True/false |
| `6` | `USER` | Resolved in the interaction |
| `7` | `CHANNEL` | Resolved in the interaction |
| `8` | `ROLE` | Resolved in the interaction |
| `10` | `NUMBER` | Fractional number |

An option can have:

- `required` (default false)
- up to 25 `choices` for string/integer/number
- `autocomplete` for string/integer/number
- `min_value` / `max_value` for integer/number
- `min_length` / `max_length` for string, 0–6,000
- up to 25 `channel_types` for a channel option
- nested `options` for a subcommand or group

Constraints:

- `choices` and `autocomplete` cannot both be set.
- `min_value` cannot exceed `max_value`.
- `min_length` cannot exceed `max_length`.
- Required options come before optional options.
- A command's options are either all value options or all subcommands/groups.
- A group holds 1–25 subcommands; a subcommand holds value options; nothing nests deeper.
- A subcommand has no `required` field.

Limits:

| Limit | Value |
| --- | --- |
| Commands per scope | 100 |
| Options per command or subcommand | 25 |
| Choices per option | 25 |
| Serialised command size | 32,768 bytes |
| Serialised command set | 524,288 bytes |

### 8.6 Example command registration

```http
PUT /v1/applications/{application_id}/guilds/{guild_id}/commands
Authorization: Bot <token>
Content-Type: application/json

[
  {
    "name": "yardım",
    "description": "Komut kategorilerini gösterir",
    "options": [
      {
        "type": 3,
        "name": "kategori",
        "description": "Gösterilecek kategori",
        "required": false,
        "autocomplete": true
      }
    ],
    "dm_permission": true
  }
]
```

The response is the stored command set. The `ETag` response header identifies its version.

### 8.7 Interaction object

The `d` payload of `INTERACTION_CREATE` contains:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | snowflake | Interaction ID |
| `application_id` | snowflake | Bot application |
| `type` | integer | Interaction type |
| `token` | string | Interaction token; a credential |
| `version` | integer | Always `1` |
| `data` | object | Command, component, autocomplete, or modal data |
| `guild_id?` | snowflake | Guild, absent in a DM |
| `channel_id` | snowflake | Channel |
| `channel` | partial channel | Channel representation |
| `member?` | interaction member | Invoking member in a guild |
| `user?` | partial user | Invoking user in a DM |
| `message?` | message | Message whose component was used |
| `app_permissions` | string | Bot's permissions in the channel |
| `locale` | string | Invoking user's locale |
| `guild_locale?` | string | Guild locale |
| `entitlements` | array | Always empty |
| `context` | integer | `0` guild, `1` DM with the bot |
| `attachment_size_limit` | integer | Largest upload in bytes |

The interaction member object adds `permissions` (channel-scoped decimal string) and `flags` (always `0`, kept for Discord libraries).

Interaction types:

| Value | Type | Delivered? |
| --- | --- | --- |
| `1` | `PING` | Never |
| `2` | `APPLICATION_COMMAND` | Yes |
| `3` | `MESSAGE_COMPONENT` | Yes |
| `4` | `APPLICATION_COMMAND_AUTOCOMPLETE` | Yes |
| `5` | `MODAL_SUBMIT` | Yes |

### 8.8 Interaction tokens and timing

- The first response must arrive within **3 seconds** of interaction creation.
- The token remains usable for **15 minutes** after creation.
- The token answers only **one first callback**. A second callback returns `409 INTERACTION_TOKEN_CONSUMED`.
- A callback after 3 seconds returns `410 INTERACTION_EXPIRED`.
- Up to **20 followups** are allowed.
- The token is a bearer credential for that interaction. Never log or store it.
- Micup stores only a SHA-256 hash of its secret part.
- A started interaction stays answerable for its 15-minute window even if the command is edited or deleted.
- The bot's membership and permissions are rechecked on every answer.

### 8.9 Callback types

| Value | Type | Purpose |
| --- | --- | --- |
| `1` | `PONG` | Registry value; **refused** on the callback route because `PING` is never delivered |
| `4` | `CHANNEL_MESSAGE_WITH_SOURCE` | Send a message |
| `5` | `DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE` | Acknowledge and send later |
| `6` | `DEFERRED_UPDATE_MESSAGE` | Acknowledge a component update without changing it |
| `7` | `UPDATE_MESSAGE` | Update the message that held the component |
| `8` | `APPLICATION_COMMAND_AUTOCOMPLETE_RESULT` | Return autocomplete choices |
| `9` | `MODAL` | Open a modal |

Validity by interaction type:

- Type `4` works for command and component interactions.
- Type `5` works for command and component interactions.
- Type `6` works for component interactions.
- Type `7` works for component interactions.
- Type `8` works only for autocomplete interactions and cannot be deferred.
- Type `9` works only as the first answer to a command or component interaction, and requires the bot's `feature_bot_modals` key.

Callback body:

```json
{
  "type": 4,
  "data": {
    "content": "Merhaba!",
    "flags": 64,
    "allowed_mentions": {"parse": []}
  }
}
```

For a deferred response:

```json
{"type": 5}
```

Then edit the original response with the interaction token.

### 8.10 Response message object

An interaction response `data` can contain:

- `content`
- `embeds`
- `components`
- `allowed_mentions`
- `flags`
- `tts`
- `attachments`
- `poll` (where supported by the message schema)

`flags: 64` makes the response **private** (ephemeral):

- Only the invoking user sees it.
- It is not written to the channel.
- It does not appear in search, notifications, or unread counts.
- It is delivered to the invoking session as an `INTERACTION_RESPONSE_CREATE` Dispatch and can be recovered with `GET /v1/interactions/{interaction_id}/response`.

`flags` on an interaction response is not a message flag. It is the private-response mechanism.

### 8.11 Components

Components are written on the bot's own messages through message create, message modify, or an interaction response.

Limits:

- At most **5 action rows** per message.
- Each action row holds **1–5 buttons**, or **exactly one select menu**.
- A button label is at most 80 characters.
- A `custom_id` is 1–100 characters and unique on the message.
- A bot's edit that omits `components` keeps the stored components.

Component types:

| Value | Type | Placement |
| --- | --- | --- |
| `1` | `ACTION_ROW` | Top level |
| `2` | `BUTTON` | In an action row |
| `3` | `STRING_SELECT` | Alone in an action row |
| `4` | `TEXT_INPUT` | In a modal only |
| `5` | `USER_SELECT` | Alone in an action row |
| `6` | `ROLE_SELECT` | Alone in an action row |
| `7` | `MENTIONABLE_SELECT` | Alone in an action row |
| `8` | `CHANNEL_SELECT` | Alone in an action row |
| `9` | `SECTION` | Components v2 |
| `10` | `TEXT_DISPLAY` | Components v2 and modals |
| `11` | `THUMBNAIL` | Components v2 |
| `12` | `MEDIA_GALLERY` | Components v2 |
| `13` | `FILE` | Components v2 |
| `14` | `SEPARATOR` | Components v2 |
| `17` | `CONTAINER` | Components v2 |
| `18` | `LABEL` | Modal only |

Button styles:

| Value | Style | Notes |
| --- | --- | --- |
| `1` | `PRIMARY` | Main action |
| `2` | `SECONDARY` | Secondary action |
| `3` | `SUCCESS` | Positive action |
| `4` | `DANGER` | Destructive action |
| `5` | `LINK` | Opens a URL; no `custom_id` |

A button needs a label or an emoji. A non-link button needs a `custom_id`. A link button needs an `https` or `http` URL, at most 512 characters, with no username or password, and not on the instance blocklist. A click on a link button is not sent to the bot.

Select menus:

- Hold 1–25 options on a string select.
- `min_values` 0–25, `max_values` 1–25, `min_values <= max_values`.
- Option values are unique.
- A returned value must be one of the stored options, or an entity the clicker can see.
- An invalid value returns `400 INVALID_COMPONENT_VALUE`.

Security:

- `custom_id` and every select value are **untrusted user input**. Validate them like any request parameter.
- Never put secrets, permissions, or authorization decisions in a `custom_id`.
- Enforce authorization in the bot. `default_member_permissions` only hides the command.
- A bot cannot click its own components; the click route returns `403 ACCESS_DENIED` for a bot token. Bots still write components.

### 8.12 Modals

A modal holds **1–5** of these:

| Component | Content |
| --- | --- |
| Label (`18`) | A `label` of at most 45 characters, optional `description` of at most 100, and one text input |
| Action row (`1`) | One text input with its own `label` |
| Text display (`10`) | Markdown text |

A text input (`4`) has:

- `custom_id`
- `style`: `1` short or `2` paragraph
- `min_length` / `max_length` up to 4,000
- `required`
- optional prefilled `value`
- `placeholder` up to 100 characters

Selects and file uploads inside a modal are not offered and fail with `400`.

A modal submit arrives as interaction type `5`. Its `data.components` contains the submitted values. Respond with a normal callback type (for example `4` or `5`) or `7` where applicable.

### 8.13 Autocomplete

- Add `autocomplete: true` to a string, integer, or number option.
- As the user types, Micup sends interaction type `4`.
- The focused option's `value` is the raw typed text.
- Respond within **3 seconds** with callback type `8`.
- Return at most **25** choices.
- An autocomplete response cannot be deferred.
- An option cannot have both `choices` and `autocomplete`.

Example:

```json
{
  "type": 8,
  "data": {
    "choices": [
      {"name": "Moderasyon", "value": "mod"},
      {"name": "Seviye", "value": "seviye"}
    ]
  }
}
```

### 8.14 Followups

Use the interaction token:

```http
POST /v1/interactions/{interaction_id}/{interaction_token}/callback
Content-Type: application/json
```

Or use the bot-token followup route:

```http
POST /v1/interactions/{interaction_id}/followups
Authorization: Bot <token>
Idempotency-Key: <1-64 printable ASCII characters>
Content-Type: application/json
```

The followup route:

- Is bot-only.
- Requires the interaction to have been answered.
- Allows at most 20 followups.
- Returns the created message.
- Uses `Idempotency-Key`; the same key returns the same message, and reusing a key with a different body returns `409 INTERACTION_NONCE_CONFLICT`.
- Returns `410 INTERACTION_EXPIRED` after 15 minutes.
- Returns `503 INTERACTION_OUTCOME_UNKNOWN` when the result cannot be confirmed; retry with the same key.

The built-in callback route also accepts followups without an `Authorization` header, because Discord libraries use it.

### 8.15 Discord library configuration

**discord.js 14**

```js
import {Client, GatewayIntentBits} from 'discord.js';

const discovery = await (await fetch('https://micup.gg/.well-known/fluxer')).json();
const api = discovery.endpoints.api_public.replace(/\/$/, '');

const client = new Client({
  intents: [GatewayIntentBits.Guilds], // Micup ignores intents; this is a client-side choice
  rest: {api, version: '1'},
  ws: {version: 1},
});

client.once('ready', () => console.log(`Logged in as ${client.user.username}`));
client.login(process.env.MICUP_TOKEN);
```

- `rest.api` is the **public API base without `/v1`**; discord.js appends `/v1`.
- `rest.version: '1'` selects `/v1`.
- `ws.version: 1` appends `v=1` to the Gateway URL.
- discord.js fetches the Gateway URL with `GET /v1/gateway/bot`.
- Do not set a Discord CDN base.

**discord.py 2**

```python
import json
import os
import urllib.request

import discord
import discord.gateway
import discord.http
import yarl

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
client.run(TOKEN)
```

Critical: discord.py does **not** discover the Gateway URL. If you do not replace `DiscordWebSocket.DEFAULT_GATEWAY`, it will connect to Discord's Gateway and send the Micup token there. Always set it from `GET /v1/gateway/bot` or discovery.

### 8.16 Differences from Discord

| Topic | Micup behavior |
| --- | --- |
| API version | REST `/v1`, Gateway `v=1` |
| Receiving interactions | Gateway only; no public HTTP endpoint |
| Interaction signatures | None; no Ed25519, no `PING` |
| Gateway intents | None; no intent field, no privileged intents |
| `applications.commands` scope | None; `scope=bot` installs |
| User-installed apps | Not supported; `integration_types` discarded |
| Command localization | `name_localizations` / `description_localizations` discarded; default name in every locale |
| `nsfw: true` commands | Rejected |
| Right-click commands | Not offered |
| Attachment option type | Not offered |
| Mentionable option type | Not offered |
| Deferred public response | No placeholder message; the first edit of `@original` creates it |
| Private response | Not written to the channel; only the invoking user sees it |
| Components v2 | Feature-gated off by default |
| Premium button | Not offered |
| Files in interaction responses | Not offered |
| Files/selects in modals | Not offered |
| Followup route | Extra bot-token route with `Idempotency-Key` |
| Command write precondition | `If-Match` optional |
| Media URLs | Media Proxy, not Discord CDN |
| Bot invite | `scope=bot`, no `applications.commands` |

### 8.17 Interaction errors

| Status | Code | Cause |
| --- | --- | --- |
| 403 | `APPLICATION_COMMAND_FEATURE_DISABLED` | A bot or user feature key is closed |
| 503 | `APPLICATION_COMMAND_BOT_UNAVAILABLE` | No Gateway session took the interaction |
| 409 | `APPLICATION_COMMAND_STALE` | The command changed since the client loaded it |
| 410 | `INTERACTION_EXPIRED` | The 3-second or 15-minute window passed |
| 409 | `INTERACTION_TOKEN_CONSUMED` | The interaction was already answered |
| 409 | `INTERACTION_NONCE_CONFLICT` | A nonce or idempotency key was reused with a different body |
| 503 | `INTERACTION_OUTCOME_UNKNOWN` | The result could not be confirmed; retry with the same key |
| 400 | `UNSUPPORTED_COMPONENT` | Component type not offered |
| 400 | `INVALID_COMPONENT_VALUE` | Select value not on the component |
| 409 | `MODAL_NOT_OPEN` | Modal submit without an open modal |
| 404 | `UNKNOWN_WEBHOOK` | Interaction token or application ID does not match |
| 400 | `INVALID_FORM_BODY` | Body validation failed |

Rate limit for interaction callbacks and followups: **5 requests per second per interaction ID**.
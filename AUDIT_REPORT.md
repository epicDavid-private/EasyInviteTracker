# EasyInviteTracker — Security & Bug Audit Report

---

# 1. SyncInvitesForGuild irreversibly deletes all invite data before confirming the fetch succeeded

**Severity:** High

**File**
`src/Utils/SyncInvites.ts`

**Line(s)**
`11-14` (delete) vs `16-18` (async fetch)

**Summary**

`SyncInvitesForGuild` deletes every invite for a guild from the database **before** the asynchronous invite fetch completes. If the fetch returns an empty array (permission error, network failure, Discord outage), all tracked invite data for that guild is permanently lost.

**Why this is a bug**

Execution path:
1. `SyncInvitesForGuild(guild)` is called (from startup, `/sync`, or hourly refresh).
2. Line 11: `Database.prepare('DELETE FROM Invites WHERE guild_id = ?').run(guild.id)` — all invites for this guild are removed from the database.
3. Line 16: `const invites = await FetchInvites(guild)` — this is async and can fail.
4. `FetchInvites` catches its own errors internally and returns `[]` on failure (see `src/Utils/Parsers/FetchInvites.ts` lines 10-20).
5. The `for` loop at line 42 has zero iterations since `invites` is `[]`.
6. The function completes with zero invites in the database for this guild.

The check on line 18 (`if (invites.length >= MAX_INVITES_PER_GUILD)`) only guards against too many invites — it does NOT guard against zero. There is no rollback or re-fetch.

**Impact**

All invite tracking data for the affected guild is wiped. Until the next successful sync (which may not happen for up to 24 hours per the hourly `RefreshServers` rotation), the bot cannot attribute new member joins to any inviter. The `MemberJoin` handler will see an empty `oldInvites` list and fail to match any invite.

**How to reproduce**

1. Have the bot tracking invites in a guild.
2. Revoke the bot's "Manage Guild" permission (required for fetching invites).
3. Trigger a sync (via `/sync`, restart, or hourly refresh).
4. Observe that `DELETE FROM Invites` executes, `FetchInvites` logs a permissions error and returns `[]`, and the database now has zero invites for that guild.

**Suggested fix**

Fetch invites first. Only delete existing records from the database if the fetch returned a non-empty result (or if you explicitly want to proceed even when the result is empty). Alternatively, wrap the delete+insert in a transaction and roll back if the fetch fails.

---

# 2. UserInfo button crashes when the queried user is not in the Members table

**Severity:** High

**File**
`src/Buttons/UserInfo.ts`

**Line(s)**
`21-23`

**Summary**

The button handler assumes the database query always returns a row, but `.get()` returns `undefined` when no matching member exists. The subsequent property access on `undefined` throws an unhandled TypeError.

**Why this is a bug**

Execution path:
1. User clicks "User Info" button (from invite info, member join log, etc.).
2. The handler defers the reply and calls `GetUser(targetID)` — this succeeds if the user exists in the Users table.
3. Line 21: `Database.prepare("SELECT joined_at, left_at FROM Members WHERE guild_id = ? AND id = ?").get(interaction.guildId, targetID)` — if this user was never synced into the Members table (e.g., the guild has >1000 members and was skipped by `SyncMembersForGuild`, or the user joined while the bot was offline and the event was missed), this returns `undefined`.
4. The result is cast `as { joined_at: number | undefined; left_at: number | undefined }` — the cast is a TypeScript-only assertion and does not change the runtime value.
5. Line 23: `memberMetadata.joined_at` — accessing `.joined_at` on `undefined` throws `TypeError: Cannot read properties of undefined (reading 'joined_at')`.
6. The error is caught by the button handler in `Events/Handlers/Buttons.ts` and logged, but the interaction is never responded to, leaving the user with a stuck "thinking..." state that eventually times out.

**Impact**

Reliable crash when viewing user info for any user who is not in the Members table for the current guild. The interaction fails silently from the user's perspective. This is easily triggered in large servers (member count > 1000 skips the sync entirely per `SyncMembersForGuild`).

**How to reproduce**

1. Add the bot to a server with more than 1000 members (member sync is skipped).
2. A new member joins (member is saved to Members table).
3. Click "User Info" on any **existing** member (one who was already in the server before the bot was added and was never synced).
4. The handler crashes.

**Suggested fix**

Check if `memberMetadata` is undefined before accessing its properties:
```typescript
const memberMetadata = Database.prepare("...").get(...) as { joined_at: number; left_at: number } | undefined;
const joinedTimestamp = memberMetadata?.joined_at ? `<t:${memberMetadata.joined_at}:f>` : null;
```

---

# 3. InviteDelete event handler crashes when audit logs are empty or executor is null

**Severity:** High

**File**
`src/Events/InviteDelete.ts`

**Line(s)**
`38-44`

**Summary**

The handler unconditionally assumes that the audit log query returns at least one entry with a non-null executor. Both assumptions are false under normal Discord behavior, causing crashes.

**Why this is a bug**

Execution path:
1. An invite is deleted (by anyone, or by the system expiring it).
2. The `inviteDelete` event fires.
3. The handler reaches line 39: `const recentAuditLogs = await guild.fetchAuditLogs({ type: AuditLogEvent.InviteDelete, limit: 1 })`.
4. Discord audit logs are **eventually consistent** — there is a known delay (sometimes several seconds) between the event firing and the audit log entry appearing. The query can return an empty collection.
5. Line 40: `const latestLog = recentAuditLogs.entries.first()!` — if the collection is empty, `first()` returns `undefined`. The `!` is a TypeScript assertion only.
6. Line 42: `latestLog.executor?.id` — accessing `.executor` on `undefined` throws `TypeError: Cannot read properties of undefined (reading 'executor')`.

Additionally, even when `latestLog` exists, `latestLog.executor` can be `null` (e.g., when an invite expires naturally, or the executor account was deleted). In that case, line 42's optional chaining (`?.`) handles it for the comparison, but line 44 calls `ParseUser(latestLog.executor!)` which passes `null` to `ParseUser`, where `user.id` throws `TypeError: Cannot read properties of null (reading 'id')`.

This path is reached only when `client.invite_delete_ownership` does NOT have the code (i.e., the invite was NOT deleted via the bot's Purge or Delete button — meaning it was deleted directly in Discord or expired naturally).

**Impact**

Crash on every invite expiration or direct Discord deletion where audit log entries are delayed or missing. The error is caught by the event dispatcher and logged, but no delete-log message is sent to the configured log channel. Over time, this means many invite deletions go unlogged.

**How to reproduce**

1. Set up the bot with a log channel.
2. Create an invite with a short expiration (e.g., 1 hour).
3. Wait for it to expire naturally.
4. The `inviteDelete` event fires, the audit log may not have the entry yet, and the handler crashes.

**Suggested fix**

Guard against missing audit log entries and null executors:
```typescript
const latestLog = recentAuditLogs.entries.first();
if (!latestLog) {
    deletingUser = null;
} else if (latestLog.executor?.id === client.user!.id && latestLog.reason?.startsWith(INVITE_PURGE_REASON)) {
    return;
} else {
    deletingUser = latestLog.executor ? ParseUser(latestLog.executor) : null;
}
```

---

# 4. Member rejoin does not clear `left_at` or update `joined_at`

**Severity:** Medium

**File**
`src/CRUD/Users.ts`

**Line(s)**
`70-75`

**Summary**

`SaveMember` uses `ON CONFLICT DO NOTHING`, which means when a member who previously left rejoins, their old `left_at` timestamp is preserved and their `joined_at` is not updated, making them appear as having left even though they are currently in the server.

**Why this is a bug**

Execution path:
1. Member joins a server. `SaveMember` inserts `(id, guild_id, joined_at)` into the Members table.
2. Member leaves. `MemberLeave` handler sets `left_at = current_timestamp`.
3. Member rejoins. `guildMemberAdd` fires and calls `SaveMember(guild, member)`.
4. `SaveMember` executes: `INSERT INTO Members (id, guild_id, joined_at) VALUES (?, ?, ?) ON CONFLICT (id, guild_id) DO NOTHING`.
5. Since the row already exists, `DO NOTHING` is executed — **no columns are updated**.
6. The Members table still shows `left_at` as the old departure time, and `joined_at` as the original join time.

When a user views this member's info via the UserInfo button:
```typescript
if (joinedTimestamp) {
    embed.description += `\n\nJoined at: ${joinedTimestamp}`;
    if (leftTimestamp) {
        embed.description += `\nLeft at ${leftTimestamp}`;
    }
}
```
The embed shows "Left at: [old date]" even though the member is currently in the server. The join date is also stale.

**Impact**

Data integrity issue: every member who leaves and rejoins is permanently shown as having left. The `joined_at` timestamp is also wrong (shows first-ever join, not the current join). This affects the UserInfo display and any future feature that queries member status.

**How to reproduce**

1. Have a member in a tracked server.
2. The member leaves the server.
3. The member rejoins the server.
4. Use the UserInfo button on this member — it shows "Left at [old date]" and the original join date.

**Suggested fix**

Change `DO NOTHING` to an upsert that resets `left_at` and updates `joined_at`:
```sql
INSERT INTO Members (id, guild_id, joined_at)
VALUES (?, ?, ?)
ON CONFLICT (id, guild_id) DO UPDATE SET
    joined_at = excluded.joined_at,
    left_at = NULL
```

---

# 5. Setup command silently fails when the guild is not yet in the database

**Severity:** Medium

**File**
`src/Commands/Setup.ts`

**Line(s)**
`56`

**Summary**

The `/setup` command uses an `UPDATE` statement that silently affects zero rows if the guild has not been inserted into the Guilds table yet. The user sees "Setup successful!" but no log channel is actually configured.

**Why this is a bug**

Execution path:
1. Bot is added to a new server.
2. User runs `/setup #channel` immediately.
3. The handler passes permission checks and reaches line 56: `Database.prepare("UPDATE Guilds SET log_channel = ? WHERE id = ?").run(channel.id, interaction.guildId)`.
4. The Guilds table has no row for this guild. The UPDATE affects 0 rows. No error is thrown — SQLite silently does nothing.
5. Line 58: The bot sends "Setup successful!" to the user.
6. The log channel is never set. No logs will ever be sent.
7. The `MemberJoin` handler calls `GetGuild(guild.id)` which would save the guild (since it's in cache), but the `log_channel` column is null because the UPDATE never matched a row. So `savedGuild!.log_channel` is null, and the handler returns early — invite tracking is completely broken for this server.

When does the guild get inserted into the Guilds table?
- `SyncInvitesForGuild` inserts a row (runs at startup and hourly per guild).
- `MemberLeave` handler calls `SaveGuild`.
- `GetGuild` calls `SaveGuild` if the guild is in cache.

But none of these are guaranteed to have run before `/setup` is called on a freshly added bot. There is no `guildCreate` event handler.

**Impact**

Users who set up the bot immediately after adding it to a server see a success message but the bot never logs anything. The user has no indication that setup failed. They must re-run `/setup` after the guild has been synced (which may take up to 24 hours for the hourly rotation, or until a member leaves).

**How to reproduce**

1. Start the bot.
2. Add it to a new server.
3. Immediately run `/setup #logs`.
4. The bot says "Setup successful!" but no logs ever appear.

**Suggested fix**

Use an `INSERT ... ON CONFLICT DO UPDATE` (upsert) instead of a plain `UPDATE`, or call `SaveGuild(interaction.guild!)` before the UPDATE to ensure the row exists. Also check `result.changes` from the `.run()` call and report failure to the user.

---

# 6. Permanent invalid-ID caches cause indefinite lookup failures after transient errors

**Severity:** Medium

**File**
`src/CRUD/Guild.ts` (line 7), `src/CRUD/Users.ts` (line 7), `src/CRUD/Invites.ts` (line 7)

**Summary**

Three module-level `Set` objects (`INVALID_GUILD_IDS`, `INVALID_USER_IDS`, `INVALID_INVITE_CODES`) permanently cache IDs that failed to resolve. A single transient network error permanently blacklists the ID for the entire process lifetime, causing all future lookups to return null without retrying.

**Why this is a bug**

Execution path (Users as example, same pattern for Guilds and Invites):
1. `GetUser(id)` is called.
2. The user is not in the database.
3. The user is not in `client.users.cache`.
4. `client.users.fetch(id)` is called and fails (e.g., temporary network timeout, Discord API rate limit, or 500 error).
5. The `catch` block adds `id` to `INVALID_USER_IDS` and returns `null`.
6. From this point on, **every** call to `GetUser(id)` returns `null` immediately (line 44: `if (INVALID_USER_IDS.has(id)) return null`), without ever retrying the fetch.
7. This persists until the bot process is restarted.

This affects:
- **Users**: If a user's fetch fails once, their name/avatar is never shown in any log. The `InviteInfo` button shows "Owner: Unknown". The `UserInfo` button shows "User not found".
- **Guilds**: If a guild fetch fails once, `GetGuild` always returns null. The `MemberJoin` handler's `savedGuild!.log_channel` would then crash (accessing property of null) — but more critically, the `SendLog` function reads the log channel directly from DB, so logs silently stop.
- **Invites**: If `client.fetchInvite(code)` fails once, the code is permanently blacklisted. `GetInvite` always returns null. The `InviteDelete` event handler's "cache miss" path triggers, sending a "couldn't find information" log and re-syncing. But the code is also in `INVALID_INVITE_CODES`, so `SaveInvite` throws if the invite is recreated with the same code.

**Impact**

Transient Discord API failures cause permanent data lookup failures until process restart. In a long-running bot, this accumulates over time, degrading the quality of logs and tracking.

**How to reproduce**

1. Start the bot.
2. Temporarily disconnect the network (or simulate a Discord API 500 error).
3. Trigger any code path that calls `GetUser`, `GetGuild`, or `GetInvite` for an uncached ID.
4. Restore the network.
5. Observe that all subsequent lookups for that ID still return null without retrying.

**Suggested fix**

Replace the permanent `Set` with a TTL-based cache (e.g., evict entries after 5 minutes), or remove the caching entirely and rely on the database as the source of truth. At minimum, add a mechanism to clear these sets periodically.

---

## Possible Concerns (Not Confirmed)

1. **`MemberJoin` early return when `log_channel` is null prevents invite DB updates**: When no log channel is configured, the handler returns after `SaveMember` without updating the DB invite `uses` counts. This means the DB state drifts from reality until the next sync. However, this appears to be a deliberate design choice (no point tracking if there's nowhere to log), and the sync process eventually reconciles the state.

2. **`MemberJoin` non-null assertion on `GetGuild` result**: `savedGuild!.log_channel` assumes `GetGuild` never returns null. In practice, since the bot is receiving a guild event, the guild is always in `client.guilds.cache`, so `GetGuild` succeeds. However, the `!` is technically unsafe. Not confirmed as a practical bug because the guild must be in cache to receive the event.

3. **`InviteCreate` handler crashes on vanity invites**: `SaveInvite` → `ParseInvite` throws for vanity URLs. Vanity invites likely don't trigger `inviteCreate` events in Discord's API, so this path may never execute. Not confirmed as reachable.

4. **`InviteInfo` command regex fallback can match wrong substring**: The fallback regex `/[\w-]{3,}/g.exec(input)` could match a word other than the invite code if the input contains spaces. In practice, users pass a single code or URL, and the primary `Invite.InvitesPattern` regex handles URLs correctly. Edge case only.

5. **No `guildCreate` or `guildDelete` event handlers**: The bot doesn't explicitly handle being added to or removed from guilds. New guilds are added to the DB through `SyncInvitesForGuild` (which runs on startup and hourly). Removed guilds are never cleaned up. This is a design limitation rather than a bug — the bot continues to function, just with stale data for removed guilds.

6. **`SyncInvitesForGuild` vs `MemberJoin` race condition**: During the window between `DELETE FROM Invites` and the subsequent inserts in `SyncInvitesForGuild`, a concurrent `MemberJoin` event would see an empty invite list and fail to attribute the join. This is a timing-dependent race condition that is unlikely to cause significant issues in practice since sync is fast and member joins are relatively infrequent.

---

## False Positives Considered

1. **`PurgeInvites` uses `Promise.all` which rejects on first failure**: While `Promise.all` does short-circuit on the first rejection, the remaining deletion promises are already in flight. The `inviteDelete` events that fire for successful deletions will properly clean up the database via `DiscardInvite`. The error message "Something went wrong" is imprecise but not incorrect — something did go wrong. Not a bug.

2. **`close` button handler calling `deleteReply` after `deferUpdate`**: For component interactions, `deferUpdate()` acknowledges the interaction (type 6 response), and `deleteReply()` deletes the original message the component is on. This is the intended behavior — the ephemeral confirmation dialog is dismissed. Not a bug.

3. **`FIFOCache` with max 10 entries for `invite_delete_ownership`**: If more than 10 invites are deleted simultaneously, older entries are evicted. This is an acknowledged trade-off (documented in the code comment "don't trust audit log"). The fallback is to use audit logs, which works in most cases. Not a bug.

4. **`Sync` command missing closing backtick in cooldown message**: The message `Please wait \`X seconds\`` is missing a closing backtick, but this is a cosmetic typo, not a bug.

5. **`Setup` command requires `ManageGuild` on the channel, not just the server**: The permission check requires the bot to have `ManageGuild` permission in the specific channel. This is a deliberate security measure to ensure the bot can only be configured in channels where it has the appropriate permissions. Design choice, not a bug.

6. **`TaskScheduler` dummy task with `setTimeout(2147483647)`**: The ~24.8 day timeout is used as a sentinel to work around `setTimeout`'s 32-bit limit. The `.unref()` call ensures it doesn't keep the process alive. This is a correct implementation. Not a bug.

7. **`DB_SETUP.sql` deletes all invites on restart**: The `DELETE FROM Invites` at the end of the SQL file intentionally clears the invites table on every startup, since invites are re-synced from Discord's API. This is documented behavior, not a bug.

8. **`InviteDelete` button authorization (cross-guild invite deletion)**: Initially suspected as an authorization bypass. However, Discord does not allow clients to forge interaction custom_ids. The delete button is only rendered (enabled) when `invite.guild_id === interaction.guildId`, and the custom_id is server-side. Users cannot craft arbitrary button interactions. Not exploitable.

9. **`RegisterCommands` deduplication uses `commandNames.includes()`**: While `includes()` on an array is O(n), the number of commands is small (< 20), so performance is not a concern. Not a bug.

10. **`SaveInvite` throwing for blacklisted codes in `InviteCreate` handler**: If an invite code is in `INVALID_INVITE_CODES` and a new invite with that code is created, `SaveInvite` throws. However, Discord invite codes are randomly generated with very high entropy, making code reuse astronomically unlikely. The throw is caught by the event dispatcher. Not a practical bug.

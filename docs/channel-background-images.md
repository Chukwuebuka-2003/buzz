# Channel Background Images Implementation Plan

**Issue:** [#4132 — Channel Background Images (Like iOS Messages)](https://github.com/block/buzz/issues/4132)

**Goal:** Let a channel have a background image on its chat surface — set by a
member via channel settings, or by an agent via a mention — visible to every
device in the community (relay-shared channel state, not a local preference).

**Architecture:** The wallpaper is a new optional channel-metadata field
`wallpaper_url` that rides the existing channel metadata pipeline end-to-end:
kind:9002 edit event (`["wallpaper", url]` tag) → relay `handle_edit_metadata`
arm → `channels.wallpaper_url` DB column → kind:39000 discovery tag →
`ChannelDetailInfo.wallpaper_url` → TS `Channel.wallpaperUrl` → rendered as a
background layer on the timeline scroll container. The agent path uses the same
SDK builder (`build_set_wallpaper`) that agents already use for topics, so a
mention → tool call → metadata event works with no new agent machinery.

**Tech Stack:** Rust (buzz-relay, buzz-db, buzz-sdk, desktop Tauri), React 19 +
Tailwind (desktop), SQL (Postgres migration).

---

## Decision: relay-shared metadata (Option A)

The wallpaper must be settable by an agent via mention, which requires it to be
relay-published channel state — exactly like `topic`/`purpose` today. A
local-only preference (Option B) would make every device look different and
wouldn't satisfy the issue's agent-set requirement.

**Storage shape:** a single optional URL string per channel. Empty string
clears the wallpaper. No per-device overrides, no image upload pipeline in v1 —
members paste a URL (or an agent sets one), exactly like `topic`.

**Scope guards (YAGNI):**
- No upload-to-relay media pipeline in v1. URL only.
- No per-user per-channel override. One shared wallpaper per channel.
- No animated/parallax effects. Static `bg-cover` layer.
- Applies to all channel types (stream, forum, dm) uniformly.

---

## Data flow (the model the wallpaper follows)

Today's topic flow, which this plan mirrors field-for-field:

1. **Desktop** `setChannelTopic()` → Tauri `set_channel_topic`
   (`desktop/src-tauri/src/commands/channels.rs:733`) →
   `events::build_set_topic` (`desktop/src-tauri/src/events.rs:225`) → signed
   kind:9002 event with `["h", <uuid>]` + `["topic", <val>]` tags → submitted
   to relay.
2. **Relay** `handle_edit_metadata` (`crates/buzz-relay/src/handlers/side_effects.rs:1431`)
   matches the `topic` tag → `db.set_topic(...)` → emits a `topic_changed`
   system message.
3. **DB** `set_topic` (`crates/buzz-db/src/channel.rs:1263`) →
   `UPDATE channels SET topic = $1, topic_set_by = $2, topic_set_at = NOW() ...`.
4. **Relay discovery** `emit_group_discovery_events`
   (`crates/buzz-relay/src/handlers/side_effects.rs:1045`) includes the topic
   as a `["topic", ...]` tag on the kind:39000 discovery event (line 1087-1091).
5. **Desktop parse** `channel_detail_from_event`
   (`desktop/src-tauri/src/nostr_convert.rs:182`) reads the tag into
   `ChannelDetailInfo` (`desktop/src-tauri/src/models.rs:137`), which the TS
   layer maps via `fromRawChannelDetail`
   (`desktop/src/shared/api/tauriChannels.ts:82`) into `Channel.wallpaperUrl`.
6. **Render** `TimelineMessageList` (`desktop/src/features/messages/ui/TimelineMessageList.tsx:567`)
   scroll container paints the wallpaper as a background layer.

The wallpaper adds a parallel column `wallpaper_url` (TEXT, nullable) to the
same `channels` table (schema root: `migrations/0001_initial_schema.sql:88-93`)
and a `["wallpaper", url]` tag everywhere topic appears.

---

## Implementation tasks

### Task 1: DB migration — `channels.wallpaper_url`

**Objective:** Add the nullable wallpaper column.

**Files:**
- Create: `migrations/0027_channel_wallpaper.sql`

**Step 1: Write the migration**

```sql
-- Channel background image (issue #4132). NULL/empty = no wallpaper.
ALTER TABLE channels
    ADD COLUMN wallpaper_url TEXT;
```

**Step 2: Verify**

```bash
cd crates/buzz-db && cargo check
```

Migration files are applied automatically on relay startup (per `migrations/`
README); the sqlx query macros in Task 3 will fail at compile time if the
column name mismatches, which is the real gate.

**Step 3: Commit**

```bash
git add migrations/0027_channel_wallpaper.sql
git commit -s -m "feat(db): add channels.wallpaper_url for channel background images"
```

---

### Task 2: SDK builder — `build_set_wallpaper`

**Objective:** Give agents and the CLI the same kind:9002 builder they use for
topics.

**Files:**
- Modify: `crates/buzz-sdk/src/builders.rs` (next to `build_set_topic` at :652)

**Step 1: Add the builder**

```rust
/// Build a NIP-29 edit-metadata event for wallpaper (kind 9002).
pub fn build_set_wallpaper(channel_id: Uuid, url: &str) -> Result<EventBuilder, SdkError> {
    let tags = vec![
        tag(&["h", &channel_id.to_string()])?,
        tag(&["wallpaper", url])?,
    ];
    Ok(EventBuilder::new(Kind::Custom(9002), "").tags(tags))
}
```

**Step 2: Unit test** — mirror the existing `build_set_topic` test in
`crates/buzz-sdk/src/builders.rs` (search `build_set_topic` test block, ~line 2473).

**Step 3: Verify**

```bash
cargo test -p buzz-sdk
```

**Step 4: Commit**

```bash
git commit -s -m "feat(sdk): add build_set_wallpaper kind:9002 builder"
```

---

### Task 3: DB write path — `set_wallpaper`

**Objective:** Persist the wallpaper like topic/purpose.

**Files:**
- Modify: `crates/buzz-db/src/channel.rs` (next to `set_purpose` at :1287)

**Step 1: Add the function**

```rust
/// Sets the background image URL for a channel.
pub async fn set_wallpaper(
    pool: &PgPool,
    community_id: CommunityId,
    channel_id: Uuid,
    url: &str,
    set_by: &[u8],
) -> Result<()> {
    let result = sqlx::query(
        "UPDATE channels SET wallpaper_url = $1, wallpaper_set_by = $2, wallpaper_set_at = NOW() \
         WHERE community_id = $3 AND id = $4 AND deleted_at IS NULL",
    )
    .bind(url)
    .bind(set_by)
    .bind(community_id.as_uuid())
    .bind(channel_id)
    .execute(pool)
    .await?;
    if result.rows_affected() == 0 {
        return Err(DbError::ChannelNotFound(channel_id));
    }
    Ok(())
}
```

Note: this requires `wallpaper_set_by` / `wallpaper_set_at` columns — add them
to the Task 1 migration (or drop the audit columns and keep the query minimal;
decision below in Open Questions).

**Step 2: Verify** — `cargo check -p buzz-db`; add a unit test only if the
crate has an existing in-memory DB test harness (check `crates/buzz-db/tests/`).
If none exists, the compile-time sqlx check plus relay e2e in Task 7 is the gate.

**Step 3: Commit**

```bash
git commit -s -m "feat(db): add set_wallpaper channel update"
```

---

### Task 4: Relay — `handle_edit_metadata` wallpaper arm + discovery tag

**Objective:** Accept `["wallpaper", url]` on kind:9002 and publish it on
kind:39000.

**Files:**
- Modify: `crates/buzz-relay/src/handlers/side_effects.rs`
  - `handle_edit_metadata` match (after the `purpose` arm, ~:1500)
  - `emit_group_discovery_events` tag block (after `purpose`, ~:1096)

**Step 1: Add the match arm**

```rust
"wallpaper" => {
    state
        .db
        .set_wallpaper(tenant.community(), channel_id, val, &actor_bytes)
        .await?;
    emit_system_message(
        tenant,
        state,
        channel_id,
        serde_json::json!({
            "type": "wallpaper_changed", "actor": actor_hex, "wallpaper": val
        }),
    )
    .await?;
}
```

**Step 2: Add the discovery tag**

```rust
if let Some(ref url) = channel.wallpaper_url {
    if !url.is_empty() {
        tags.push(Tag::parse(["wallpaper", url])?);
    }
}
```

**Step 3: Verify** — `cargo check -p buzz-relay`, and extend the existing
`handle_edit_metadata` test module in the same file (the file already has a
test harness; search `mod tests` in `side_effects.rs`) with a
`wallpaper_changed`-system-message assertion.

**Step 4: Commit**

```bash
git commit -s -m "feat(relay): accept and publish channel wallpaper metadata"
```

---

### Task 5: Desktop backend — command, event builder, parse, model

**Objective:** Wire the desktop Tauri surface end-to-end.

**Files:**
- Modify: `desktop/src-tauri/src/events.rs` (after `build_set_purpose` at :234)
- Modify: `desktop/src-tauri/src/commands/channels.rs` (after `set_channel_purpose` at :745)
- Modify: `desktop/src-tauri/src/nostr_convert.rs` (`channel_detail_from_event`, after `purpose` read at :190)
- Modify: `desktop/src-tauri/src/models.rs` (`ChannelDetailInfo`, after `purpose_set_at` at :149)
- Modify: `desktop/src-tauri/src/lib.rs` (register the new command, after `set_channel_purpose` at :750)

**Step 1: Event builder** — mirror `build_set_purpose`:

```rust
/// Kind 9002 — set wallpaper.
pub fn build_set_wallpaper(channel_id: Uuid, url: &str) -> Result<EventBuilder, String> {
    let tags = vec![
        tag(vec!["h", &channel_id.to_string()])?,
        tag(vec!["wallpaper", url])?,
    ];
    Ok(EventBuilder::new(Kind::Custom(9002), "").tags(tags))
}
```

**Step 2: Tauri command** — mirror `set_channel_purpose`:

```rust
#[tauri::command]
pub async fn set_channel_wallpaper(
    channel_id: String,
    url: String,
    state: State<'_, AppState>,
) -> Result<(), String> {
    let uuid = parse_channel_uuid(&channel_id)?;
    let builder = events::build_set_wallpaper(uuid, &url)?;
    submit_event(builder, &state).await?;
    Ok(())
}
```

**Step 3: Parse + model** — read the tag in `channel_detail_from_event` and add
`wallpaper_url: Option<String>` to `ChannelDetailInfo`. Add the existing
`nostr_convert.rs` unit test pattern (`channel_detail_from_event` tests at
~:720) for the wallpaper tag.

**Step 4: Register** in `lib.rs` invoke_handler list.

**Step 5: Verify**

```bash
cd desktop/src-tauri && cargo check
```

**Step 6: Commit**

```bash
git commit -s -m "feat(desktop): add set_channel_wallpaper command and channel wallpaper field"
```

---

### Task 6: Desktop frontend — TS types, API wrapper, settings UI, render

**Objective:** Surface the wallpaper in the UI and paint it on the timeline.

**Files:**
- Modify: `desktop/src/shared/api/types.ts` — `Channel` (after `purpose` at :12) and `ChannelDetail` (after `purposeSetAt` at :31): `wallpaperUrl: string | null`
- Modify: `desktop/src/shared/api/tauriChannels.ts` — `RawChannelDetail` (+`wallpaper_url`), `fromRawChannelDetail` map it
- Modify: `desktop/src/shared/api/tauriChannels.ts` — add:

```ts
export async function setChannelWallpaper(
  channelId: string,
  url: string,
): Promise<void> {
  await invokeTauri("set_channel_wallpaper", { channelId, url });
}
```

- Modify: `desktop/src/features/channels/ui/ChannelManagementSheet.tsx` — add a
  "Background image" field next to Topic/Purpose (input: URL; empty clears)
  wired to `setChannelWallpaper` + query invalidation
- Modify: `desktop/src/features/messages/ui/TimelineMessageList.tsx` — paint
  the wallpaper behind the VList scroll container (line 567): a
  `style={{ backgroundImage: ... }}` on the container or an absolutely
  positioned `bg-cover bg-center` layer, gated on the channel's
  `wallpaperUrl`. Thread the value from `ChannelPane`
  (`desktop/src/features/channels/ui/ChannelPane.tsx`) through the existing
  `MessageTimeline` props.
- Modify: `desktop/src/features/channels/hooks.ts` — add the mutation (mirror
  the `setChannelTopic` mutation at :392)

**Step 1:** TS types + wrapper + hook mutation.
**Step 2:** Render layer (CSS-only; verify with a hardcoded URL in dev).
**Step 3:** Management sheet field.
**Step 4: Verify**

```bash
cd desktop && pnpm typecheck && node --import ./test-loader.mjs --experimental-strip-types --test src/features/channels/hooks.test.mjs
```

**Step 5: Commit**

```bash
git commit -s -m "feat(desktop): channel background image UI and timeline rendering"
```

---

### Task 7: Agent + CLI path

**Objective:** Let an agent set the wallpaper by mention, and the CLI by
subcommand.

**Files:**
- Modify: `crates/buzz-cli/src/commands/channels.rs` — add a
  `channels set-wallpaper <channel-id> <url>` subcommand mirroring the existing
  topic subcommand (~:871, which calls `buzz_sdk::build_set_topic`)

**Step 1: CLI subcommand** — reuse `build_set_wallpaper` from Task 2, follow the
existing topic subcommand's signing/submit flow exactly.

**Step 2: Verify**

```bash
cargo check -p buzz-cli
```

**Step 3: Commit**

```bash
git commit -s -m "feat(cli): add channels set-wallpaper subcommand"
```

---

### Task 8: Integration verification + full gate

**Objective:** Prove the round trip works end-to-end.

**Files:** none (verification only).

**Step 1:** `cargo test -p buzz-sdk -p buzz-db -p buzz-relay` — all pass.
**Step 2:** `just ci` (fmt + clippy + desktop lint + unit tests + builds).
**Step 3:** Manual e2e against a local relay: create channel → set wallpaper
via settings → verify timeline shows it; set via `buzz channels set-wallpaper`
→ verify a second device view picks it up; clear via empty URL → verify it
disappears.
**Step 4: Commit** any fixups, squash if needed.

```bash
git commit -s -m "chore: final verification pass"
```

---

## Risks, tradeoffs, open questions

- **Audit columns:** `topic`/`purpose` track `_set_by`/`_set_at`; wallpaper can
  either mirror that (2 extra columns) or skip it (YAGNI — nobody's asked for
  wallpaper audit). Decision: **skip the audit columns in v1** unless the relay
  system-message is deemed insufficient. If skipped, the `set_wallpaper` SQL
  drops those two assignments. **Recommended:** skip — the
  `wallpaper_changed` system message already records actor + timestamp.
- **URL validation:** v1 stores whatever string is submitted (like topic). No
  URL-parsing validation; clients render with `bg-cover` so a broken URL is
  visibly broken, not dangerous. Optional hardening: reject non-`http(s)://`
  prefixes in the Tauri command.
- **Relay e2e coverage:** `buzz-relay` integration tests need Postgres/Redis
  (`just test`); the sandbox can't run them. CI covers this on the PR.
- **Migration numbering:** `0026_replica_heartbeat.sql` is the current tail;
  this plan assumes `0027`. Renumber to the actual tail at implementation time.
- **DM channels:** wallpaper applies to DMs too. No special-casing needed, but
  confirm with maintainers that per-DM wallpaper is desired (privacy-wise it's
  per-channel metadata shared with all members, consistent with topic).
- **Kind consistency:** the desktop `events.rs` and `buzz-sdk` both build
  kind:9002. Both must emit the identical `["wallpaper", url]` tag shape —
  the relay arm is the single point of truth for the tag name.

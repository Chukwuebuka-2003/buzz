# Issue #6554 — Newly created workflows cannot be deleted

## Symptom

Workflows create and run fine, but deletion does not stick. The user confirms
the delete (UI button or `buzz workflows delete <id>`), the relay logs an
"accepted" deletion, yet on refresh / client restart the workflow reappears,
still triggers, and remains in the dashboard list.

## Root cause

The workflow list shown in the desktop app is sourced from the relay's
**`events` table**, not the `workflows` table.

- `get_channel_workflows` (`desktop/src-tauri/src/commands/workflows.rs`,
  ~line 96) issues a Nostr REQ for **kind:30620** events filtered by `#h`
  (channel) and renders each as a `WorkflowWire`. The `workflows` table row is
  only a trigger/execution cache consumed by the scheduler and event-trigger
  path; it is NOT what the list reads.
- Deletion flows through a kind:5 NIP-09 `a`-tag event
  `30620:<owner>:<uuid>`, handled by `handle_a_tag_deletion` in
  `crates/buzz-relay/src/handlers/side_effects.rs`.
- The `KIND_WORKFLOW_DEF` branch (side_effects.rs:2162-2208) **only** calls
  `delete_workflow_for_owner`, which removes the `workflows` *table* row and
  invalidates the per-channel trigger cache.
- It **never soft-deletes the kind:30620 event** from the `events` table. The
  generic NIP-33 `soft_delete_by_coordinate` path (side_effects.rs:2217+) is
  reached only for other parameterized-replaceable kinds, because
  `KIND_WORKFLOW_DEF` is matched first (line 2162) and returns.

Result: the `workflows` row is gone (triggers stop), but the kind:30620 event
stays in `events` with `deleted_at IS NULL`. The desktop's `#h` list query
keeps returning it, so the workflow "resurrects" on every refresh. The kind:5
deletion event is the "accepted deletion status event" in the logs; the
"tracking database fails to clear the event record" is precisely the
un-soft-deleted kind:30620 row. The reporter's `enabled: false` workaround
only hides firing — the row still lists because the list reads the event.

The CLI (`buzz workflows delete <id>`) builds the same kind:5 `a`-tag, so it
hits the identical gap.

## Fix

In the `KIND_WORKFLOW_DEF` branch of `handle_a_tag_deletion`, after deleting the
`workflows` table row, also call `soft_delete_by_coordinate` on the kind:30620
event so the UI list stops returning it.

Details / correctness:

- `soft_delete_by_coordinate` keys on `(community_id, kind, pubkey, d_tag,
  created_at <= deletion_time)` in `crates/buzz-db/src/event.rs:838`.
- Use the **coordinate's** pubkey (`parts[1]`, already available as
  `pubkey_hex` in `handle_a_tag_deletion`), decoded via `hex::decode`, NOT
  `actor_bytes`. When an agent deletes on behalf of its owner, `actor_bytes`
  is the agent but the event's `pubkey` is the human owner; the event row must
  be matched by the owner's pubkey. (`validate_standard_deletion_event`
  already permits the agent via `is_agent_owner`, so passing `actor_bytes`
  here would also break the table delete in that case — using the coordinate
  pubkey fixes both.)
- Use the event's `d` tag as `d_tag`:
  - UUID branch: `d_tag` is already the workflow UUID, which equals the
    kind:30620 event's `d` tag (set by `build_workflow_definition`).
  - Name branch: the event's `d` tag is the UUID, not the name, so pass the
    resolved `wf.id` (already looked up) as the `d_tag`.
- Pass `event.created_at.as_secs() as i64` as the deletion timestamp so the
  at-or-before scoping matches the generic path (side_effects.rs:2238).
- Keep the existing `delete_workflow_for_owner` table delete + trigger-cache
  invalidation; add the `soft_delete_by_coordinate` call alongside it.

## Files to change

- `crates/buzz-relay/src/handlers/side_effects.rs` — `handle_a_tag_deletion`,
  `KIND_WORKFLOW_DEF` branch (UUID + name sub-branches).

## Verification

1. `cargo test -p buzz-relay --lib handle_a_tag` (add a unit/integration test
   asserting that after a kind:5 `a`-tag delete, a REQ for kind:30620 `#h`
   no longer returns the event AND `delete_workflow_for_owner` reports the
   table row gone). Reuse the include-pattern tests there
   (`#[path = "..." mod tests]`).
2. `just ci` (fmt + clippy + desktop lint + unit tests + builds) passes.
3. Manual: create a workflow in a channel, confirm it lists, delete it via UI
   and via `buzz workflows delete <id>`, refresh — it must stay gone from the
   list and stop triggering.

## Note

This is a backend (relay) fix; no desktop frontend change is required because
the UI already derives the list from kind:30620 events. Closing the gap at the
relay makes the existing UI delete work end-to-end.

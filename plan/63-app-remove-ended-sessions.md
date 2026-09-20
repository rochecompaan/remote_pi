# 63 — App: remove ended sessions immediately

> **Status**: DONE (2026-09-20). Verified on-device (debug APK, Galaxy
> S21 Ultra, self-hosted relay): killing a Pi session removes its room
> from Home within ~1–2 s, and a fresh pairing renders an accurate
> snapshot with no restored ghosts.

## Context

Investigation (2026-09-20, fork `rochecompaan/remote_pi`) found the app's
"31 online / 5 offline" ghost rows are durable cache entries, not relay
state:

- `RoomEnded` removed only the live marker and explicitly kept the cached
  room (`app/lib/data/transport/connection_manager.dart`).
- `RoomsSnapshot` seeded the merge from all cached rooms and never pruned.
- Cached rooms are persisted to disk and restored on every boot, so ended
  sessions survived cold starts.
- Home renders one row per cached `(peer, room)` and labels non-live rooms
  offline.

Relay contract (verified in `relay/src/`):

- `registry.rs` `unregister()` emits `room_ended` per room when its last
  connection dies, and `peer_offline` when no rooms remain.
- The `rooms_check` reply is an authoritative per-peer `rooms` snapshot
  (empty when the peer has no live rooms). Dedup suppression is
  per-connection, so the first reply on every fresh (re)connect always
  arrives. The app already sends `rooms_check` for all subscribed peers on
  every connect.

**User decision (2026-09-20):** ended sessions are removed immediately — no
history section, no TTL.

| # | Decision |
|---|---|
| **A** | `RoomEnded` prunes the room from memory **and** disk; no grey historical tile |
| **B** | `RoomsSnapshot` is authoritative per peer: the cached list becomes exactly the snapshot; cached rooms absent from it ended while the app was away and are pruned |
| **C** | Boot restore stays (last-known rooms render grey on cold start); the first authoritative snapshot prunes the ghosts, typically seconds after connect |
| **D** | `_markActiveRoomOffline` (3 missed protocol pings) unchanged — it is a local degraded-state signal, not an ended-room signal |
| **E** | Merge semantics preserved for rooms still live: `localName`, previously-known `model`/`thinking` survive announces and snapshots; `working` handling unchanged |

**Accepted trade-off:** a long-press rename is forgotten when its room ends
(the persisted record is pruned). A later session reusing the same roomId
starts with the wire name again. This follows directly from "ended sessions
disappear completely".

## Objective

Ended Pi sessions disappear from the app's Home list as soon as the relay
reports them, instead of lingering as grey "offline" tiles forever.

## Non-goals

- No relay (`relay/`) or pi-extension changes. No UI widget changes.
- Lifecycle-aware resume recovery (validate/replace the WebSocket on app
  resume) — next plan.
- Runtime-configurable connection timings.
- Upstream issue #194 (saved-session browsing) — in tension with this plan;
  the fork owner chose immediate removal regardless.

## Structure (what changed)

```
app/lib/data/transport/connection_manager.dart   # _onControl: RoomEnded + RoomsSnapshot cases; comment contract
app/test/transport/connection_manager_test.dart  # 1 test rewritten, 6 added
app/test/ui/home/home_viewmodel_test.dart        # presence-filter test rewritten, 1 added
```

Interfaces consumed (existing, unchanged): `PairingStorage.loadRooms/saveRooms`,
`PersistedRoom`, `RoomInfo`, `RoomAnnounced/RoomEnded/RoomsSnapshot`.
Interfaces produced: none — the public API of `ConnectionManager` is
unchanged (consumers `chat_viewmodel.dart` and `actions_repository.dart`
already tolerate a missing room).

## Steps

### 1. `RoomEnded` prunes the room (memory + disk) — DONE

The `RoomEnded` case in `_onControl` removes `(peer, roomId)` from
`_roomsByPeer` and `_liveRoomIds` and persists the pruned list via
`_persistRoomsForPeer`. A `RoomEnded` for an unknown room is a no-op; a
room re-announced after ending reappears as live.

Acceptance: killing a Pi session removes its tile without manual deletion;
the persisted list no longer contains the ended room.

### 2. `RoomsSnapshot` becomes authoritative — DONE

The `RoomsSnapshot` case rebuilds the peer's cached list as exactly the
snapshot's rooms, dropping absent ones from memory and disk. Re-emitted
identical snapshots are skipped (no listener churn). An empty snapshot
drops the peer key entirely; restored ghosts are pruned by the first
snapshot after boot. `localName` and previously-known `model`/`thinking`
are preserved for rooms present in both.

Acceptance: a cached room absent from the snapshot is pruned from memory
and disk; the local rename survives for rooms still live.

### 3. Comment contract + Home viewmodel tests — DONE

Field comments on `_roomsByPeer`/`_liveRoomIds` and `_restoreCachedRooms`
state the plan-63 contract (cache = live-confirmed + not-yet-contradicted
restored rooms; ended rooms are pruned, not greyed). The presence-filter
test now sources its offline row from a restored-but-unconfirmed room, and
a new test asserts an ended room disappears from counts and items
immediately.

Acceptance: home viewmodel tests green against the new contract.

## Definition of Done

- [x] `RoomEnded` removes the room from memory and disk; the Home tile disappears without manual deletion
- [x] `RoomsSnapshot` prunes any cached room the relay no longer reports, including restored rooms after a cold start
- [x] Live-room merge semantics preserved (rename / model / thinking / `working`)
- [x] `flutter analyze` zero issues; `flutter test` green (549 tests)
- [x] No changes to `relay/`, `pi-extension/`, or UI widgets
- [x] On-device: room removal propagates to a paired phone within ~1–2 s; fresh pairing shows no restored ghosts

Known pre-existing flakes, unrelated to this plan: a retry-timing test that
fails intermittently under full-suite load but passes in isolation, and a
`speech_service` test that fails on Linux hosts — both reproduce on
pristine `main`.

## Next plans

- **64** — App: lifecycle-aware resume recovery (validate/replace the
  WebSocket on `AppLifecycleState.resumed`, reconnect, replay
  subscriptions, refresh presence/rooms) — fixes the screen-lock
  disconnect.
- **65** — Runtime configurability of connection timings.

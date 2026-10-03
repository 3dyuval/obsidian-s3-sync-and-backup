# Baseline & Multi-Machine Initialization Spec

## Problem

The three-way reconciler decides each file's action from two classifications
plus one fact: **does a journal baseline exist for this path?**

- `L0` local absent, `R0` remote absent
- `L=`/`R=` present and matches baseline
- `LΔ`/`RΔ` present and differs from baseline
- `L+`/`R+` present but **no baseline**

The baseline is the "last known good" shared state per path. A deletion is
inferred, never declared: `L0 + R=` (local gone, remote still matches baseline)
is read as "this machine deleted the file" and produces `delete-remote`.

That inference is only sound when the baseline is **trustworthy**. On a new or
cloned machine the journal is empty or stale. An empty journal is currently
indistinguishable from "I deleted everything," so:

```
local empty  +  journal has baselines  +  remote full
  => every path classifies L0 + R=
  => delete-remote for the entire vault
```

This is the observed failure: a clone with a populated-but-wrong journal
scheduled deletion of the whole remote vault. Only the absence of the
`DeleteObject` permission on the key limited the loss.

## Root cause

`L0 + baseline-present` is overloaded. It means both:

1. "I had this file and deleted it" — propagate deletion. **Correct** only when
   the baseline reflects reality.
2. "My journal is stale/foreign and claims a baseline I never actually had" —
   must NOT delete.

The plugin cannot tell these apart because it has no concept of **whether this
machine has ever been explicitly initialized against this remote.** An empty or
inherited journal is treated as authoritative rather than as "unknown."

## Principle

Baseline is explicit. For multi-machine sync, **initialization must be explicit
too.** A machine's standing against a remote is one of:

- **uninitialized** — no verified relationship to this remote yet. The journal,
  if any, is not trusted. `delete-remote` is forbidden.
- **initialized** — the user has explicitly declared this machine's relationship
  (cloned from remote, or adopted local as authoritative). Baselines are now
  trustworthy; the full decision table (including deletion) is permitted.

The transition from uninitialized to initialized is a deliberate user action,
never inferred from journal contents.

## Existing machinery (verified in source)

The engine already carries safety scaffolding that this spec builds on rather
than replaces:

- **Phase 0 — destination fingerprint** (`SyncEngine.reconcileDestinationFingerprint`):
  stores a fingerprint of bucket/endpoint/region/prefix/provider in journal
  metadata. If it differs from current settings, the journal is wiped so stale
  baselines from another destination cannot drive a destructive plan. This did
  NOT fire on the clone incident because the destination matched — the stale
  baselines were for the *right* remote, just wrong about local state.

- **`lastSuccessfulSyncAt`** journal-metadata key: set only after a fully
  successful sync. This is already a de-facto "has this machine ever synced this
  destination" marker and should back R1 instead of a new `initialized` key.

- **Phase 1.5 — destructive-plan guard** (`SyncEngine.checkDestructivePlan`):
  refuses a plan when it contains deletions AND `lastSuccessfulSyncAt` is unset.

### The actual bug (clone incident root cause)

`checkDestructivePlan` counts **only** `delete-local`:

```
item.action === 'delete-local'      // delete-remote is NOT counted
```

The clone's plan was full of `delete-remote`, so the guard saw zero destructive
actions and let the whole plan through. The one backstop meant to prevent
exactly this only watched one of the two deletion directions. Extending it to
`delete-remote` is the single highest-value fix.

## Requirements

### R0 — Guard both deletion directions (highest priority)

`checkDestructivePlan` MUST count `delete-remote` as destructive, not only
`delete-local`. A plan containing either kind of deletion against a destination
with no `lastSuccessfulSyncAt` MUST be blocked. This closes the exact gap that
destroyed the clone and is independent of the larger init-flow work below.

### R1 — Init gate (marker)

- "Initialized" = `lastSuccessfulSyncAt` is set in journal metadata (reuse the
  existing key; no new `initialized` key needed). Absent ⇒ uninitialized.
- The marker lives in the journal (IndexedDB), NOT in `data.json`. `data.json`
  is dotfile-managed and would be copied to a new machine, wrongly signalling
  "initialized." The journal is per-machine and per-vault and is not copied.
- The explicit clone/adopt actions (R3/R4) set this marker on success, same as
  an ordinary successful sync does today (`SyncEngine` Phase 3).

### R2 — Uninitialized behavior

- In uninitialized mode the engine MUST NOT execute any `delete-remote` or
  `delete-local`. Only non-destructive actions are allowed to be *considered*:
  `adopt`, `download`, `upload`.
- On a sync attempt while uninitialized, the engine blocks and surfaces an
  explicit prompt asking how to initialize (see R3). No reconciliation that can
  delete runs until the user chooses.

### R3 — Explicit initialization choices

The prompt offers mutually exclusive intents:

- **Clone from remote** — remote is authoritative. Local is replaced by a full
  download. Baselines are seeded from the downloaded objects. Sets the marker.
  (Backed by R4.)
- **Adopt local** — local is authoritative. Local files are the starting truth;
  baselines are seeded by adopting/uploading as needed. Sets the marker.
- **Cancel** — do nothing, remain uninitialized.

### R4 — Destroy-local-and-clone action

A dedicated, explicit action distinct from ordinary sync:

- Removes local vault content and pulls the full remote state down
  (download-only; the `delete-remote` code path is never reachable from it).
- Seeds the journal baselines from the freshly downloaded files so a subsequent
  normal sync is a no-op (`L= / R=`).
- Sets the init marker on success.
- Is irreversible for local content; requires explicit confirmation before
  running.

### R5 — Safety invariant

- Deletion (`delete-remote`, `delete-local`) is reachable ONLY when the machine
  is initialized (`lastSuccessfulSyncAt` set). This holds regardless of journal
  `stateRecords`, so a stale or foreign set of baselines can never by itself
  cause data loss. R0 is the minimal enforcement of this invariant for the
  no-prior-sync case; R2 extends it to the explicit uninitialized-prompt flow.

### Hook points (verified)

- The gate belongs in `SyncEngine.sync()` between Phase 1 (plan built) and
  Phase 2 (execute) — the same seam where `checkDestructivePlan` already runs.
  No change needed in `SyncExecutor` or the pure decision table.
- `SyncExecutor.executeDeleteRemote` already has an ETag guard (aborts if the
  remote changed since planning); it does not need to know about init state —
  the engine gate upstream is authoritative.
- The destination-fingerprint wipe (Phase 0) and the init gate are
  complementary: Phase 0 catches "pointed at a different remote," the init gate
  catches "same remote, but this machine's local/journal state is untrusted."

## Decision table deltas (per file)

Unchanged when **initialized**:

| local | remote | baseline | action |
|-------|--------|----------|--------|
| L0 | R+ | none | download (new remote file) |
| L0 | R= | present | delete-remote (I deleted it) |
| L+ | R+ | none | adopt if fingerprints match, else conflict |

Added gate when **uninitialized** (overrides the above):

| local | remote | action |
|-------|--------|--------|
| any | any that would delete | **blocked → prompt (R3)** |
| L0 | R+ | download (allowed; non-destructive) |
| L+ | R+ | adopt/upload (allowed; non-destructive) |

The ambiguous `L0 + R=` case — the one that destroyed the clone — can only
resolve to `delete-remote` after explicit initialization, never before.

## Out of scope

Implementation details (modal wiring, command/button placement, engine hook
points) are deferred. This spec fixes only the semantics: baseline trust,
explicit init, the uninitialized safety gate, and the clone action.

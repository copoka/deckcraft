# Adapter plan: DeckCraft against the RFC-0020 contracts

The application-local protocol is closed (ADR-0021 §3): #1975, #1998 and #2012 were closed as
superseded by RFC-0020 on 2026-11-23, and no pull request is open from this fork. `collab/pr15`
stays in `copoka/deckcraft` as the source of the adapter. This page is the inventory it starts
from: what each module becomes when the contracts land, which six surfaces the adapter has to
meet, what the shared layer has today, and the first increment.

Line counts are `find <dir> -name '*.rs' | xargs cat | wc -l` on `collab/pr15` at `647ce7e6`,
2026-11-23. They describe the code being inventoried, not a delivered head.

## What the branch holds

| module | lines | what it is under RFC-0020 |
|---|---|---|
| `crates/collab-proto` | 2 629 | The v1 envelope: handshake, operations, authoritative results, presence, limits, and `OpKind::{Command, Structural}`. Becomes the app's structured-command frame on the ordered path; `crdt.*` text frames stay opaque and are never parsed here. |
| `crates/collab/src/apply.rs` | 6 105 | The normaliser and adapter: wire operation → engine command through explicit `Target` keys, authoritative delta back. `class_of` already answers the contract's per-kind question (`Command` or `Structural`), which is what a scope lock keys on. App-side policy, stays here. |
| `crates/collab/src/local.rs` | 4 794 | The client document: normalise local commands, settle on `applied`, optimistic preview with rollback, authoritative apply. Becomes the client half of the session contract; its text path is replaced by `Texts`. |
| `crates/collab/src/session.rs` | 1 812 | `DocSession`: one order, `opId` dedup, journal tail, snapshots, the commit bytes. The server half behind the session contract. |
| `crates/collab/src/hub.rs` | 2 088 | Room membership, the role gate before any engine call, per-peer outboxes. The relay half: one room per document, sequence reconciled on join. |
| `crates/collab/src/client.rs` | 1 617 | Client state machine: pending queue, resend, recovery, version-gap resync, base transfer. The client half behind the same contract. |
| `crates/collab/src/history.rs` | 1 072 | Per-user undo: before/after images of the objects a step touched, `undo-conflict` when someone else changed them. App-side; the contract does not define undo (TASK §4). |
| `crates/collab/src/media.rs` | 617 | Content-addressed pictures, fonts, audio, video and 3-D models by `sha256`, with separate allowlists and budgets. A DeckCraft extension; the shared contract does not carry media. |
| `crates/collab/src/keys.rs` | 272 | Session-scoped slide and shape keys plus tombstones. Required because file ids are positional (see the model surface). |
| `crates/collab/src/presence.rs` | 211 | Latest-wins per actor, TTL, stable colour. Moves behind the presence contract; the app keeps only the projection (slide, shapes, cursor, viewport). |
| `delta.rs`, `animation.rs`, `transition.rs`, `link.rs` | 378 | The delta the client resolves keys against, the model↔wire conversions, and the one-method transport seam. App-side. |
| `crates/collab/src/transport.rs` | 481 | The desktop socket thread. Kept: the contract names no transport. |
| `crates/collab/tests` | 517 | Two-client convergence and a restarted room; the seed of the conformance harness. |
| `crates/collab-web` | 1 097 | The browser link and DeckCraft's side of the host bridge, `web_sys`-free and headless-tested. The messages are `craft-web-host`'s, pinned to the fork at `7666e2b1` (`#716`). |
| `crates/ui-egui/src/collab.rs` | 1 843 | Feature flag, sync indicator, participants pane, presence on the canvas, viewer gate. The participants cluster moves to the shared `craft_ui::presence::PresenceBar`. |
| `apps/deckcraft-collab-server` (src) | 2 796 | WebSocket, JWT, room map, the store (base + journal), the cabinet callback, metrics, limits. Stays a backend; nothing in the app names it. |
| `apps/deckcraft-collab-server/tests` | 3 557 | Socket, commit and gate suites, including the 120-slide soak and the three-client soak. |
| `apps/deckcraft-web/src/{host_session,host}.rs` | 213 new; ~400 changed | The hosting page's bridge; the `session` message opens the browser session. |
| `docs/collab` (without this page) | 7 161 | The protocol, the measurements and the run books; the conformance cases are taken from here. |

## The six surfaces it must meet

1. **Session.** Ordered submission with opaque scope locks for commands that do not commute, and
   lock-free `crdt.*` passthrough for text (RFC-0020, landing order item 2). The branch already
   orders in `DocSession`, dedups by `opId`, reconciles the join version from the store's
   persisted base version and refuses a structural step whose base is not the head. The gap is the lock:
   v1 serialises everything and has no scope, because the room is the only writer. The adapter
   supplies the scope from `class_of`/`Target` (the slide key for a slide step, the shape key for
   a shape step, the presentation for a document-level one) and lets the contract's session layer
   own the lock table. The RFC's session contract has no PR in `craft-libs` yet; `#128` is the
   CraftHub GOPM core (`DocSchema`/`HubAdapter` over an object tree), not this contract, and must
   not be pinned as if it were.

2. **Presence.** Ephemeral latest-wins per peer and kind, allowed without edit permission, never
   in the version, journal or snapshot. `presence.rs` and `PresenceMsg` hold this today and the
   tests assert it never moves a version. The adapter feeds `craft_ui::presence::PresenceBar`
   and the host's `Participant` list (`#716`); the title bar's sync state reads
   `craft_web_host::status` (`#2108`). One mapping detail: the host contract's role strings
   include `commenter`, which DeckCraft does not have — it maps to `viewer`, never to `editor`.

3. **Texts.** Today text is a relay-ordered offset command (`text.insert`, `text.delete`,
   `text.set`, `text.setBody`, notes, `format.*` over ranges and carets), and the server rebases
   offsets against the operations since the base. RFC-0020 moves text to the lock-free `Texts`
   path: the engine choice is a shared-layer decision, `#345` (`craft-crdt`) is an oracle only
   and conditional on the differential suite. One conformance gap is ours and not the
   contract's: guarantee 1 says that when a structural command deletes an object another peer is
   typing into, the delete wins identically on every replica **and the typist keeps a local
   recoverable copy that is surfaced**. The branch refuses the late operation deterministically
   (tombstones, tested), but throws the typed text away. That copy is the first thing the
   adapter adds.

4. **The deck model.** `deckcraft_model::Presentation` over `crates/pptx`'s parts: `Arc<Slide>`
   copy-on-write, `shapes` order as z-order, inheritance resolved at render. Unmodelled content
   survives through `ShapeKind::Opaque`, `raw_ext`, `Presentation::kept_parts`, `Chart.kept` and
   `Diagram` XML; the adapter edits only through engine commands and never re-encodes a part it
   does not model. File identity is not session identity: the writer renumbers slides
   (`p:sldId@id = 256 + i`) and `cNvPr` per part, which is why keys are session-scoped and kept
   out of the file. Layout must not depend on who typed last: a remote edit is applied before the
   next layout pass, never inside one.

5. **Commit and the cabinet.** The server builds the `.pptx` from the authoritative document and
   posts `{fileId, name?, bytes, sha256, version, expectedVersion?}` to the cabinet, which answers
   `{version, etag}`; `409` means the file changed outside the session and offers a copy, never an
   overwrite. The accepted version survives a restart, and a commit without a callback returns
   the bytes. The shared cloud contract is `craft-libs#160`, merged 2026-11-17 in `fd7232d7`:
   `craft_ui::cloud::Event::Saved`, `craft_cloud::Entry::writable`, `Fault::Conflict` for a stale
   etag. DeckCraft does not depend on `craft-cloud` yet — its `CabinetConfig`/`Published` carry
   the same shape — so the adapter maps to those types instead of growing a second vocabulary.
   Still outside this repository: the cabinet's own endpoint and a desktop "save to cabinet"
   command (production.md, gap P0).

6. **Roles.** JWT claim → `Role`; the hub refuses a viewer's operation before the engine; the UI
   gate is a courtesy, not the gate; the owner closes the room, and a close without its recovery
   snapshot keeps the room live. A revoked participant must lose its seat (socket tests cover a
   dropped and a killed connection). The contract's rule is that the backend enforces and the app
   may mirror it; the adapter must not be the only gate.

## State of the contracts (2026-11-23)

- `craft-libs#160` (cloud contracts) **merged** 2026-11-17, merge commit `fd7232d7`, head
  `aa4647a7`: `Event::Saved`, `Fault::Conflict`, `Entry::writable`, and the shared presence
  control in `craft-ui`.
- `craft-libs#128` (`craft-collab`, the CraftHub §2.15 core) open, head `9a6148e0`, last updated
  2026-10-24. It is the GOPM batch core with `DocSchema`/`HubAdapter`, not the RFC-0020 live
  session; the session contract of landing order item 2 has no PR and no head.
- `craft-libs#345` (`craft-crdt`) open, head `cef9e963`, updated 2026-11-23T20:04Z. Conditional
  on the differential oracle; may close in favour of a mature engine behind the same `Texts` API.
- `craft-libs#716` (web-host session messages) open, head `7666e2b1`, updated 2026-11-08.
  DeckCraft already pins `copoka/craft-libs` at that revision in `apps/deckcraft-web`
  (ADR-0016 §3); `crates/collab-web` mirrors the messages without depending on the crate.
- `craft-libs#2108` (host statuses for a title bar) open, head `2882d72`, updated
  2026-11-23T20:04Z.
- House rule 10 forbids pinning an unmerged revision; the fork pin under ADR-0016 §3 is the
  recorded exception, and no contract change is made locally.

## First increment

One commit series on the adapter base, app side only, no protocol change, **at most 1 500 lines
of new work**. The base is `collab/pr15`; the drift fix (merge or rebase onto `origin/main`
`2d031bb6`, seven conflicted files: `AGENTS.md`, `Cargo.lock`, `apps/deckcraft-web/src/web.rs`
and four `crates/pptx` files) is part of that pass and lands as its own commits, not inside the
increment. The push credential has no `workflow` scope, so a merge that carries main's
`.github/workflows` changes is not pushable from here; a rebase adds no workflow commit and is
the route to take. The PR body separates the inherited branch from the increment, so the review
delta is the increment.

The increment:

1. `crates/collab/src/contract.rs` — the seam in the RFC's vocabulary: `Class` (from
   `apply::class_of`), `Scope` (an opaque string built from `Target`), `Session`
   (submit/sequenced/applied/join sequence), `Presence` (publish/latest-wins), `Commit`
   (bytes → version/etag, `Conflict`), `RoleGate`. It is implemented by `Hub`, `DocSession` and
   `ClientSession` and changes no wire byte. It is deliberately app-side: the shared trait does
   not exist yet, and when it lands it delegates to this seam rather than re-cutting the adapter.
2. Deleted-scope recovery (RFC-0020 guarantee 1) — a text operation refused because its target
   was removed by an acknowledged structural command returns its text to the originator as a
   recoverable copy (`Effect::RecoveredText`), surfaced by the UI, while every replica keeps the
   same document. Tests: two clients, one types while the other deletes the shape, and the slide.
3. A conformance harness over the seam, running the branch's existing cases on the two-client
   and restarted-room fixtures: convergence, stale structural step → resync, reconnect after a
   relay restart, viewer refused at the seam and in the hub, commit `409` keeping the original,
   and the deleted-scope case above.
4. `docs/collab/adapters.md` and the `STATUS.md` entry.

Budget: seam ~350, recovery ~300 plus ~250 tests, harness ~350, docs ~150 — about 1 400 lines.
Not in this increment: a `Texts` dependency (waits for the session contract and the oracle
verdict on `#345`), the host-message swap to the merged `#716`, and the scope-lock table itself.

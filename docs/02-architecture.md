# Telescale — Architecture

This document describes the system architecture of the Telescale POC on
Veilid. Data schemas live in `03-poc-spec.md`; protocol flows in
`04-protocols.md`.

## Layer model

```
┌─────────────────────────────────────────────────────────┐
│  Apps (POC: CLI chat / kv-doc demo)                     │
├─────────────────────────────────────────────────────────┤
│  Telescale layer (this project)                         │
│  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────────┐  │
│  │ Identity │ │ Groups & │ │  Sync  │ │ Routing +    │  │
│  │          │ │ Invites  │ │ Engine │ │ ACL          │  │
│  └──────────┘ └──────────┘ └────────┘ └──────────────┘  │
├─────────────────────────────────────────────────────────┤
│  Veilid (veilid-core)                                   │
│  DHT records · private routes · AppCall/AppMessage ·    │
│  crypto (VLD0) · NAT traversal · relays · bootstrap     │
├─────────────────────────────────────────────────────────┤
│  Internet (UDP / TCP / WebSocket)                       │
└─────────────────────────────────────────────────────────┘
```

Telescale never talks to sockets directly; everything below the Telescale
layer is Veilid's job. Conversely, Veilid knows nothing about groups, members,
or ACLs.

## Components

### 1. Identity

- A **member identity** is an Ed25519 keypair (Veilid VLD0 crypto system).
  In the POC, one identity = one device = one keypair; multi-device user
  identity is future work.
- The secret key lives in Veilid's ProtectedStore (encrypted at rest).
- A member's public key is their stable name inside a group. Display names
  are metadata, never identity.
- Identity keys are **application keys**, deliberately distinct from the
  Veilid node keypair — a member can move between nodes/devices (future) and
  node-level identity stays out of the trust model.

### 2. Groups and the root of trust

A **group** is anchored by two keypairs generated at creation:

- **Group identity key** — long-lived; its public half is the group ID.
  Signs the things that define the group: roster versions, ACL policy,
  invites. Held by admins (POC: the single creator).
- **Group record owner key** — owns the group's main DHT record (Veilid DHT
  ownership = write access). In the POC the admin holds both.

Group state is published as DHT records (layout in `03-poc-spec.md`):

- **Group record** — header, signed member roster, signed ACL policy, and a
  route book (each member's current private-route blob for direct contact).
  Owner-writable only; members read and verify signatures.
- **Member outbox records** — one per member, owned by that member. Holds the
  member's signed, encrypted update log (application data, sync metadata).
  This keeps the multi-writer problem out of the group record: everyone
  writes only records they own.

Roster and ACL carry monotonically increasing **versions**; every signed
artifact references the version it was authored against.

### 3. Invitations (membership plane)

Membership changes only through invites — there is no open "join" endpoint.

- An **invite** is a compact signed token: group ID, group DHT record key,
  an admin contact route, a single-use nonce, expiry, and the group identity
  signature. Encoded as `telescale://invite/<base64>` (link or QR).
- Redemption is an `AppCall` to the admin over a Veilid private route; the
  admin verifies the nonce/expiry, and on acceptance appends the new member
  (pubkey + role) to the roster, bumps the roster version, re-signs, and
  re-wraps the group data key for the new member set.
- Invites are capability-shaped: possession + a fresh keypair is sufficient
  to request membership; the admin's acceptance is what grants it.

### 4. Sync engine (explicit sync)

Telescale converges state through deliberate, observable sync — no
epidemic gossip:

- Members **watch** the group record via Veilid DHT watches; a version bump
  (roster/ACL/route book) triggers a fetch-verify-apply cycle.
- Application data syncs via **per-member sequence numbers**: each member's
  outbox is an append-only log; every other member tracks
  `last_seen[member] = seq` (a version vector) and fetches only the gap.
- A `SyncRequest`/`SyncResponse` AppCall pair lets two members reconcile
  directly (faster than DHT propagation, and works for large gaps); the DHT
  remains the fallback and the source of truth for offline members.
- All fetched updates are verified (author in roster at that version, ACL
  permits the operation, signature valid) **before** being applied to local
  state. Invalid updates are dropped and logged, never partially applied.
- Local state lives in Veilid's TableStore; the node can restart and resync
  from its version vector.

### 5. Routing + ACL layer

Logical routing above Veilid's transport routing:

- **Addressing**: messages target a member (direct AppCall via their route
  book entry) or the group (write to own outbox; others pull via sync).
- **Private routes**: members publish a private-route blob in the route book
  and refresh it when Veilid rotates routes. Peers never need each other's IP.
- **ACL policy**: a signed document mapping **roles → capabilities**. POC
  roles: `admin`, `member`, `observer`. Capabilities: `invite`, `write`,
  `read`, `manage_roster`, `manage_acl`.
- **Enforcement is local and universal**: every node checks every inbound
  update against the roster + ACL at the update's declared version. There is
  no reference monitor to bypass — an unauthorized write is simply rejected
  by every honest member.

### 6. Content encryption

- Each group has a symmetric **group data key** (XChaCha20-Poly1305). All
  outbox payloads are encrypted with it — DHT nodes and non-members see
  ciphertext.
- The current data key is distributed by **key-wrapping**: encrypted to each
  member's X25519 key and published in the group record (`keywrap` subkey).
- On revocation the admin generates a new data key and re-wraps for the
  reduced member set → forward exclusion. (POC does not re-encrypt history;
  a revoked member keeps what they already saw — same as any messaging
  system.)

## Crate layout (planned)

```
rmesh/
├── Cargo.toml                 # workspace
├── crates/
│   ├── telescale-proto/       # schemas, envelopes, signing & validation.
│   │                          #   Pure: no I/O, no veilid dep. The TCB.
│   ├── telescale-core/        # group/invite/sync/ACL state machines,
│   │                          #   storage traits. Depends on -proto.
│   ├── telescale-veilid/      # veilid-core adapter: DHT record I/O,
│   │                          #   routes, AppCall plumbing, watches.
│   └── rmesh-cli/             # the POC binary: node daemon + CLI commands.
└── docs/
```

The dependency rule: `proto ← core ← veilid-adapter ← cli`. Everything
security-critical is in `telescale-proto` + `telescale-core` and testable
with an in-memory fake network — the Veilid adapter is replaceable plumbing.

## Trust model (POC)

| Actor | Trusted with |
|---|---|
| Group admin | Membership decisions, ACL policy, data-key rotation. **Not** able to forge other members' messages. |
| Members | Reading group content; writing per their role. |
| Veilid DHT / relay nodes | Availability only. Can withhold or delay data; cannot read content (E2EE) or forge state (signatures). |
| Bootstrap servers | First contact into Veilid. Self-hostable. |

Known POC-accepted risks (tracked for post-POC): admin is a single point of
membership control (no admin quorum); DHT withholding = availability attack;
metadata (record access patterns) partially mitigated by Veilid safety
routes; no post-compromise security beyond data-key rotation.

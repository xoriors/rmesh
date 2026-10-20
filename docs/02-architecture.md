# rmesh — Architecture

System architecture of the rmesh POC on Veilid. Decisions:
`00-brainstorm-decisions.md` (incl. the 2026-10 amendments). Positioning:
`06-landscape.md`. Data schemas: `03-poc-spec.md`. Protocol flows:
`04-protocols.md`.

## Layer model

```
┌──────────────────────────────────────────────────────────┐
│  Local apps (psql, ssh, curl … anything TCP)             │
├──────────────────────────────────────────────────────────┤
│  rmesh daemon (this project, unprivileged)               │
│  ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌───────────┐  │
│  │ Identity │ │ Groups & │ │ Membership │ │ Routing + │  │
│  │          │ │ Invites  │ │ Sync       │ │ ACL       │  │
│  └──────────┘ └──────────┘ └────────────┘ └───────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Data plane: port-forward tunnels (SSH-forward-like)│  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│  Transport adapter (narrow trait — swappable)            │
├──────────────────────────────────────────────────────────┤
│  Veilid (veilid-core)      [planned option: Iroh +       │
│  DHT records · private      mainline DHT]                │
│  routes · AppCall/AppMessage · crypto (VLD0) ·           │
│  NAT traversal · relays · bootstrap                      │
├──────────────────────────────────────────────────────────┤
│  Internet (UDP / TCP / WebSocket)                        │
└──────────────────────────────────────────────────────────┘
```

The daemon never touches sockets for mesh traffic — that's the transport's
job. The **only** crate that imports veilid-core is the adapter; everything
above it programs against a small trait (`Transport`: publish/read/watch
records, direct call, direct message, route handle lifecycle). This serves
testability (an in-memory fake drives all core tests) and keeps the planned
Iroh migration honest — everything above the adapter must carry over
unchanged.

## Components

### 1. Identity

- A **node identity** is an Ed25519 keypair (VLD0). One node = one daemon =
  one keypair in the POC; an X25519 keypair handles group-key wrapping.
- Secret keys live in Veilid's ProtectedStore (encrypted at rest).
- The public key is the node's stable name inside groups; display names are
  metadata. Application identity is deliberately distinct from the Veilid
  node keypair, so the transport swap doesn't change who anyone is.

### 2. Groups and the root of trust

A group is anchored by two keypairs generated at creation:

- **Group identity key** — its public half is the group ID; signs roster
  versions, ACL policy, and invitation tokens. Held by admins (POC: the
  single creating node). Per-admin signed authority — not a shared network
  secret (differentiator #1, `06-landscape.md`).
- **Group record owner key** — owns the group's DHT record (DHT ownership =
  write access). POC: the admin holds both.

Group state lives in DHT records (layout in `03-poc-spec.md`):

- **Group record** — header, signed roster, signed ACL policy, route book
  (members' current contact routes), key-wrap set. Owner-writable; members
  read + verify. Encrypted with the group key — DHT nodes and non-members
  see ciphertext (differentiator #2).
- **Per-member presence records** — owned by each member: heartbeat
  timestamp, current route blob, exposed services. Each node writes only
  records it owns — no multi-writer contention anywhere.

Roster and ACL carry monotonic versions; every signed artifact references
the version it was authored against.

### 3. Invitations (membership plane)

Membership changes only through invitations — there is no open join.

- An **invitation token** is a compact signed voucher: group ID, group
  record key, admin contact route, offered role, single-use nonce, expiry,
  group identity signature. Shared as `rmesh://invite/<base64>`
  (paste/link/QR).
- Redemption is a direct call to the admin over the mesh; the admin
  verifies the voucher, appends the member to the roster, bumps + re-signs
  it, and re-wraps the group key for the new member set.
- Any member can *carry* an invite out-of-band; only holders of the group
  identity key can *mint* one (POC: admin). Voucher validation is done by
  every member independently.

### 4. Membership sync (explicit, pull-based)

Per brainstorm decision #4: **explicit sync, not gossip**, for the POC.

- **DHT watch/poll**: members watch the group record; a roster/ACL version
  bump triggers fetch → verify → apply.
- **Direct pull**: on startup, on watch gaps, and periodically, a node asks
  any reachable member *"who's in this group now?"* (`MembersRequest`) and
  reconciles: the response carries the signed roster at its version, so a
  stale or lying peer can't inject members — worst case it withholds news.
- **Presence & liveness**: each member heartbeats its presence record every
  30 s. Peers mark a member `offline` after 3 missed beats — presence is
  *advisory* (routing hint), never a membership change.
- **Leave**: a leaving member publishes a signed goodbye and the admin
  prunes it from the roster (version bump + key rotation). Crash-leave is
  prolonged `offline`; pruning stays an explicit admin action.
- Everything applied passes signature + version validation first; invalid
  data is dropped and logged.

Post-POC this module gains a gossip strategy behind the same interface
(`MembershipSync` trait) — the upgrade is a new impl, not a redesign (and
on an Iroh transport it would ride `iroh-gossip`).

### 5. Routing + ACL layer

The enforcement core — group membership determines who can send what to
whom:

- **Addressing**: `(group, member)` for control calls; `(group, member,
  service)` for tunnels. Contact routes come from the route book / presence
  records; peers never see each other's IPs (Veilid private routes —
  differentiator #5).
- **ACL policy**: signed document mapping **roles → capabilities** and
  **service rules**. POC roles: `admin`, `member`, `observer`.
  Capabilities: `invite`, `manage_roster`, `manage_acl`, `connect`,
  `expose`. Service rules: which roles/members may connect to which exposed
  services — first match wins, default **deny** (differentiator #4).
- **Exposed services**: a member declares named services (`postgres →
  127.0.0.1:5432`) in its presence record; the bind target stays local and
  unpublished. Only declared services are reachable — the daemon is a
  deny-by-default gateway, not an open proxy to localhost.
- **Enforcement is local and universal**: the *destination* daemon is the
  authority for its own services — it validates the caller's membership,
  role, and the service rule before opening the local connection; callers
  pre-check to fail fast. Every rejection is logged with a reason.

### 6. Data plane: port-forward tunnels

The SSH-port-forwarding model — unprivileged, no TUN, no root
(differentiator #3):

- `rmesh forward --group g --to nodeB:postgres --listen 127.0.0.1:8443`
  makes the local daemon listen on 8443; each accepted TCP connection
  becomes a **tunnel**: `TunnelOpen` (authenticated, ACL-checked at B) then
  a bidirectional stream of `TunnelData` chunks, then `TunnelClose`.
- Transport mapping (POC): tunnels run over Veilid AppCall/AppMessage with
  per-tunnel ids, chunking, sequence numbers, and backpressure via
  acknowledged windows — Veilid has no native byte-streams, so the adapter
  provides a small reliable-stream shim. (On Iroh this shim collapses to a
  native QUIC stream — another reason the adapter owns it.)
- Every chunk is end-to-end encrypted member↔member (Veilid transport
  encryption plus per-tunnel session keys — content is private even from
  other group members; private routes hide the traffic pattern too).
- Throughput over private routes will be modest; the POC target is
  correctness (`psql`, `ssh`, `curl` work reliably), not line-rate. A
  latency/throughput measurement is a POC exit criterion, informing the
  Iroh-migration decision (per-group "direct mode vs. private-route mode"
  is an open question in the roadmap).
- Roster/ACL changes re-evaluate open tunnels: **revocation tears down live
  tunnels**, not just future ones.

### 7. Content encryption

- Group control state (roster, ACL, presence, service catalogs) is
  encrypted with a symmetric **group key** (XChaCha20-Poly1305,
  epoch-numbered): DHT nodes and non-members see ciphertext.
- The group key is distributed by key-wrapping to each member's X25519 key
  (`keywrap` subkey), and **rotated on revocation** → forward exclusion.
  Members keep current + previous epoch briefly for rotation races.
- Tunnels additionally derive per-tunnel session keys (X25519 ECDH between
  the two members + BLAKE3 KDF) so tunnel content isn't readable even by
  other group members.

## Daemon & CLI shape

One binary, two roles: `rmeshd` (long-running, unprivileged; systemd *user*
unit or container sidecar) and `rmesh` (CLI over a local unix socket):

```
rmesh node status | id show
rmesh group create <name> | show | members | leave
rmesh invite create --group g --role member --ttl 24h
rmesh join <rmesh://invite/…>
rmesh expose add postgres 127.0.0.1:5432 --group g
rmesh forward --group g --to <member>:postgres --listen 127.0.0.1:8443
rmesh acl show | allow --group g --role member --service postgres
rmesh revoke --group g <member>
rmesh doctor
```

## Crate layout (planned)

```
rmesh/
├── Cargo.toml                 # workspace
├── crates/
│   ├── rmesh-proto/           # schemas, envelopes, signing & validation.
│   │                          #   Pure: no I/O, no veilid dep. The TCB.
│   ├── rmesh-core/            # group/invite/sync/ACL/tunnel state machines,
│   │                          #   Transport + MembershipSync traits, storage
│   │                          #   traits. Depends only on -proto.
│   ├── rmesh-veilid/          # the ONLY crate importing veilid-core:
│   │                          #   Transport impl, DHT I/O, routes, stream shim.
│   └── rmesh-cli/             # rmeshd daemon + rmesh CLI (unix socket RPC).
└── docs/
```

Dependency rule: `proto ← core ← veilid ← cli`. Everything security-critical
sits in `proto` + `core`, testable with an in-memory fake `Transport` — and
the same fake is the harness that later validates an `rmesh-iroh` adapter.

## Trust model (POC)

| Actor | Trusted with |
|---|---|
| Group admin | Membership decisions, ACL policy, key rotation. Cannot forge members' messages or read tunnels it isn't an endpoint of. Every admin action is signed and attributable (no shared network secret — `06-landscape.md` §1). |
| Members | Reading group state; connecting per ACL; honesty of their own presence/service declarations (affects only routing to themselves). |
| Veilid DHT / relays | Availability only — can withhold/delay, cannot read (E2EE) or forge (signatures); private routes deny them the traffic graph. |
| Bootstrap servers | First contact into the Veilid network; self-hostable. |

POC-accepted risks (tracked in the roadmap): single-admin authority (quorum
is post-POC); DHT withholding as availability attack; presence records are
self-declared (a member can lie — it only affects routing to *itself*); no
post-compromise security beyond key rotation; traffic-analysis resistance
bounded by what Veilid safety/private routes provide.

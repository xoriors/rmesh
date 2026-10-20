# rmesh — Architecture

System architecture of the rmesh POC on iroh. Decisions: `00-brainstorm-
decisions.md` (incl. the 2026-10 amendments). Positioning: `06-landscape.md`.
Data schemas: `03-poc-spec.md`. Protocol flows: `04-protocols.md`.

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
│  │ Data plane: port-forward tunnels over QUIC streams │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│  Transport adapter (narrow trait — fake for tests)       │
├──────────────────────────────────────────────────────────┤
│  iroh: QUIC endpoints · dial-by-key (Ed25519) · hole     │
│  punching · relays · discovery (pkarr/mainline DHT, DNS, │
│  mDNS) · iroh-gossip (post-POC)                          │
├──────────────────────────────────────────────────────────┤
│  Internet (UDP; relay fallback over TCP/HTTPS)           │
└──────────────────────────────────────────────────────────┘
```

The daemon never touches sockets for mesh traffic. Only `rmesh-iroh` imports
iroh; everything above programs against a small `Transport` trait (dial,
accept, open/accept streams, publish/resolve announce records). The trait's
job today is **testability** (an in-memory fake drives all core tests);
keeping a second real backend possible is a side benefit, not a goal —
the Veilid-POC plan it once served was dropped (amendment 6′).

## Components

### 1. Identity

- A **node identity** is an Ed25519 keypair, used directly as the iroh
  endpoint key — so the node's mesh address (EndpointId) *is* its identity,
  and dialing a member means dialing its key. One node = one daemon = one
  keypair in the POC.
- An X25519 keypair (derived storage, same protection) handles group-key
  wrapping.
- Secrets live in an encrypted on-disk keystore (age-style file key or OS
  keychain where present); public keys are stable names, display names are
  metadata.
- QUIC gives mutual authentication of EndpointIds on every direct
  connection (TLS channel binding), so direct RPC needs no extra signatures
  on the channel — signatures live on the *data* that travels further than
  one hop (state blobs, invites, announces).

### 2. Groups and the root of trust

A group is anchored by the **group identity keypair** generated at creation:
its public half is the group ID, it signs the group's state and invitation
tokens, and (as the pkarr keypair) it signs the group's announce record.
POC: the single creating node (admin) holds it.

Group state has two tiers:

- **Group announce** — a tiny *signed* pkarr record published to the
  mainline DHT under the group key: state version counters, the BLAKE3 hash
  of the current state blob, and a few bootstrap member EndpointIds. It fits
  pkarr's ~1 KiB DNS-packet budget and is the only thing the public DHT
  ever sees. It proves *freshness* and *where to fetch*; it contains no
  membership data.
- **GroupState blob** — the real state: roster, ACL policy, key-wrap set.
  Signed by the group key, **encrypted with the group key material** so only
  members read it, content-addressed by the hash in the announce, and served
  by every member over direct QUIC (`StateRequest`). New joiners get it from
  the admin at join time; everyone else fetches on version bumps.

This replaces the Veilid-era subkey layout: the public DHT holds a pointer
+ proof, members hold and serve the encrypted truth. (Same shape Rayfish
uses for its record — except rmesh's state is ciphertext, not public.)

### 3. Invitations (membership plane)

Membership changes only through invitations — there is no open join.

- An **invitation token**: group ID, admin EndpointId, offered role,
  single-use nonce, expiry, group-key signature. Shared as
  `rmesh://invite/<base64>` (paste/link/QR).
- Redemption: the joiner dials the admin by EndpointId (iroh discovery
  resolves it) and sends `JoinRequest` with the invite; the channel proves
  the joiner holds its claimed key. The admin verifies voucher + nonce,
  appends the member to the roster, bumps + re-signs + re-encrypts the
  state, re-wraps the group key, republishes the announce.
- Every member independently re-validates the roster it fetches — a member
  entry carries the hash of the invite that admitted it (audit trail).

### 4. Membership sync (explicit, pull-based)

Decision #4 stands: **explicit sync, no gossip, in the POC.**

- **Announce poll/refresh**: members re-resolve the group's pkarr record on
  a timer and on reconnect; a version/hash change triggers
  `StateRequest` → verify → apply.
- **Direct pull**: on startup, periodically, and on any gap, a node asks any
  reachable member `MembersRequest` → the signed, encrypted state at its
  version — a peer can withhold news, never fabricate members.
- **Change nudge**: after publishing new state, the admin sends a
  lightweight `Poke {version}` to reachable members (fire-and-forget) so
  convergence doesn't wait on the poll timer.
- **Presence & liveness**: members heartbeat (`Presence` message with
  exposed-services list) to reachable peers every 30 s; a member is marked
  offline after 3 missed beats. Presence is advisory routing state, never
  membership.
- **Leave**: signed `Goodbye`; admin prunes (roster bump + key rotation).
  Crash-leave is prolonged offline; pruning stays an explicit admin action.

Post-POC, `MembershipSync` gains an `iroh-gossip` implementation (per-group
topic) behind the same trait — presence and pokes become gossip messages;
the validation pipeline is unchanged.

### 5. Routing + ACL layer

- **Addressing**: `(group, member)` — and members are EndpointIds, so
  addressing *is* dialing; iroh discovery replaces the Veilid-era route
  book entirely.
- **ACL policy**: signed document mapping **roles → capabilities** and
  **service rules**. POC roles: `admin`, `member`, `observer`. Capabilities:
  `invite`, `manage_roster`, `manage_acl`, `connect`, `expose`. Service
  rules: which roles/members may connect to which service on which node —
  first match wins, default **deny**.
- **Exposed services**: a member declares named services (`postgres →
  127.0.0.1:5432`); the bind target stays local and unpublished. Only
  declared services are reachable — the daemon is a deny-by-default
  gateway, not an open proxy to localhost.
- **Enforcement is local and universal**: the destination daemon is the
  authority for its own services — it checks the caller's membership, role,
  and service rule against *its* current state before connecting out;
  callers pre-check to fail fast. Every rejection is logged with a reason.

### 6. Data plane: port-forward tunnels

- `rmesh forward --group g --to nodeB:postgres --listen 127.0.0.1:8443`
  makes the daemon listen locally; each accepted TCP connection opens one
  **QUIC bidirectional stream** to B (ALPN `rmesh/0`): a `TunnelOpen`
  header frame, an accept/reject frame, then raw byte piping both ways.
  QUIC provides the reliability, ordering, flow control, and backpressure
  the Veilid-era chunk/ack shim had to hand-build — that shim is gone.
- Encryption: iroh QUIC is end-to-end between the two members' keys; relays
  and other members see ciphertext. No extra per-tunnel key layer in the
  POC (it added nothing against this threat model once the transport
  authenticates both endpoint keys).
- Roster/ACL changes re-evaluate open tunnels: **revocation tears down live
  streams**, not just future ones.

### 7. Content encryption

- The **group key** (XChaCha20-Poly1305, epoch-numbered) encrypts the
  GroupState blob (and, post-POC, gossip payloads). DHT nodes and
  non-members see only the announce.
- Distribution by key-wrapping to each member's X25519 key inside the
  state; **rotated on every membership change** → forward exclusion.
  Members keep current + previous epoch briefly for rotation races.

## Daemon & CLI shape

One binary, two roles: `rmeshd` (long-running, unprivileged; systemd user
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
│   │                          #   Pure: no I/O, no iroh dep. The TCB.
│   ├── rmesh-core/            # group/invite/sync/ACL/tunnel state machines,
│   │                          #   Transport + MembershipSync traits,
│   │                          #   storage traits. Depends only on -proto.
│   ├── rmesh-iroh/            # the ONLY crate importing iroh:
│   │                          #   endpoints, discovery/pkarr publish,
│   │                          #   streams, (later) iroh-gossip.
│   └── rmesh-cli/             # rmeshd daemon + rmesh CLI (unix socket).
└── docs/
```

Dependency rule: `proto ← core ← iroh ← cli`. Everything security-critical
sits in `proto` + `core`, tested against the in-memory fake transport.

## Trust model (POC)

| Actor | Trusted with |
|---|---|
| Group admin | Membership, ACL policy, key rotation. Cannot forge members' traffic or read tunnels it isn't an endpoint of. Every admin action is signed and attributable (no shared network secret — see `06-landscape.md` §1). |
| Members | Reading group state; connecting per ACL; honesty of their own presence/service declarations (affects only routing to themselves). |
| Mainline DHT nodes | Availability of the announce record only — it's signed (unforgeable) and reveals no membership. |
| iroh relays | Availability + connection metadata (who connects to whom, when); never content. |
| iroh discovery infra (DNS/pkarr relays) | Availability of address resolution; records are node-signed. |

POC-accepted risks (tracked in the roadmap): single-admin authority (quorum
is post-POC); DHT/relay withholding as availability attack; connection
metadata visible to relay operators (rmesh claims content + membership
privacy, not transport anonymity — `01-vision.md`); no post-compromise
security beyond key rotation.

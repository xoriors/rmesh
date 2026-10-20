# rmesh POC — Data Spec

Concrete schemas for the POC on iroh. Encoding is **canonical CBOR** (serde +
`ciborium`, deterministic map ordering) — everything signed must have a
canonical byte form. Hashes BLAKE3, signatures Ed25519, DH X25519, symmetric
crypto XChaCha20-Poly1305 (RustCrypto crates; same suite the Veilid-era spec
used, now dependency-free of any framework).

> iroh specifics (EndpointId types, pkarr record limits, ALPN registration)
> must be re-verified against the iroh version pinned at implementation
> time; the shapes here are the contract.

## Conventions

```rust
type PubKey    = [u8; 32];   // Ed25519 public key
type Sig       = [u8; 64];   // Ed25519 signature
type GroupId   = PubKey;     // group identity public key (also pkarr key)
type NodeId    = PubKey;     // node identity = iroh EndpointId
type Hash      = [u8; 32];   // BLAKE3
type Nonce     = [u8; 24];
```

Every signed struct: signature over `blake3(domain_sep ||
canonical_cbor(struct minus sig))`, domain separator per type
(`b"rmesh/state/v1"`, `b"rmesh/invite/v1"`, …) — prevents cross-context
signature reuse.

## Published state

### Group announce (pkarr record on the mainline DHT)

Published under the **group keypair** (pkarr signs the whole record with
it). Budget: fits a pkarr DNS packet (~1 KiB). Public by nature — contains
freshness + fetch pointers, **no membership data**.

```rust
struct GroupAnnounce {
    v: u8,                    // schema version = 1
    state_version: u64,       // monotonic; = GroupState.version
    key_epoch: u64,
    state_hash: Hash,         // blake3 of the encrypted GroupState blob
    bootstrap: Vec<NodeId>,   // ≤ 4 members likely online (admin first);
                              //   joiners/syncers dial these for the blob
    published_at: u64,
}
// carried in TXT-style records inside the signed pkarr packet;
// pkarr's own signature (group key) covers it — no inner sig needed.
```

Republished on every state change and on a keepalive timer (mainline
records expire; POC: republish every 60 min, and bootstrap members
re-publish the latest announce if the admin is offline — same bytes, same
signature, so relays/DHT accept it from anyone).

### GroupState blob (served member-to-member)

The truth. `enc(group_key_epoch, canonical_cbor(GroupStateInner))`, plus a
plaintext envelope that lets a joiner bootstrap:

```rust
struct GroupState {
    v: u8,
    group_id: GroupId,
    version: u64,             // monotonic, +1 steps only
    key_epoch: u64,
    keywraps: Vec<KeyWrap>,   // plaintext section: who can unwrap
    ct: Vec<u8>,              // encrypted GroupStateInner
    sig: Sig,                 // group identity key, over all of the above
}
struct KeyWrap {
    node: NodeId,
    ephemeral_pub: [u8; 32],  // sealed-box: XChaCha20(group_key),
    nonce: Nonce,             //   key = X25519(eph_sk, member.dh_pub)
    ct: Vec<u8>,
}

struct GroupStateInner {
    name: String,             // display only, ≤ 64 bytes
    created_at: u64,
    admin: NodeId,            // POC: single admin
    members: Vec<MemberEntry>,
    acl: AclPolicy,
}
struct MemberEntry {
    node: NodeId,             // = iroh EndpointId → dialable directly
    dh_pub: [u8; 32],         // X25519, for key-wrapping
    role: Role,               // Admin | Member | Observer
    added_at: u64,
    added_by: NodeId,
    invite_hash: Hash,        // blake3 of redeemed invite (audit)
}
struct AclPolicy {
    version: u64,
    role_caps: Vec<(Role, Vec<Capability>)>,
    service_rules: Vec<ServiceRule>,
}
// Capability = Invite | ManageRoster | ManageAcl | Connect | Expose
// POC defaults: Admin → all; Member → Connect, Expose; Observer → (none)
struct ServiceRule {
    service: String,          // service name or "*"
    on: NodeSel,              // Any | Node(NodeId)
    allow: RoleSel,           // Role(Role) | Node(NodeId)
}
```

No route book: members are EndpointIds and iroh discovery (pkarr/DNS/mDNS)
resolves their addresses — addressing is the transport's job now.

## Invitation token

```rust
struct Invite {
    v: u8,
    group_id: GroupId,
    admin: NodeId,            // whom to dial for redemption
    role: Role,               // offered role
    nonce: Nonce,             // single-use
    expires_at: u64,
    sig: Sig,                 // group identity key = the voucher
}
```

Wire form: `rmesh://invite/` + base64url(canonical CBOR). The admin keeps a
nonce table; redemption consumes the nonce. The group announce is found via
the group key in `group_id` — the invite alone is enough to bootstrap.

## Wire protocol (QUIC, ALPN `rmesh/0`)

Two stream kinds on a member↔member connection. The QUIC handshake mutually
authenticates both NodeIds, so **control messages need no per-message
signatures** — authentication is channel binding; only data that outlives
the channel (state blobs, invites, announces) carries signatures.

### Control streams — length-prefixed CBOR frames

```rust
enum Ctl {
    // membership plane
    // the ONLY message accepted from a non-member (validated by voucher;
    // the channel already proves the sender holds `node`)
    JoinRequest  { invite: Invite, node: NodeId, dh_pub: [u8;32],
                   display_name: String },
    JoinAccept   { state: GroupState },      // full current state, directly
    JoinReject   { reason: String },

    // explicit pull sync
    MembersRequest  { group: GroupId, have_version: u64 },
    MembersResponse { state: Option<GroupState> },  // if newer; else None

    StateRequest  { group: GroupId, state_hash: Hash }, // fetch by announce
    StateResponse { state: Option<GroupState> },

    Poke     { group: GroupId, version: u64 },   // admin nudge on change
    Presence { group: GroupId, seq: u64, ts: u64,
               services: Vec<String>, status: Status }, // heartbeat;
               // Status = Online | Leaving; advisory only
    Goodbye  { group: GroupId },
}
```

### Tunnel streams — one QUIC bidi stream per TCP connection

```
frame 0 (opener):   TunnelOpen  { group: GroupId, service: String }
frame 1 (acceptor): TunnelOk {} | TunnelErr { reason }
then: raw bytes piped 1:1 with the local TCP socket in both directions.
QUIC supplies ordering, flow control, and backpressure. Stream FIN/RESET
maps to TCP FIN/RST.
```

## Validation rules (normative)

**Connection**: the peer's NodeId comes from the QUIC handshake. Unless the
first frame is `JoinRequest`, the peer must be in the local roster of the
referenced group; otherwise the stream is rejected and logged.

**GroupState version V**: `sig` valid for `group_id`; `V == local + 1`
(strictly in sequence — on a jump, fetch the intermediate versions via
`MembersRequest`, or accept a full newer state only if its membership deltas
can be audited: every added entry carries `added_by` with `ManageRoster` at
the prior version and a well-formed `invite_hash`); `blake3(blob) ==
announce.state_hash` when fetched via announce; `key_epoch` present in
keywraps for *this* node (else this node was excluded — surface loudly).

**Announce**: pkarr signature by `group_id` (the DHT/pkarr layer enforces
this); `state_version` monotonic vs. local knowledge; stale announces
(older than local state) are ignored.

**Invite** (checked by admin at redemption *and* re-checkable by anyone):
signature by group key, unexpired, nonce unredeemed; `JoinRequest` must
arrive on a channel authenticated as the `node` it claims.

**Presence**: sender in roster; `seq` monotonic per node; never mutates
membership. Offline after 3 missed heartbeats (POC: 30 s period).

**TunnelOpen** (checked by the destination — the authority for its
services):
1. peer in roster of `group`;
2. peer's role has `Connect`; destination's role has `Expose`;
3. `service` currently declared by the destination;
4. a `service_rules` entry allows (peer, service, this node) — first match
   wins, default **deny**;
5. open-tunnel quota not exceeded (POC: 64 per peer).

Roster/ACL changes re-run checks on open tunnels; failures close streams.
Every rejection is dropped-and-logged with a reason code (`rmesh doctor`).

## Local storage

Encrypted on-disk keystore (identity + dh secret keys) + a small embedded
store (redb or sqlite) for the rest:

| Table | Contents |
|---|---|
| `identity` | node keypair, dh keypair (keystore-backed) |
| `groups` | per group: latest GroupState, group keys (current + previous epoch), group secret key if admin, nonce table (admin) |
| `presence:<group>` | last heartbeat per member (liveness) |
| `forwards` | configured `expose` / `forward` entries (restored on restart) |
| `audit:<group>` | membership changes + rejected traffic (ring buffer) |

## Size budget

`GroupAnnounce` must fit a pkarr packet (~1 KiB → `bootstrap` ≤ 4).
`GroupState` is served over QUIC so it can breathe, but POC caps it at
256 KiB / 256 members (beyond that: state sharding, post-POC). Control
frames ≤ 64 KiB. The daemon rejects, never silently truncates.

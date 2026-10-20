# rmesh POC — Data Spec

Concrete schemas for the POC on Veilid. Encoding is **canonical CBOR**
(serde + `ciborium`, deterministic map ordering) — everything signed must
have a canonical byte form. Hashes BLAKE3, signatures Ed25519, DH X25519,
symmetric crypto XChaCha20-Poly1305 (Veilid VLD0 suite).

> Veilid API specifics (record schemas, watches, size limits) must be
> re-verified against the veilid-core version pinned at implementation
> time; the shapes here are the contract, exact calls may differ.

## Conventions

```rust
type PubKey    = [u8; 32];   // Ed25519 public key
type Sig       = [u8; 64];   // Ed25519 signature
type GroupId   = PubKey;     // group identity public key
type NodeId    = PubKey;     // node (member) identity public key
type RecordKey = ...;        // transport DHT record key (opaque to core)
type Nonce     = [u8; 24];
```

Every signed struct: signature over `blake3(domain_sep ||
canonical_cbor(struct minus sig))`, domain separator per type
(`b"rmesh/roster/v1"`, `b"rmesh/invite/v1"`, …) — prevents cross-context
signature reuse.

## DHT record layout

### Group record (one per group)

Owner: group record owner key (admin). Members read + verify; all subkeys
owner-writable (DFLT schema). Subkeys 1–4 are encrypted with the group key
(non-members and DHT nodes see ciphertext — differentiator #2 in
`06-landscape.md`); subkey 0 is plaintext-but-signed (bootstrap needs it
before key unwrap).

| Subkey | Content | Changes when |
|---|---|---|
| 0 | `GroupHeader` | any version bumps |
| 1 | `Roster` | join / revoke / role change / prune |
| 2 | `AclPolicy` | policy change |
| 3 | `RouteBook` | members rotate contact routes |
| 4 | `KeyWrapSet` | every group-key rotation |

```rust
struct GroupHeader {
    v: u8,                    // schema version = 1
    group_id: GroupId,
    name: String,             // display only, ≤ 64 bytes
    created_at: u64,          // unix seconds
    admin: NodeId,            // POC: single admin
    roster_version: u64,      // cheap change detection for watchers
    acl_version: u64,
    key_epoch: u64,           // current group-key epoch
    sig: Sig,                 // group identity key
}

struct Roster {
    v: u8,
    group_id: GroupId,
    version: u64,             // monotonic, +1 steps only
    members: Vec<MemberEntry>,
    sig: Sig,                 // group identity key
}

struct MemberEntry {
    node: NodeId,
    dh_pub: [u8; 32],         // X25519, for key-wrapping + tunnel keys
    role: Role,               // Admin | Member | Observer
    presence: RecordKey,      // this member's presence record
    added_at: u64,
    added_by: NodeId,
    invite_hash: [u8; 32],    // blake3 of redeemed invite (audit trail)
}

struct AclPolicy {
    v: u8,
    group_id: GroupId,
    version: u64,
    role_caps: Vec<(Role, Vec<Capability>)>,
    service_rules: Vec<ServiceRule>,
    sig: Sig,
}
// Capability = Invite | ManageRoster | ManageAcl | Connect | Expose
// POC defaults: Admin → all; Member → Connect, Expose; Observer → (none)
struct ServiceRule {
    // "who may connect to which service on which node"; first match wins,
    // default deny. POC keeps this flat and simple.
    service: String,          // service name or "*"
    on: NodeSel,              // Any | Node(NodeId)
    allow: RoleSel,           // Role(Role) | Node(NodeId)
}

struct RouteBook {
    v: u8,
    version: u64,
    routes: Vec<RouteEntry>,
    sig: Sig,                 // group identity key (admin republishes)
}
struct RouteEntry {
    node: NodeId,
    route_blob: Vec<u8>,      // transport contact blob (Veilid private route)
    published_at: u64,
    sig: Sig,                 // by the member — admin relays, cannot forge
}

struct KeyWrapSet {
    v: u8,
    group_id: GroupId,
    key_epoch: u64,
    wraps: Vec<KeyWrap>,      // one per current member
    sig: Sig,
}
struct KeyWrap {
    node: NodeId,
    ephemeral_pub: [u8; 32],  // sealed-box: XChaCha20(group_key),
    nonce: Nonce,             //   key = X25519(eph_sk, member.dh_pub)
    ct: Vec<u8>,
}
```

### Presence record (one per member per group)

Owner: the member. Heartbeat + self-declared state; encrypted with the
group key. **Advisory only** — never a membership source of truth.

```rust
struct Presence {
    v: u8,
    group_id: GroupId,
    node: NodeId,
    seq: u64,                 // bumps each heartbeat
    ts: u64,                  // heartbeat time; peers mark offline after
                              //   OFFLINE_AFTER = 3 × HEARTBEAT_PERIOD
    route_blob: Vec<u8>,      // current contact route (fast path; RouteBook
                              //   is the fallback)
    services: Vec<Service>,   // what this node exposes
    status: Status,           // Online | Leaving
    sig: Sig,                 // by the member
}
struct Service {
    name: String,             // "postgres"
    // bind target stays LOCAL to the exposing node and is NOT published;
    // publishing only the name keeps localhost topology private.
}
// POC: HEARTBEAT_PERIOD = 30s
```

## Invitation token

```rust
struct Invite {
    v: u8,
    group_id: GroupId,
    group_record: RecordKey,  // where the group record lives
    admin_route: Vec<u8>,     // contact blob for redemption call
    role: Role,               // offered role
    nonce: Nonce,             // single-use
    expires_at: u64,
    sig: Sig,                 // group identity key = the voucher
}
```

Wire form: `rmesh://invite/` + base64url(canonical CBOR). The admin keeps a
nonce table; redemption consumes the nonce. Every member can independently
verify a redeemed invite via `invite_hash` in the roster entry.

## RPC envelopes

All direct node↔node communication (AppCall for request/response,
AppMessage for fire-and-forget) uses one envelope:

```rust
struct Envelope {
    v: u8,
    group_id: GroupId,
    from: NodeId,
    ts: u64,                  // reject if |now - ts| > 120s (replay bound)
    body: Body,
    sig: Sig,                 // by `from`, domain-separated per body kind
}

enum Body {
    // ---- membership plane ----
    // the ONLY body accepted from a non-member; validated by invite voucher
    JoinRequest  { invite: Invite, node: NodeId, dh_pub: [u8;32],
                   presence: RecordKey, display_name: String },
    JoinAccept   { roster_version: u64 },  // roster+keywrap now in the DHT
    JoinReject   { reason: String },

    // explicit pull sync: "who's in this group now?"
    MembersRequest  { have_roster: u64, have_acl: u64 },
    MembersResponse { roster: Option<Roster>,        // full, signed, if newer
                      acl: Option<AclPolicy>,
                      key_epoch: u64 },

    Goodbye { },                            // voluntary leave announcement

    // ---- data plane: tunnels ----
    TunnelOpen  { tunnel: u64,              // caller-chosen id, unique per pair
                  service: String,          // must be declared + ACL-allowed
                  eph_pub: [u8;32] },       // caller X25519 for session key
    TunnelAccept{ tunnel: u64, eph_pub: [u8;32] },
    TunnelReject{ tunnel: u64, reason: TunnelError },
    TunnelData  { tunnel: u64, seq: u64, ct: Vec<u8> },  // ≤ 16 KiB plaintext
    TunnelAck   { tunnel: u64, upto: u64 },  // window/backpressure
    TunnelClose { tunnel: u64, seq: u64 },   // after last data seq
}
```

Tunnel session key: `blake3_kdf("rmesh/tunnel/v1", ecdh(eph_a, eph_b),
group_id, tunnel_id)`; `TunnelData.ct` is XChaCha20-Poly1305 with the seq
in the nonce — content is private even from other group members. Window:
send up to 32 unacked chunks, then wait for `TunnelAck`. (On an Iroh
transport this whole framing collapses to a native QUIC stream; it lives in
the adapter for that reason.)

## Validation rules (normative)

**Envelope**: signature valid for `from` with per-kind domain separator,
and `ts` within the replay bound. Then, unless the body is `JoinRequest`,
`from` must be in the local roster.

**Roster version V**: signed by the group identity key, `V == local + 1`
(apply strictly in sequence; pull the gap via `MembersRequest` on a jump),
and deltas comply with the ACL at `V − 1` (`ManageRoster`).

**AclPolicy / RouteBook / KeyWrapSet**: signed by the group identity key,
version monotonic; `RouteEntry.sig` must also verify against the member's
own key.

**Presence**: signed by its owner, `seq` monotonic per node, owner must be
in the roster. Never mutates the roster.

**MembersResponse**: only signed artifacts are accepted (`roster`, `acl`
are whole signed structs) — a peer can withhold but cannot fabricate.

**TunnelOpen** (checked by the destination — the authority for its
services):
1. envelope valid, caller in roster;
2. caller's role has `Connect`; destination's role has `Expose`;
3. `service` is currently declared by the destination;
4. a `service_rules` entry allows (caller, service, destination) — first
   match wins, default **deny**;
5. per-pair tunnel id unused; open-tunnel quota not exceeded (POC: 64).

**TunnelData**: known tunnel, in-window `seq`, AEAD opens with the session
key. Roster/ACL changes re-run checks on open tunnels; failures close them.
Anything failing any rule is dropped and logged with a reason code —
rejected traffic must be observable (`rmesh doctor`).

## Local storage (Veilid TableStore; keys in ProtectedStore)

| Table | Contents |
|---|---|
| `identity` | node keypair, dh keypair |
| `groups` | per group: header, roster, ACL, route book, group keys (current + previous epoch), group secret key if admin, nonce table (admin) |
| `presence:<group>` | last seen presence per member (liveness/timeout) |
| `forwards` | configured `expose` / `forward` entries (restored on restart) |
| `audit:<group>` | membership changes + rejected-traffic log (ring buffer) |

## Size budget

DHT subkey values are limited (~32 KiB): roster ≤ 256 members for the POC
(then it must shard — post-POC); presence ≤ 4 KiB; `TunnelData` plaintext
≤ 16 KiB per chunk. The daemon rejects, never silently truncates.

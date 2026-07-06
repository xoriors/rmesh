# Telescale POC — Data Spec

Concrete schemas for the POC. Encoding is **canonical CBOR** (serde +
`ciborium`, deterministic map ordering) — everything signed must have a
canonical byte form. All hashes BLAKE3, all signatures Ed25519, all DH X25519,
all symmetric crypto XChaCha20-Poly1305 (Veilid VLD0 suite).

> Veilid API names below (record schemas, watches) should be re-verified
> against the veilid-core version pinned at implementation time; the shapes
> here are the contract, the exact calls may differ slightly.

## Conventions

```rust
type PubKey    = [u8; 32];   // Ed25519 public key
type Sig       = [u8; 64];   // Ed25519 signature
type GroupId   = PubKey;     // group identity public key
type MemberId  = PubKey;     // member identity public key
type RecordKey = ...;        // Veilid DHT record key (TypedKey)
type Nonce     = [u8; 24];
```

Every signed struct follows the same pattern: the signature covers
`canonical_cbor(struct with sig field omitted)` prefixed by a domain
separator, e.g. `b"telescale/roster/v1"`. Domain separators prevent
cross-context signature reuse.

## DHT record layout

### Group record (one per group)

Owner: group record owner key (admin). Members have read access only —
all subkeys are owner-writable, DFLT schema.

| Subkey | Content | Notes |
|---|---|---|
| 0 | `GroupHeader` | rarely changes |
| 1 | `Roster` | bumped on join/revoke/role change |
| 2 | `AclPolicy` | bumped on policy change |
| 3 | `RouteBook` | refreshed as members rotate private routes |
| 4 | `KeyWrapSet` | re-published on every data-key rotation |

```rust
struct GroupHeader {
    v: u8,                    // schema version = 1
    group_id: GroupId,
    name: String,             // display only, ≤ 64 bytes
    created_at: u64,          // unix seconds
    admin: MemberId,          // POC: single admin
    roster_version: u64,      // current versions, for cheap change detection
    acl_version: u64,
    key_epoch: u64,           // current data-key epoch
    sig: Sig,                 // by group identity key
}

struct Roster {
    v: u8,
    group_id: GroupId,
    version: u64,             // monotonic; must increase by exactly 1
    members: Vec<MemberEntry>,
    sig: Sig,                 // by group identity key
}

struct MemberEntry {
    member: MemberId,
    dh_pub: [u8; 32],         // X25519, for key-wrapping
    role: Role,               // Admin | Member | Observer
    outbox: RecordKey,        // this member's outbox DHT record
    added_at: u64,
    added_by: MemberId,
    invite_hash: [u8; 32],    // blake3 of the redeemed invite (audit)
}

struct AclPolicy {
    v: u8,
    group_id: GroupId,
    version: u64,
    rules: Vec<(Role, Vec<Capability>)>,
    sig: Sig,
}
// Capability = Invite | Write | Read | ManageRoster | ManageAcl
// POC defaults:
//   Admin    → all
//   Member   → Read, Write
//   Observer → Read

struct RouteBook {
    v: u8,
    version: u64,
    routes: Vec<RouteEntry>,
    sig: Sig,                 // by group identity key (admin republishes)
}
struct RouteEntry {
    member: MemberId,
    route_blob: Vec<u8>,      // Veilid private route blob
    published_at: u64,
    sig: Sig,                 // by the member (admin can't forge routes)
}

struct KeyWrapSet {
    v: u8,
    group_id: GroupId,
    key_epoch: u64,           // bumped on every rotation
    wraps: Vec<KeyWrap>,
    sig: Sig,
}
struct KeyWrap {
    member: MemberId,
    // sealed-box style: XChaCha20-Poly1305( group_data_key ),
    // key = X25519(ephemeral_sk, member.dh_pub), ephemeral pub included
    ephemeral_pub: [u8; 32],
    nonce: Nonce,
    ct: Vec<u8>,
}
```

### Member outbox record (one per member per group)

Owner: the member. Append-only log of that member's updates.

| Subkey | Content |
|---|---|
| 0 | `OutboxHead { v, member, group_id, seq: u64, sig }` |
| 1..N | `Update` entries, ring-buffered over subkeys (`subkey = 1 + (seq % N)`) |

The ring buffer bounds DHT usage; peers that fall more than N behind
reconcile via direct `SyncRequest` instead (see `04-protocols.md`).
POC: N = 64.

```rust
struct Update {
    v: u8,
    group_id: GroupId,
    author: MemberId,
    seq: u64,                 // per-author, starts at 1, no gaps
    prev_hash: [u8; 32],      // blake3 of author's previous Update (hash chain)
    roster_version: u64,      // roster the author held when writing
    key_epoch: u64,           // data-key epoch used for payload
    ts: u64,
    payload: EncPayload,      // encrypted with group data key
    sig: Sig,                 // by author, over everything above
}

struct EncPayload { nonce: Nonce, ct: Vec<u8> }

// plaintext inside EncPayload:
enum Payload {
    Chat { text: String },                     // demo app 1
    KvPut { key: String, value: Vec<u8> },     // demo app 2
    KvDel { key: String },
    Tombstone,                                 // member leaving
}
```

## Invite token

```rust
struct Invite {
    v: u8,
    group_id: GroupId,
    group_record: RecordKey,  // where to find the group record
    admin_route: Vec<u8>,     // private route blob for redemption AppCall
    role: Role,               // role being offered
    nonce: Nonce,             // single-use
    expires_at: u64,
    sig: Sig,                 // by group identity key
}
```

Wire form: `telescale://invite/` + base64url(canonical CBOR). Admin keeps
a local table of unredeemed nonces; redemption consumes the nonce.

## RPC envelopes (AppCall / AppMessage)

```rust
struct Envelope {
    v: u8,
    group_id: GroupId,
    from: MemberId,
    ts: u64,
    body: Body,
    sig: Sig,                 // by `from`, domain-separated per body kind
}

enum Body {
    // join (invitee → admin); the ONLY unauthenticated-sender call —
    // validated by invite possession instead of roster membership
    JoinRequest { invite: Invite, member: MemberId, dh_pub: [u8;32],
                  outbox: RecordKey, display_name: String },
    JoinAccept  { roster_version: u64 },   // roster + keywrap now in DHT
    JoinReject  { reason: String },

    // direct sync (member ↔ member)
    SyncRequest  { have: Vec<(MemberId, u64)> },          // version vector
    SyncResponse { updates: Vec<Update>,                  // the gap
                   roster_version: u64, acl_version: u64 },

    // notification nudge (fire-and-forget AppMessage)
    Poke { outbox_seq: u64 },
}
```

## Validation rules (normative)

An `Update` is **valid** iff, in order:

1. `sig` verifies for `author` with the correct domain separator;
2. `author` ∈ roster at `roster_version`, and `roster_version` ≤ local
   current roster version (updates referencing a *future* roster are held in
   a pending queue until the roster syncs, then re-validated);
3. the ACL at the matching version grants `author`'s role the `Write`
   capability (Tombstone requires only membership);
4. `seq` = local `last_seen[author] + 1` and `prev_hash` matches the stored
   hash of the author's previous update (gap or fork ⇒ hold and reconcile
   via SyncRequest; a *fork* — two valid updates with same `seq` — is flagged
   as author equivocation and surfaced to the user);
5. `payload` decrypts under the data key for `key_epoch` (members keep the
   current + previous epoch keys to tolerate rotation races; anything older
   is rejected).

A `Roster` version V is **valid** iff signed by the group identity key,
`V == local_version + 1`, and membership deltas comply with the ACL at
`V - 1` (`ManageRoster`). Roster versions apply strictly in sequence.

A `JoinRequest` is **valid** iff the invite signature verifies, the invite is
unexpired, the nonce is unredeemed, and the envelope `sig` verifies for the
candidate `member` key (proves key possession).

## Local storage (TableStore)

| Table | Contents |
|---|---|
| `identity` | member keypair, dh keypair (ProtectedStore-backed) |
| `groups` | per group: header, roster, ACL, route book, key epochs (current+prev) |
| `log:<group>` | applied updates (full history, not ring-buffered locally) |
| `vv:<group>` | version vector `member → last_seen seq` + per-author head hashes |
| `pending:<group>` | held updates awaiting roster sync / gap fill |
| `invites:<group>` | issued invites: nonce → status (admin only) |

## Size budget

DHT subkey values are limited (~32 KiB); the POC enforces:
`Update` ≤ 16 KiB (payloads chunk at the app layer later — POC just rejects
larger), roster ≤ 256 members (beyond that the roster moves to its own
multi-subkey record — post-POC).

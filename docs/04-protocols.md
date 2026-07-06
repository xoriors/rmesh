# Telescale POC — Protocol Flows

Message and record schemas are in `03-poc-spec.md`. All flows below run over
Veilid: DHT operations for shared state, AppCall/AppMessage over private
routes for direct interaction.

## 1. Group creation

```mermaid
sequenceDiagram
    participant A as Admin (creator)
    participant D as Veilid DHT
    A->>A: generate group identity keypair (group_id)
    A->>A: generate record owner keypair
    A->>A: generate data key, epoch = 1
    A->>D: create group record (subkeys 0..4)
    A->>D: create own outbox record
    A->>D: write GroupHeader, Roster v1 (self as Admin),<br/>default AclPolicy v1, RouteBook, KeyWrapSet(epoch 1)
```

Post-condition: a group of one. The group is *only* discoverable by knowing
the group record key — which travels inside invites.

## 2. Invitation and join

```mermaid
sequenceDiagram
    participant A as Admin
    participant I as Invitee
    participant D as Veilid DHT
    A->>A: mint Invite {nonce, expiry, role, admin_route}, sign
    A-->>I: out-of-band: telescale://invite/… (link / QR)
    I->>I: generate identity + dh keypairs
    I->>D: create own outbox record
    I->>A: AppCall JoinRequest {invite, member, dh_pub, outbox}
    A->>A: validate invite (sig, expiry, nonce unredeemed),<br/>envelope sig proves key possession
    A->>D: Roster v+1 (append member), re-sign
    A->>D: KeyWrapSet: wrap current data key for new member set
    A->>D: RouteBook + GroupHeader updated
    A->>I: AppCall reply JoinAccept {roster_version}
    I->>D: fetch group record, verify all sigs
    I->>I: unwrap data key, start watches, initial sync
```

Failure paths: expired/duplicate nonce → `JoinReject`; admin unreachable →
invitee retries while invite valid (route blob in the invite may go stale —
invitee falls back to fetching the admin's current route from the RouteBook
subkey, which is readable once they know the record key).

## 3. Explicit sync

Two complementary paths; both end in the same validation pipeline
(`03-poc-spec.md` § validation rules).

### 3a. DHT watch path (background convergence)

```mermaid
sequenceDiagram
    participant M as Member
    participant D as Veilid DHT
    M->>D: watch group record + all member outbox heads
    D-->>M: value-changed (outbox head: seq bumped)
    M->>D: read Update subkeys last_seen+1 ..= seq
    M->>M: validate chain → apply → bump version vector
    D-->>M: value-changed (group record: roster/acl/keywrap)
    M->>M: fetch, verify sigs, apply in version order
```

### 3b. Direct reconcile path (fast / gap recovery)

Used on startup, when the watch path reports a gap larger than the outbox
ring buffer (N=64), or opportunistically after sending.

```mermaid
sequenceDiagram
    participant M1 as Member 1
    participant M2 as Member 2
    M1->>M2: AppCall SyncRequest {have: version vector}
    M2->>M2: diff against own log
    M2->>M1: SyncResponse {updates gap, roster/acl versions}
    M1->>M1: validate each update (same pipeline), apply
    Note over M1,M2: if M2's roster/acl versions are newer,<br/>M1 refreshes from DHT before applying held updates
```

Sender nudge: after writing an update, a member fire-and-forgets a `Poke`
AppMessage to members with fresh RouteBook entries — cuts perceived latency
below DHT watch latency, but is purely an optimization; correctness never
depends on pokes arriving.

## 4. Sending application data

1. Author builds `Payload` (Chat/KvPut/…), encrypts with current data key →
   `EncPayload`.
2. Wraps in `Update {seq = own_seq+1, prev_hash, roster_version, key_epoch}`,
   signs.
3. Writes to own outbox subkey `1 + (seq % 64)`, then updates `OutboxHead`.
4. Sends `Poke` to reachable members.

Receivers validate + apply per the normative rules. The demo CLI renders
`Chat` as a chat timeline and `KvPut/KvDel` as a live key-value document.

## 5. Revocation and key rotation

```mermaid
sequenceDiagram
    participant A as Admin
    participant D as Veilid DHT
    participant R as Revoked member
    participant M as Remaining members
    A->>D: Roster v+1 (remove R), re-sign
    A->>A: generate data key epoch e+1
    A->>D: KeyWrapSet(e+1) wrapped only for remaining members
    A->>D: GroupHeader {roster_version+1, key_epoch e+1}
    D-->>M: watch fires → fetch, verify, unwrap new key
    Note over M: new updates use epoch e+1
    Note over R: R still holds epoch ≤ e keys (past content)<br/>but cannot decrypt epoch e+1 → forward exclusion
    Note over M: updates from R at roster ≥ v+1 fail validation<br/>(author not in roster) and are rejected by every member
```

Rotation race handling: members accept payloads at epoch `e` for a grace
window (current + previous epoch) so in-flight updates aren't lost; anything
older is rejected.

Voluntary leave: member writes `Tombstone` as their final update; admin then
runs the same revocation flow.

## 6. Route refresh

Private routes are ephemeral — Veilid can drop/rotate them.

1. Member detects its route died (Veilid callback) → allocates a new one.
2. Member sends signed `RouteEntry` to admin (AppCall; falls back to writing
   a poke via any reachable member if the admin route is also stale).
3. Admin republishes `RouteBook` v+1.

Because `RouteEntry` is signed by the member, the admin relays but cannot
forge routes (prevents an admin silently MITM-ing member↔member traffic by
substituting routes it controls).

## 7. Startup sequence (every node)

1. Init veilid-core, attach to network (public bootstrap or configured).
2. Load identity + group state from TableStore.
3. For each group: fetch `GroupHeader`; if versions moved, sync roster/ACL/
   keywrap first (state plane before data plane).
4. Install DHT watches; run one 3b reconcile against any reachable member.
5. Allocate/refresh private route; publish `RouteEntry` if changed.

## Threat notes per flow

| Flow | Main threat | Countermeasure |
|---|---|---|
| Invite | token theft/replay | single-use nonce, expiry, key-possession proof in JoinRequest |
| Join | fake admin | invite + roster + keywrap all signed by group identity key the invitee got out-of-band inside the invite |
| Sync | forged/withheld updates | signatures + hash chain + per-author seq; withholding degrades to availability, detected by cross-peer version-vector diffs |
| Send | unauthorized writer | ACL check against roster at declared version, on every member |
| Revocation | revoked member re-injects | roster check fails at v+1; new epoch key unavailable |
| Routing | admin substitutes routes | RouteEntry signed by the member itself |

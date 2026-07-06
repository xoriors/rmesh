# Telescale — Roadmap

Phases are ordered so that every phase ends in something runnable and
demoable. Security-critical logic (proto/core) is built and tested before the
network is involved.

## Phase 0 — Foundations

**Goal: a node that attaches to Veilid; workspace skeleton in place.**

- [ ] Cargo workspace: `telescale-proto`, `telescale-core`,
      `telescale-veilid`, `rmesh-cli`.
- [ ] Pin veilid-core; minimal node: init, attach, report network state
      (`rmesh node status`).
- [ ] Identity: generate/load member keypair (ProtectedStore),
      `rmesh id show`.
- [ ] CI: fmt, clippy, test; no network needed in tests.

**Demo:** `rmesh node status` shows an attached Veilid node with our identity.

## Phase 1 — Groups (single member)

**Goal: create a group and read it back — schemas proven on the real DHT.**

- [ ] `telescale-proto`: all `03-poc-spec.md` structs, canonical CBOR,
      domain-separated sign/verify. Property tests: encode/decode roundtrip,
      signature tamper detection.
- [ ] `telescale-veilid`: DHT record create/read/write wrappers.
- [ ] `rmesh group create <name>`, `rmesh group show` (fetch + verify from
      DHT, even for our own group — validation path exercised from day one).

**Demo:** create a group on machine A; `group show` on machine A after
restart (state from TableStore + DHT).

## Phase 2 — Invites and join

**Goal: two real machines, one group.**

- [ ] Invite mint/encode/parse (`rmesh invite create --role member --ttl 24h`).
- [ ] Private routes + AppCall plumbing in `telescale-veilid`.
- [ ] JoinRequest/Accept/Reject state machine (admin side + invitee side).
- [ ] Roster v+1 publish, KeyWrapSet publish, invitee verify-and-unwrap.
- [ ] Negative tests: expired invite, replayed nonce, tampered invite,
      stolen invite without key possession.

**Demo:** machine B joins via QR/link from machine A; `rmesh group members`
agrees on both.

## Phase 3 — Sync engine + demo apps

**Goal: the "explicit sync for updates" pillar, end to end.**

- [ ] Outbox writer: seq, hash chain, encryption, ring buffer.
- [ ] Validation pipeline exactly per spec (pure, in `telescale-core`,
      fuzz/property tested with an in-memory fake network).
- [ ] DHT watch path + version vectors + pending queue.
- [ ] Direct reconcile (SyncRequest/Response) + Poke nudges.
- [ ] Demo apps: `rmesh chat <group>` (TUI-lite) and `rmesh kv get/put/del`.

**Demo:** 3 nodes chatting; kill one node mid-conversation, restart it, watch
it catch up via reconcile.

## Phase 4 — ACL enforcement + revocation

**Goal: the "routing and ACL layer" pillar; membership is a lifecycle.**

- [ ] AclPolicy application: roles/capabilities on every inbound update;
      `rmesh acl show/set-role`.
- [ ] Observer role proves read-only enforcement (its writes rejected by all).
- [ ] Revocation: roster removal + key epoch rotation + grace window.
- [ ] Route refresh flow (signed RouteEntry, RouteBook republish).
- [ ] Adversarial tests: revoked member replays old updates, writes at stale
      roster version, attempts epoch e+1 decrypt; equivocation (seq fork)
      detection surfaced in CLI.

**Demo:** revoke machine B live; show its messages stop being accepted and it
can't read new ones.

## Phase 5 — POC hardening & demo polish

**Goal: something you can hand to a stranger with a README.**

- [ ] `rmesh doctor` (network reachability, watch health, version vectors).
- [ ] Startup sequence resilience: offline start, DHT slow paths, route decay.
- [ ] Structured logging of every validation rejection (the observability of
      "explicit sync").
- [ ] Docs: quickstart, demo script, threat model recap.
- [ ] Tag `v0.1.0-poc`.

**Exit criteria for the POC:** 3+ nodes on different networks (NAT'd) create
a group, invite, chat, sync a kv doc, revoke a member — with no
infrastructure of ours running anywhere.

## Post-POC — toward Telescale proper

Not scheduled; recorded so POC decisions don't paint us out of them.

1. **Multi-device identity** — user key signs device keys; roster entries
   become users with device sets.
2. **Admin quorum** — group identity key → threshold signatures (FROST) so
   membership isn't a single admin's power.
3. **Large payloads / files** — chunking + content-addressed blocks
   (Veilid block store when available), maybe rencfs-style encrypted trees.
4. **Overlay IP mode** — the Tailscale-shaped feature: TUN device, group
   members get stable overlay addresses, ACLs become packet policy. This is
   where "Telescale" earns its name; the membership/ACL/sync core built in
   the POC is exactly the control plane it needs.
5. **Mobile** — veilid-flutter bindings over `telescale-core`.
6. **Private Veilid subnet option** — self-hosted bootstrap for fully
   air-gapped deployments.
7. **Stronger crypto lifecycle** — per-sender ratchets / MLS-style tree keys
   if group sizes or PCS requirements grow.

## Open questions (to settle during Phases 0–2)

- **Watch limits:** how many DHT watches does veilid-core sustain per node in
  practice? May force head-polling fallback for big groups.
- **Route churn rate** in the wild → RouteBook republish frequency and
  whether the admin becomes a bottleneck.
- **DHT record TTL/persistence semantics** — do group records need periodic
  re-publish keepalives? (Affects offline-group recovery.)
- **veilid-core API stability** — pin and vendor-note the version; the
  adapter crate isolates the churn.

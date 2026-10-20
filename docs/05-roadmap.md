# rmesh — Roadmap

Phases ordered so every phase ends runnable and demoable; security-critical
logic (proto/core) is built and tested before the network is involved. The
POC implements the brainstorm decisions as amended (`00-brainstorm-
decisions.md`): **Veilid POC, iroh + mainline DHT as the planned migration
option**, container-native positioning, and the unprivileged
`rmesh forward` demo as the proof point.

## Phase 0 — Foundations

**Goal: a daemon that attaches to Veilid; workspace skeleton in place.**

- [ ] Cargo workspace: `rmesh-proto`, `rmesh-core`, `rmesh-veilid`,
      `rmesh-cli`.
- [ ] Pin veilid-core; minimal node: init, attach, report network state
      (`rmesh node status`).
- [ ] Identity: generate/load node keypair + X25519 pair (ProtectedStore);
      `rmesh id show`.
- [ ] `Transport` trait in `rmesh-core` + in-memory fake for tests; the
      Veilid impl in `rmesh-veilid` (the iroh-swap seam from day one).
- [ ] Daemon/CLI split over a unix socket; systemd *user* unit + container
      example (unprivileged is the point — prove it immediately).
- [ ] CI: fmt, clippy, test — no network needed in tests.

**Demo:** `rmesh node status` shows an attached Veilid node with our
identity.

## Phase 1 — Groups (single member)

**Goal: create a group and read it back — schemas proven on the real DHT.**

- [ ] `rmesh-proto`: all `03-poc-spec.md` structs, canonical CBOR,
      domain-separated sign/verify; property tests (roundtrip, tamper).
- [ ] Group key generation, key-wrapping, encrypted DHT subkeys.
- [ ] `rmesh group create <name>` / `rmesh group show` — show fetches from
      the DHT and verifies signatures even for our own group, so the
      validation path is exercised from day one.
- [ ] Presence record + heartbeat loop.

**Demo:** create a group on machine A; restart the daemon; `group show`
rebuilds verified state from TableStore + DHT.

## Phase 2 — Invitations and join

**Goal: two real machines, one group — invitation-based membership.**

- [ ] Invite mint/encode/parse (`rmesh invite create --role member --ttl 24h`).
- [ ] Private-route plumbing + envelope RPC in `rmesh-veilid`.
- [ ] JoinRequest/Accept/Reject state machines (admin + joiner sides).
- [ ] Roster v+1 + KeyWrapSet publish; joiner verify-and-unwrap.
- [ ] Negative tests: expired invite, replayed nonce, tampered invite,
      stolen invite without key possession.

**Demo:** machine B joins via a pasted `rmesh://invite/…` from machine A;
`rmesh group members` agrees on both.

## Phase 3 — Explicit membership sync + liveness

**Goal: the "explicit sync for updates" pillar.**

- [ ] DHT watch on the group record; version-ordered fetch-verify-apply.
- [ ] `MembersRequest`/`MembersResponse` direct pull (startup, periodic,
      gap).
- [ ] Presence watches; offline at 3 missed heartbeats; `Goodbye` + admin
      prune; rejoin resync.
- [ ] Pending queue for updates referencing a future roster version.
- [ ] `MembershipSync` behind a trait (the future gossip seam).
- [ ] Chaos tests on the in-memory fake transport: partitions, stale
      peers, withholding peers — state must converge and never accept
      unsigned data.

**Demo:** 3 nodes; kill one mid-session, watch peers mark it offline;
restart it and watch it resync and come back online.

## Phase 4 — Routing + ACL + the port-forward data plane

**Goal: the product moment — `psql` across clouds, no VPN, no root.**

- [ ] `rmesh expose add <name> <host:port>` (declared in presence).
- [ ] ACL: role capabilities + service rules, default deny;
      `rmesh acl show/allow`.
- [ ] Tunnel protocol: open/accept/data/ack/close, chunking, 32-chunk
      window, per-tunnel session keys; reliable-stream shim in the adapter.
- [ ] `rmesh forward --to <member>:<service> --listen <addr>`.
- [ ] Destination-side authoritative enforcement + observable rejects.
- [ ] Revocation: roster removal + key epoch rotation + live-tunnel
      teardown.
- [ ] Adversarial tests: non-member opens tunnel, member without Connect,
      undeclared service, rule-denied pair, revoked node replaying old
      routes, observer trying anything.
- [ ] Baseline latency/throughput over private routes vs. Veilid
      direct/safety settings — the numbers that inform the iroh-migration
      decision and the per-group privacy/speed knob.

**Demo:** Postgres on a bare-metal box, `psql` from an AWS instance and
from a *rootless container* through the mesh; revoke the container's node
live and watch the tunnel drop and reconnects get denied.

## Phase 5 — POC hardening & demo polish

**Goal: something you can hand to a stranger with a README.**

- [ ] `rmesh doctor`: attachment state, watch health, roster/ACL versions,
      presence table, recent rejects with reasons.
- [ ] Resilience: offline start, DHT slow paths, route decay,
      admin-offline behavior documented (joins/revocations queue until the
      admin returns).
- [ ] Structured audit log of membership changes + rejected traffic.
- [ ] Docs: quickstart, demo script, threat-model recap; positioning
      one-pager aligned with `06-landscape.md`.
- [ ] Tag `v0.1.0-poc`.

**POC exit criteria:** 3+ nodes on different networks (NAT'd, mixed cloud +
bare metal + one rootless container) create a group, invite, sync
membership, forward real TCP services under ACL, and revoke a member —
with zero infrastructure of ours running anywhere, **zero sudo anywhere**,
and measured tunnel performance numbers in hand.

## Post-POC directions

Recorded so POC decisions don't paint us out of them; unscheduled.

1. **Gossip membership propagation** — swap the explicit-sync impl behind
   `MembershipSync` for an epidemic protocol (on an iroh transport:
   `iroh-gossip` per-group topics); validation pipeline unchanged.
2. **Iroh + mainline DHT transport** (the recorded migration option) —
   implement `rmesh-iroh` against the `Transport` trait: the tunnel shim
   collapses to native QUIC streams, discovery rides pkarr signed records,
   and the encrypted-state model moves to an announce + member-served
   blob. Compare hole-punch success and throughput against the Phase 4
   baselines, then choose per deployment — noting the trade: iroh gives
   speed and routing control, Veilid gives route privacy (differentiator
   #5), so this may end as a per-group transport policy rather than a
   wholesale swap.
3. **Custom path selection** — latency-, bandwidth-, reputation-aware
   routing; the reason iroh stays on the table.
4. **Admin quorum** — group identity key → threshold signatures (FROST);
   removes the single-admin trust concentration and doubles down on
   differentiator #1.
5. **Overlay IP mode (TUN)** — *optional* privileged mode for
   whole-network use cases (where Rayfish lives today); port forwarding
   stays the container-native default.
6. **Multi-device identity** — user key signs device keys.
7. **Web dashboard** (chat decision #8: CLI first) and **mobile** (Veilid
   is mobile-first — a reason it keeps its seat).
8. **UDP forwarding**, large-transfer optimization, self-hosted bootstrap
   for air-gapped deployments.

## Open questions (to settle during Phases 0–2)

- veilid-core watch capacity per node (may force header polling for big
  groups); DHT record TTL/keepalive semantics for offline groups.
- Realistic AppCall throughput/latency over private routes vs.
  safety-route settings — how usable are interactive tunnels (ssh) in
  practice?
- Whether tunnels should default to private routes (max privacy) or allow
  direct-connection mode inside a group (max speed) — likely a per-group
  policy knob.
- veilid-core API stability — pin the version; the adapter isolates churn.

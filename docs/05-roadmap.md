# rmesh — Roadmap

Phases ordered so every phase ends runnable and demoable; security-critical
logic (proto/core) is built and tested before the network is involved. The
POC implements the brainstorm decisions as amended by the 2026-10 landscape
review (`00-brainstorm-decisions.md`): **iroh directly, no Veilid detour**,
and the unprivileged `rmesh forward` demo as the proof point.

## Phase 0 — Foundations

**Goal: a daemon that comes up on iroh; workspace skeleton in place.**

- [ ] Cargo workspace: `rmesh-proto`, `rmesh-core`, `rmesh-iroh`,
      `rmesh-cli`.
- [ ] Pin iroh; minimal endpoint: come up, resolve/publish discovery,
      report state (`rmesh node status`).
- [ ] Identity: generate/load node keypair (= EndpointId) + X25519 pair in
      the encrypted keystore; `rmesh id show`.
- [ ] `Transport` trait in `rmesh-core` + in-memory fake for tests; iroh
      impl in `rmesh-iroh`.
- [ ] Daemon/CLI split over a unix socket; systemd *user* unit + container
      example (unprivileged is the point — prove it immediately).
- [ ] CI: fmt, clippy, test — no network needed in tests.

**Demo:** `rmesh node status` on two machines shows live endpoints that can
dial each other by key (direct or via relay).

## Phase 1 — Groups (single member)

**Goal: create a group; announce + encrypted state proven on the real DHT.**

- [ ] `rmesh-proto`: all `03-poc-spec.md` structs, canonical CBOR,
      domain-separated sign/verify; property tests (roundtrip, tamper).
- [ ] Group key generation, key-wrapping, state encrypt/decrypt.
- [ ] pkarr announce publish/resolve (+ 60-min republish timer).
- [ ] `rmesh group create <name>` / `rmesh group show` — show re-resolves
      the announce and re-verifies the state even for our own group, so the
      validation path is exercised from day one.

**Demo:** create a group on machine A; restart the daemon; `group show`
rebuilds verified state from store + announce.

## Phase 2 — Invitations and join

**Goal: two real machines, one group — invitation-based membership.**

- [ ] Invite mint/encode/parse (`rmesh invite create --role member --ttl 24h`).
- [ ] Control-stream RPC (ALPN `rmesh/0`, channel-bound peer identity).
- [ ] JoinRequest/Accept/Reject state machines (admin + joiner sides);
      state v+1 with re-wrap; joiner verify-and-unwrap; announce republish
      + `Poke`.
- [ ] Negative tests: expired invite, replayed nonce, tampered invite,
      stolen invite on a channel not matching the claimed node.

**Demo:** machine B joins via a pasted `rmesh://invite/…` from machine A;
`rmesh group members` agrees on both.

## Phase 3 — Explicit membership sync + liveness

**Goal: the "explicit sync for updates" pillar.**

- [ ] Announce poll/refresh; `StateRequest` by hash; `MembersRequest`
      direct pull (startup, periodic, on gap); version-sequence validation
      with audited catch-up.
- [ ] Presence heartbeats; offline at 3 missed beats; `Goodbye` + admin
      prune; rejoin resync.
- [ ] Bootstrap members re-publishing the announce (admin-offline
      survivability).
- [ ] `MembershipSync` behind a trait (the future iroh-gossip seam).
- [ ] Chaos tests on the fake transport: partitions, stale peers,
      withholding peers, announce rollback attempts — state must converge
      and never accept unsigned/unsequenced data.

**Demo:** 3 nodes; kill one mid-session, watch peers mark it offline;
restart it and watch it resync; take the admin offline and confirm the
group still syncs (and that *changes* correctly queue).

## Phase 4 — Routing + ACL + the port-forward data plane

**Goal: the product moment — `psql` across clouds, no VPN, no root.**

- [ ] `rmesh expose add <name> <host:port>` (declared via presence).
- [ ] ACL: role capabilities + service rules, default deny;
      `rmesh acl show/allow`.
- [ ] Tunnel streams: TunnelOpen/Ok/Err header frames, raw byte piping,
      FIN/RST mapping; destination-side authoritative enforcement;
      observable rejects.
- [ ] `rmesh forward --to <member>:<service> --listen <addr>`.
- [ ] Revocation: state v+1 + epoch rotation + live-tunnel teardown.
- [ ] Adversarial tests: non-member opens tunnel, member without Connect,
      undeclared service, rule-denied pair, revoked node with old state,
      observer trying anything.
- [ ] Baseline latency/throughput: direct path vs. relay fallback, vs. a
      raw `iroh` stream (our overhead), vs. Tailscale on the same pair
      (context).

**Demo:** Postgres on a bare-metal box, `psql` from an AWS instance and
from a *rootless container* through the mesh; revoke the container's node
live and watch the tunnel drop and reconnects get denied.

## Phase 5 — POC hardening & demo polish

**Goal: something you can hand to a stranger with a README.**

- [ ] `rmesh doctor`: endpoint/discovery state, announce freshness, state
      versions, presence table, recent rejects with reasons.
- [ ] Resilience: offline start, DHT slow paths, relay-only networks,
      admin-offline behavior documented (joins/revocations queue).
- [ ] Structured audit log of membership changes + rejected traffic.
- [ ] Docs: quickstart, demo script, threat-model recap; positioning
      one-pager aligned with `06-landscape.md`.
- [ ] Tag `v0.1.0-poc`.

**POC exit criteria:** 3+ nodes on different networks (NAT'd, mixed cloud +
bare metal + one rootless container) create a group, invite, sync
membership, forward real TCP services under ACL, and revoke a member — with
zero infrastructure of ours running anywhere, **zero sudo anywhere**, and
measured tunnel performance numbers in hand.

## Post-POC directions

Recorded so POC decisions don't paint us out of them; unscheduled.

1. **Gossip membership propagation** — `iroh-gossip` per-group topics
   behind `MembershipSync`; presence and pokes become gossip; validation
   pipeline unchanged.
2. **Admin quorum** — group identity key → threshold signatures (FROST);
   removes the single-admin trust concentration and doubles down on
   differentiator #1 (per-admin accountability — `06-landscape.md`).
3. **Overlay IP mode (TUN)** — *optional* privileged mode for whole-network
   use cases (where Rayfish lives today); port forwarding stays the
   container-native default.
4. **Multi-device identity** — user key signs device keys.
5. **Web dashboard** (chat decision #8: CLI first) and **mobile** (iroh on
   iOS/Android; more integration work than Veilid would have been — the
   price of amendment 6′, accepted).
6. **UDP forwarding**, large transfers (iroh-blobs), service naming sugar.
7. **Self-hosted relays/DNS** for constrained or air-gapped deployments.
8. **Anonymity option** — if route-level privacy (who-talks-to-whom) ever
   becomes a requirement, evaluate per-group onion-style relaying; rmesh's
   current claim is content + membership privacy only.

## Open questions (to settle during Phases 0–2)

- pkarr record size in practice: how many bootstrap EndpointIds fit beside
  versions/hash in ~1 KiB; republish cadence vs. mainline expiry.
- iroh discovery latency cold vs. warm; mDNS behavior in container
  networks without multicast.
- Whether `JoinAccept` carrying full `GroupState` needs chunking for
  large groups (control-frame cap is 64 KiB; state cap 256 KiB).
- Relay-only environments (UDP blocked): throughput/latency floor for
  interactive tunnels (ssh) — measure in Phase 4.
- iroh API stability across minor versions; pin + isolate in `rmesh-iroh`.

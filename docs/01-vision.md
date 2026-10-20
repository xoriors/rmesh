# rmesh — Vision

## One-liner

**rmesh gives machines private, invitation-only groups over a peer-to-peer
mesh — container-native (no root, no TUN), with encrypted membership and
per-service ACLs — built on iroh, with no coordination server and no company
in the middle.**

## The problem

You have machines everywhere — EC2 instances in AWS, VMs in Azure, bare-metal
servers, CI runners, containers, a workstation at home — and you want them to
talk privately, as if they shared a network. Today's answers:

- **VPN hub (WireGuard/OpenVPN server)**: all traffic hairpins through a
  server you must host, pay for, and trust; latency suffers.
- **Tailscale / NetBird / ZeroTier**: excellent UX, direct paths — but
  membership, key distribution, and ACLs live in a company-operated
  coordination plane. The data plane is E2EE; the control plane isn't yours.
- **Decentralized mesh VPNs (Rayfish, MeshGuard, EasyTier…)**: no
  coordination server — but they want root for a TUN device, their
  membership rosters sit *readable* on a public DHT, and their authority
  model is typically one shared network secret (see `06-landscape.md`).

rmesh takes the remaining corner seriously: **fully decentralized *and*
container-native *and* private**. Peer-to-peer transport (direct connections,
no VPN-server hairpin) with a decentralized control plane — and four
deliberate choices that are rmesh's identity:

1. **Admins are roles in signed data, not holders of a shared network key.**
   Every membership and policy change is signed by an identified key and
   validated by every member — per-admin accountability, no single secret
   whose theft burns the network.
2. **Group state is encrypted, not just signed.** The roster, ACLs, and
   service catalog are ciphertext to DHT nodes and everyone outside the
   group; the group key rotates on every membership change.
3. **No root, no TUN.** The data plane is SSH-forwarding-style port
   tunneling through an unprivileged daemon — it runs in containers, CI
   runners, and shared hosts. Overlay-IP/TUN mode is a roadmap *option*,
   never the entry price.
4. **Per-service, per-role ACLs enforced on every node.** Only declared,
   named services are reachable, under default-deny rules checked at the
   destination — application-level policy, not host firewall edits.

## What it looks like in use

An rmesh **daemon** runs on every node (unprivileged). Nodes find each other
through the mesh and enforce group boundaries:

- **Groups** are private networks inside the big mesh. Only members
  communicate within them.
- **Joining is invitation-based**: `rmesh group create` on node A mints a
  group; `rmesh invite create` produces a cryptographic token; paste it into
  `rmesh join` on node B; members validate the voucher and B is in.
- **Access is like SSH port forwarding**: an app on machine A connects to a
  local port served by the daemon; the daemon carries the bytes over an
  end-to-end-encrypted QUIC connection to the daemon on machine B, which
  forwards them to the target port on B's localhost — if, and only if, the
  ACL allows it.

```
psql ──▶ localhost:8443 (rmesh daemon, machine A)
              │  E2E-encrypted QUIC, direct when possible (iroh)
              ▼
         rmesh daemon (machine B) ──▶ localhost:5432 (postgres)
```

## Transport: iroh

Building DHTs, NAT traversal, and relays from scratch is years of work that
isn't rmesh's value. rmesh builds on **[iroh](https://iroh.computer)**:
QUIC-based peer-to-peer connections with best-in-class hole punching and
relay fallback, dial-by-key (Ed25519 node ids), discovery via **pkarr signed
records on the BitTorrent mainline DHT** (plus DNS and local discovery), and
`iroh-gossip` for epidemic broadcast when we want it.

The original brainstorm chose a Veilid POC with a later iroh migration,
because iroh then lacked discovery infrastructure. That rationale is stale —
iroh now ships mainline-DHT discovery natively — so the plan was amended
(`00-brainstorm-decisions.md`, amendment 6′): **build on iroh directly**.
Native QUIC byte-streams also make the tunnel data plane nearly free, and
identity = iroh's Ed25519 keys keeps one keypair per node.

Trade-off, stated plainly: Veilid's onion-style private routing would have
hidden *who talks to whom* even from relays; iroh connections expose peer IPs
to each other (fine inside a private group) and relay operators see
connection metadata (not content). rmesh's privacy claim is **encrypted
membership and content**, not transport anonymity. If route-level anonymity
ever becomes a requirement, it returns as a per-group policy question.

## The POC (this repo)

A Rust daemon + CLI (`rmesh`) proving the lifecycle end to end on real
machines across real NATs:

1. **Create a group** — root-of-trust keys; encrypted group state published
   and announced via a signed pkarr record under the group's key.
2. **Invite** — mint a signed, expiring invitation token (paste/link/QR).
3. **Join** — redeem the invite over a direct QUIC connection to the admin;
   members validate the voucher; the roster version bumps.
4. **Sync membership explicitly** — pull "who's in this group now?" from the
   group announce + any member; joins/leaves propagate by explicit sync (no
   gossip yet); liveness via heartbeats and timeouts.
5. **Forward ports** — `rmesh expose` + `rmesh forward`, ACL-checked at the
   destination: `psql` from an AWS box to a bare-metal Postgres through the
   mesh — no VPN, no public exposure, **no sudo anywhere**.
6. **Revoke** — roster removal + group-key rotation; live tunnels torn down;
   forward exclusion from new state.

Out of scope for the POC: gossip propagation, overlay-IP/TUN mode, exit
nodes, Magic-DNS naming, web dashboard, mobile, browser. Roadmap, not
blocked.

## Design principles

1. **No privileged infrastructure.** Admins are roles expressed in signed
   data; any member can verify everything.
2. **Explicit sync.** Membership converges through deliberate, observable
   pulls and version bumps — attributable and debuggable. Gossip comes
   later, behind the same interface.
3. **Verify, don't trust.** Every roster change, ACL change, and connection
   attempt is validated against signed state on every node. The DHT and
   relays are trusted for availability only.
4. **Private by construction.** Group state is encrypted to members;
   non-members (including DHT nodes) learn nothing but the group's announce
   record exists.
5. **Unprivileged by default.** Everything works without root; privileged
   features (TUN) must justify themselves later as options.

# rmesh — Vision

## One-liner

**rmesh is a Tailscale-like private network without the VPN and without the
coordination server: machines across clouds and homes form invitation-only
groups inside a peer-to-peer mesh, and traffic flows directly between peers.**

## The problem

You have machines everywhere — EC2 instances in AWS, VMs in Azure, bare-metal
servers, a workstation at home — and you want them to talk to each other
privately, as if they shared a network. Today's answers:

- **VPN hub (WireGuard/OpenVPN server)**: all traffic hairpins through a
  server you must host, pay for, and trust; latency suffers.
- **Tailscale / NetBird / ZeroTier**: excellent UX, direct WireGuard paths —
  but membership, key distribution, and ACLs live in a **company-operated
  coordination plane**. The data plane is E2EE; the control plane is not
  yours.

rmesh takes the third path: **fully decentralized**. Peer-to-peer transport
(no VPN server → traffic goes direct, like torrents, lower latency) *and* a
decentralized control plane (membership, invitations, ACLs are signed data
structures the peers themselves replicate and verify — no company in the
middle).

## What it looks like in use

An rmesh **service/daemon** runs on every node. Node instances find each
other over the mesh and enforce group boundaries:

- **Groups** are private networks inside the big mesh — VPC-style
  segmentation. Only members of a group can communicate within it.
- **Joining is invitation-based**: create a group on node A, it mints a
  cryptographic invitation token; paste it into the CLI on node B; existing
  members validate the voucher and node B is in.
- **Access is like SSH port forwarding**: an app on machine A connects to a
  local port served by the rmesh daemon; the daemon carries the bytes over
  the encrypted mesh to the rmesh daemon on machine B, which forwards them to
  the target port on B's localhost. The daemons are the gateways — the
  routing + ACL layer decides, on every node, whether a given peer may reach
  a given service.

```
app ──▶ localhost:8443 (rmesh daemon, machine A)
             │  E2E-encrypted, peer-to-peer (direct when possible)
             ▼
        rmesh daemon (machine B) ──▶ localhost:5432 (postgres)
```

## Transport strategy: Veilid now, Iroh option later

Building DHTs, NAT traversal, relays, and private routing from scratch is
years of work that isn't rmesh's value. Two Rust libraries were evaluated:

- **[Veilid](https://veilid.com)** — batteries-included P2P framework: DHT
  with encrypted records, peer discovery, connection management, RPC
  (AppCall/AppMessage), privacy-preserving private routes, VLD0 crypto suite
  (Ed25519/X25519/XChaCha20-Poly1305/BLAKE3), mobile-first multi-platform.
  Opinionated — custom routing semantics fight its abstractions.
- **[Iroh](https://iroh.computer)** — thin QUIC transport: best-in-class hole
  punching, stateless relays. No DHT/storage/RPC of its own — discovery
  (e.g. mainline DHT with encrypted metadata) and everything above it is
  yours to build, which is exactly what full custom routing control needs.

**Decision (see `00-brainstorm-decisions.md`): POC on Veilid** — fastest way
to validate what actually matters (groups, invitations, membership sync, the
routing/ACL model) because discovery, DHT, encryption, and RPC come for free.
**Iroh + mainline DHT remains the planned migration path** if/when custom
path selection (latency-, bandwidth-, reputation-aware routing) becomes the
differentiator. Consequently the POC keeps all transport access behind a
narrow adapter so the swap is realistic, and treats Veilid-specific features
with care.

## The POC (this repo)

A Rust daemon + CLI (`rmesh`) proving the lifecycle end to end on real
machines across real NATs:

1. **Create a group** on one node — the group's root of trust and DHT records.
2. **Invite** — mint a signed, expiring invitation token (link/QR/paste).
3. **Join** — redeem the invite over the mesh; members validate the voucher
   cryptographically; the roster updates.
4. **Sync membership explicitly** — nodes pull "who's in this group now?"
   from members and the group's DHT records; joins/leaves propagate by
   explicit sync (no gossip yet); dead nodes are detected by health-check
   timeouts; leaves announce goodbye.
5. **Forward ports** — `rmesh forward` opens a local port that tunnels to a
   port on another member, subject to the ACL; this is the demo that matters:
   `psql` from an AWS box to a bare-metal Postgres through the mesh, no VPN,
   no public exposure.
6. **Enforce ACLs** — roles and per-service rules checked by every node on
   every connection; revoke a member and watch its access die.

Out of scope for the POC: true gossip membership propagation, custom path
selection, overlay IP/TUN mode, web dashboard, mobile and browser clients.
All are roadmap items (`05-roadmap.md`); none are blocked by POC decisions.

## Design principles

1. **No privileged infrastructure.** Admins are roles expressed in signed
   data, not servers. Any member can verify everything.
2. **Explicit sync.** Membership converges through deliberate, observable
   pulls and version bumps — attributable, debuggable, no opaque gossip (yet).
3. **Verify, don't trust.** Every roster change, ACL change, and connection
   attempt is validated against signed state on every node. The DHT and
   relays are trusted for availability only.
4. **E2EE above the transport.** Veilid encrypts transport and DHT records;
   rmesh additionally encrypts group state so non-members (including DHT
   nodes) see nothing, and rotates keys on membership change.
5. **Swappable transport.** All Veilid access goes through one adapter crate
   with a minimal interface — the Iroh migration must never require touching
   group/ACL/sync logic.

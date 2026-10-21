# rmesh — Vision

## One-liner

**rmesh gives machines private, invitation-only groups over a peer-to-peer
mesh — container-native (no root, no TUN), with encrypted membership,
per-service ACLs, and private routing — with no coordination server and no
company in the middle.**

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
  membership rosters sit *readable* on a public DHT, their authority model
  is typically one shared network secret, and who-talks-to-whom is visible
  to relays and observers (see `06-landscape.md`).

rmesh takes the remaining corner seriously: **fully decentralized *and*
container-native *and* private**. Peer-to-peer transport (direct member
connections, no VPN-server hairpin) with a decentralized control plane — and
five deliberate choices that are rmesh's identity:

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
5. **Transport privacy.** Over Veilid's private routes, even *who talks to
   whom* is hidden from relays and observers — not just the content.

## What it looks like in use

An rmesh **daemon** runs on every node (unprivileged). Nodes find each other
over the mesh and enforce group boundaries:

- **Groups** are private networks inside the big mesh — VPC-style
  segmentation. Only members communicate within them.
- **Joining is invitation-based**: `rmesh group create` on node A mints a
  group; `rmesh invite create` produces a cryptographic token; paste it into
  `rmesh join` on node B; members validate the voucher and B is in.
- **Access is like SSH port forwarding**: an app on machine A connects to a
  local port served by the daemon; the daemon carries the bytes over the
  end-to-end-encrypted mesh to the daemon on machine B, which forwards them
  to the target port on B's localhost — if, and only if, the ACL allows it.

```
psql ──▶ localhost:8443 (rmesh daemon, machine A)
              │  E2E-encrypted, peer-to-peer, private routing
              ▼
         rmesh daemon (machine B) ──▶ localhost:5432 (postgres)
```

## Inspiration: gsocket's secret-as-address

A useful reference point from outside the mesh-VPN world is
**[gsocket](https://github.com/hackerschoice/gsocket)** (Global Socket, by
The Hacker's Choice). gsocket connects two programs that share a secret —
regardless of NAT, firewall, or geography — by routing them through the
**Global Socket Relay Network (GSRN)**. There is no IP address to know and
no inbound port to open: each side derives a session ID from the shared
secret, both make *outbound* connections to the relay, and the relay pairs
them. The secret never leaves the host; key exchange is SRP (RFC 5054) with
a fresh ephemeral AES-256 session key and perfect forward secrecy, so the
relay only ever sees ciphertext and never needs a PKI. On top of this sit
simple tools — `gs-netcat` (a netcat/ssh-style pipe and shell),
`gs-sftp`/`gs-mount` (file transfer and remote mounts), and `gsocket`
itself, which makes an existing service behind NAT reachable by connecting
to a `*.gsocket` hostname.

Three of gsocket's ideas map directly onto what rmesh already wants to be,
and are worth stating as explicit design pressure:

1. **The credential *is* the address.** You reach a peer by proving you
   hold a secret, not by knowing where it is. rmesh's invitation tokens are
   already this shape — a redeemed invite is the rendezvous credential that
   bootstraps the first contact — and the parallel is worth keeping in mind
   as the invite/join path is designed: a single pasted token should be
   enough to find and authenticate a peer, with no address, port, or
   discovery configuration from the user.
2. **Zero inbound exposure, outbound-only.** gsocket needs no open ports and
   no privileged network setup to make a service behind NAT reachable. This
   is the same promise as rmesh's no-root/no-TUN data plane (choice #3) —
   gsocket is independent evidence that "reachable from anywhere without
   opening a port or touching the host's network" is a real, shippable
   property, and rmesh should hold itself to it.
3. **Relay for transport, blind to content.** The GSRN carries bytes it
   cannot read. rmesh treats the DHT and relays the same way — trusted for
   availability only (principle #3) — and goes further by encrypting *group
   state*, not just the transport.

And gsocket's **limitations are precisely where rmesh's differentiators
live** — it is as useful as a foil as it is as an inspiration:

- **A single relay operator.** The GSRN is one volunteer-run service, not a
  decentralized plane. rmesh's control plane is the DHT plus signed,
  member-validated state — no operator in the middle.
- **Traffic analysis.** gsocket concedes that an observer (and the relay)
  can see *that* two parties are talking, even if not what they say.
  rmesh's Veilid private routes hide the who-talks-to-whom graph itself
  (choice #5) — the single biggest thing gsocket does not attempt.
- **Shared-secret trust.** When more than two parties share a gsocket
  secret, any of them can take the listener slot and impersonate a peer;
  the model assumes members "play nicely." rmesh replaces the shared secret
  with per-admin signed identity and encrypted, rotating group keys (choices
  #1 and #2) — accountability and revocation instead of mutual trust.

Net: gsocket is the clean, two-party, relay-centric version of "connect by
secret, not by address." rmesh keeps that ergonomics goal — a token is all
you need — while adding the group lifecycle, decentralized authority, and
transport privacy that a single shared secret over a single relay cannot
give. (See `06-landscape.md` for where gsocket sits among the mesh-VPN
projects.)

## Transport strategy: Veilid now, Iroh option later

Building DHTs, NAT traversal, relays, and private routing from scratch is
years of work that isn't rmesh's value. Two Rust libraries were evaluated
(`00-brainstorm-decisions.md`):

- **[Veilid](https://veilid.com)** — batteries-included P2P framework: DHT
  with encrypted records and watches, peer discovery, RPC
  (AppCall/AppMessage), privacy-preserving **private routes** (onion-like),
  VLD0 crypto suite (Ed25519/X25519/XChaCha20-Poly1305/BLAKE3),
  mobile-first multi-platform.
- **[Iroh](https://iroh.computer)** — QUIC-based thin transport:
  best-in-class hole punching, stateless relays, dial-by-key, discovery via
  pkarr records on the mainline DHT, `iroh-gossip`. Full control over
  routing — and the stack Rayfish is built on.

**The POC is built in Rust on Veilid** (batteries-included P2P: DHT, private
routing, encryption), **with a planned option to migrate the transport to
Iroh + mainline DHT once custom routing control matters.** The 2026-10
landscape review recommended jumping straight to iroh; the decision
(amendment 6″) is to keep the Veilid POC: its DHT records, watches, and RPC
remain the fastest path to validating what actually matters (groups,
invitations, sync, ACLs), its private routing is differentiator #5 — which
no iroh-based project offers — and its mobile-first support keeps the later
mobile option open. iroh's matured discovery stack (pkarr/mainline DHT) cuts
the cost of the *migration*, not the value of the POC: everything above the
transport adapter carries over. Consequently all transport access stays
behind one narrow adapter crate, and Veilid-specific features are used
deliberately.

## The POC (this repo)

A Rust daemon + CLI (`rmesh`) proving the lifecycle end to end on real
machines across real NATs:

1. **Create a group** on one node — the group's root of trust and DHT
   records.
2. **Invite** — mint a signed, expiring invitation token (paste/link/QR).
3. **Join** — redeem the invite over a Veilid private route; members
   validate the voucher cryptographically; the roster updates.
4. **Sync membership explicitly** — nodes pull "who's in this group now?"
   from members and the group's DHT records; joins/leaves propagate by
   explicit sync (no gossip yet); dead nodes are detected by health-check
   timeouts; leaves announce goodbye.
5. **Forward ports** — `rmesh expose` + `rmesh forward`, ACL-checked at the
   destination: `psql` from an AWS box to a bare-metal Postgres through the
   mesh — no VPN, no public exposure, **no sudo anywhere**.
6. **Revoke** — roster removal + group-key rotation; live tunnels torn
   down; forward exclusion from new state.

Out of scope for the POC: gossip propagation, custom path selection,
overlay-IP/TUN mode, exit nodes, web dashboard, mobile, browser. Roadmap,
not blocked.

## Design principles

1. **No privileged infrastructure.** Admins are roles expressed in signed
   data, not servers. Any member can verify everything.
2. **Explicit sync.** Membership converges through deliberate, observable
   pulls and version bumps — attributable and debuggable. Gossip comes
   later, behind the same interface.
3. **Verify, don't trust.** Every roster change, ACL change, and connection
   attempt is validated against signed state on every node. The DHT and
   relays are trusted for availability only.
4. **Private by construction.** Veilid encrypts transport and hides routes;
   rmesh additionally encrypts group state with a group key known only to
   current members, rotated on membership change.
5. **Unprivileged by default.** Everything works without root; privileged
   features (TUN) must justify themselves later as options.
6. **Swappable transport.** All Veilid access goes through one adapter
   crate with a minimal interface — the Iroh migration must never require
   touching group/ACL/sync logic.

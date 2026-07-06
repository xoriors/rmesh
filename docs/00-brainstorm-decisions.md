# rmesh — Decision Log from the Initial Brainstorm

Distilled from the voice brainstorm (2026-07, "Building a Tailscale-like mesh
network with custom transport layers"). These are the decisions the rest of
the docs implement; if a doc contradicts this log, the log wins until the doc
is fixed.

## What we're building

A **Tailscale-like private network without VPN servers**: machines spread
across AWS, Azure, bare metal, and private computers form **private groups**
inside a larger peer-to-peer mesh. Only members of a group can communicate
with each other — VPC-style segmentation, but decentralized and peer-to-peer
(direct connections between nodes, like torrents do it, no traffic through a
central VPN server → lower latency).

## How it works (agreed shape)

- An instance of our application runs on each node as a **service/daemon**.
- These service instances form the mesh and talk to each other.
- Cross-machine access works **like SSH port forwarding**: a local app on
  machine A connects to the local rmesh service, the service carries the
  traffic over the mesh to the rmesh service on machine B, which forwards it
  to the target port on B's localhost. The rmesh service is the gateway that
  enforces group boundaries.
- The **routing + ACL layer** checks: is this peer in the target group and
  allowed by policy? If yes, route; if no, drop.

## Decisions

| # | Decision | Rationale from the chat |
|---|---|---|
| 1 | **Rust** | Preferred language; both candidate libraries are Rust. |
| 2 | **Fully decentralized control plane** | No coordination server at all — harder but the point of the project. |
| 3 | **Invitation-based group membership** (not open discovery) | Tailscale-analogous UX; simpler; avoids Byzantine "forged membership" problems early. Existing members sign vouchers for new members; others validate cryptographically. |
| 4 | **Explicit sync for membership updates** (not gossip, for the POC) | Pull-based: a node asks a member "who's in this group now?" and updates its view. Leave = goodbye message; crash = detected by health-check timeout. True gossip is a post-POC upgrade. |
| 5 | **Routing + ACL layer is our application logic** | Group membership determines who can send what to whom; enforced by every node. |
| 6 | **Veilid for the POC** | Batteries-included (DHT, encryption, discovery, RPC) → fastest path to validating the group/ACL/routing model. Also mobile-first, which keeps the later mobile option open. |
| 7 | **Migrate transport to Iroh + mainline DHT after the POC** (planned option) | Iroh is a thin transport with best-in-class hole punching and stateless relays; paired with mainline DHT for encrypted discovery metadata it gives full control over custom routing (latency-, bandwidth-, reputation-aware path selection). The POC must keep the transport behind an abstraction so this swap is realistic. |
| 8 | **CLI control plane for the POC**; web dashboard later | Create group on node A → invitation token → paste into CLI on node B → validated → joined. |
| 9 | **Targets: servers & bare metal first**; mobile later; browser/wasm not a goal | Browser peers can't hole-punch (WebTransport/WebRTC sandbox) and would be relay-only — not worth it now. |
| 10 | **Traffic encrypted end-to-end, privacy-oriented** | A motivation for choosing Veilid (private routing) in the first place. |

## Veilid vs Iroh — the comparison that drove #6/#7

- **Veilid**: full application framework (DHT storage, discovery, RPC,
  encryption, mobile/web support) from the Cult of the Dead Cow. Opinionated;
  custom routing semantics mean working against its abstractions. Fastest POC.
- **Iroh** (iroh.computer): QUIC-based thin transport; excellent NAT
  traversal/hole punching (~95% success), stateless relays, mainline-DHT-based
  discovery is ours to build. No built-in DHT/storage/RPC — more upfront work,
  full control over routing. The long-term fit if custom routing becomes the
  competitive advantage.
- POC strategy: **build on Veilid, validate groups/invitations/sync/ACL,
  learn the real requirements, then decide on the Iroh migration.**

## Open items deliberately deferred

- True gossip protocol for membership propagation.
- Custom path selection (latency/bandwidth/reputation-aware routing).
- Web dashboard; mobile clients.
- Overlay IP mode (TUN device) vs. port-forwarding-only.

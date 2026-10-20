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

---

## Amendments — 2026-10 landscape review

A competitive-landscape review (see `06-landscape.md`) found the closest
existing project, **Rayfish** (Rust, iroh, MPL-2.0, experimental), already
shipping the generic "Tailscale without the coordination server" pitch, and
found that iroh now provides what decision #7 said it lacked: mainline-DHT
discovery (pkarr signed records, opt-in `discovery-pkarr-dht`) plus
`iroh-gossip`. Decisions revised accordingly:

| # | Revision | Rationale |
|---|---|---|
| 6″ | **#6/#7 stand: the POC stays on Veilid; iroh + mainline DHT remains the planned migration option** (owner's decision, 2026-10, reaffirmed after the review). | The review recommended building directly on iroh, since its old "no DHT" gap is closed (pkarr/mainline-DHT discovery, `iroh-gossip`) and Rayfish proves the stack. Reasons to keep the Veilid POC anyway: batteries-included DHT records/watches/RPC still make it the fastest path to validating the group/ACL/sync model; Veilid **private routing** gives transport privacy none of the surveyed projects have (a fifth differentiator); and mobile-first support keeps decision #9's later-mobile door open. iroh's matured discovery stack makes the *migration* cheaper than planned — it doesn't make the POC throwaway, because everything above the `Transport` adapter carries over. |
| 11 | **Positioning: the container-native, privacy-first variant** — not a general mesh VPN. | The generic pitch is taken (Rayfish is far ahead on it). rmesh's wedge is choices Rayfish doesn't make: (a) admins as roles in per-member signed data, not a shared network secret key; (b) *encrypted* group state — roster invisible to DHT nodes, key rotation on membership change; (c) unprivileged port-forward mode — no TUN, no root: containers, CI runners, shared hosts; (d) per-service, per-role ACLs verified on every node, not host firewall rules; (e) transport privacy via Veilid private routes — who-talks-to-whom is hidden, not just content. |
| 12 | **Unprivileged `rmesh forward` demo is the proof point** and a POC exit criterion. | It's the fastest demonstration of wedge (c) and needs no sudo anywhere. |

Decision #4 (explicit sync for the POC) stands; `iroh-gossip` would make the
post-POC gossip upgrade cheaper if the migration happens. Decisions #1–#5,
#8–#10 stand unchanged.

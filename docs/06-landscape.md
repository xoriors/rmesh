# rmesh — Competitive Landscape (2026-10)

Snapshot distilled from the October 2026 review discussion, with claims
spot-checked against public sources. This doc exists to keep rmesh's
positioning honest: what already exists, and what rmesh does differently.

## Closest project: Rayfish

**Rayfish** — Rust, built on iroh, MPL-2.0, experimental/pre-1.0
(crates.io `rayfish`). The generic "peer-to-peer mesh VPN with no server to
trust" pitch, shipping today:

- DHT peer discovery (pkarr signed records on the BitTorrent mainline DHT);
  no accounts, no policy server, no coordination service.
- Closed networks with one-time invitations (`ray invite`), live approval
  (`ray requests` / `ray accept`), reusable fleet keys, member removal.
- TUN data plane over iroh QUIC: identity-derived IPv4/IPv6 addresses,
  Magic-DNS-style naming, exit nodes; per-device firewall rules.
- Control plane: membership is a `GroupBlob` (members + approved + suggested
  firewall + reusable keys) **signed by a per-network secret key** and
  published to the DHT; peers reconverge only from that signed record. A
  coordinator becomes co-coordinator by being handed the network secret key;
  the coordinator can be offline once members are admitted.
- Status: no third-party security audit; Linux + macOS today; mobile absent.

## Other neighbors

| Project | Stack | Model | Gap vs. rmesh's goals |
|---|---|---|---|
| MeshGuard | Zig, WireGuard | SWIM gossip discovery (seed peers/DNS TXT/mDNS), per-node authorized keys or org-signed certs with revocation, per-peer/org port policies | needs a reachable seed; host-level trust model |
| EasyTier | Rust | shared-secret networks | no invitations, no ACLs |
| EdgeVPN | Go, libp2p | shared-secret networks | no invitations, no ACLs |
| iroh-lan, wgmesh | Rust | LAN-style meshes over iroh/WireGuard | shared secret, no membership lifecycle |
| Tinc, Nebula, Yggdrasil | C/Go | older generation | manual config or CA-centric; no DHT-era UX |
| gsocket (GSRN) | C | connect-by-shared-secret through one relay network; SRP E2EE, zero inbound ports | single relay operator; two-party trust model; relay/observer sees the traffic graph; no groups, ACLs, or membership lifecycle |

**gsocket** is not a mesh VPN — it is the closest thing to rmesh's
*ergonomics* goal from a different corner: reach a peer by proving you hold
a secret, with no address and no open port, through the Global Socket Relay
Network. It is worth tracking as both inspiration and foil (see the
"Inspiration: gsocket's secret-as-address" section in `01-vision.md`): its
secret-as-address, outbound-only, relay-blind-to-content properties are
exactly what rmesh's invite/join path and no-root data plane aim for, while
its centralized relay, two-party shared-secret trust, and traffic-analysis
visibility are precisely rmesh's differentiators #1, #2, and #5.

## What this means for rmesh

"Tailscale without the coordination server," stated generically, is **taken**
— Rayfish is far ahead on that pitch. rmesh is viable as the
**container-native, privacy-first variant**, on four differentiators none of
the above make:

1. **Per-admin accountability instead of a shared network key.** Rayfish's
   authority model is one per-network secret key passed between
   coordinators: no per-admin attribution, and one compromised coordinator
   burns the whole network. rmesh expresses admin actions as per-member
   signed data (and, post-POC, threshold signatures) — every membership
   change is attributable and revocable.
2. **Encrypted group state.** Rayfish's DHT record is signed but readable —
   the membership roster is public to anyone with the record key. rmesh's
   roster/ACL/state are encrypted with a group key known only to members and
   rotated on membership change; DHT nodes see ciphertext.
3. **Unprivileged port-forward mode.** No TUN device, no root, no network
   extension: rmesh runs in containers, CI runners, and shared hosts where
   Rayfish's `sudo` requirement is a non-starter. (TUN/overlay-IP mode stays
   on the roadmap as an *option*, not the entry price.)
4. **Per-service, per-role ACLs enforced by every node.** Named services and
   role/service rules checked at the destination on every connection —
   application-level policy, not host firewall rules.
5. **Transport privacy (Veilid private routes).** Every surveyed project
   exposes who-talks-to-whom to relays and observers; rmesh's POC transport
   hides the traffic graph itself, not just the content.

The review also recommended building directly on iroh (its old "no DHT"
gap is closed by pkarr/mainline discovery). **Outcome (amendment 6″ in
`00-brainstorm-decisions.md`): the POC stays on Veilid** — differentiator
#5 depends on it, and the batteries-included DHT/RPC remain the fastest
validation path — with iroh + mainline DHT kept as the recorded migration
option behind the transport adapter.

The honest corollaries, also recorded: if rmesh's goal were impact on the
general mesh-VPN problem, contributing differentiators 1–3 to Rayfish would
be higher leverage than a parallel build; and rmesh must not drift into
re-pitching the generic product. The wedge is the product.

## Verification notes

Spot-checked 2026-10: `rayfish` crate exists (0.1.x, docs builds flaky),
MPL-2.0, iroh-based TUN mesh VPN with invite/approval admission —
consistent with the review. iroh's mainline-DHT discovery via pkarr is
documented (opt-in `discovery-pkarr-dht` feature; `iroh-gossip` and
`iroh-pkarr-node-discovery` crates exist). Star/commit counts from the
review were not re-verified and are omitted above; re-check before quoting
numbers externally.

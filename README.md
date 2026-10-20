# rmesh

**rmesh** gives machines private, invitation-only groups over a peer-to-peer
mesh — **container-native (no root, no TUN), with encrypted membership,
per-service ACLs, and private routing** — with no coordination server, no
accounts, and no company in the middle.

Machines across AWS, Azure, bare metal, CI runners, and containers join
groups via cryptographic invitations; membership and ACLs are signed *and
encrypted* data replicated through a DHT and the members themselves; and
cross-machine access works like SSH port forwarding through an unprivileged
daemon, checked against default-deny per-service rules on every connection.

The POC is built in Rust on [Veilid](https://veilid.com) (batteries-included
P2P: DHT, private routing, encryption), with a planned option to migrate the
transport to [Iroh](https://iroh.computer) + mainline DHT once custom routing
control matters. Status: **design phase** — implementation follows the docs.

## How it's different

The generic "Tailscale without the coordination server" already exists
(see the [landscape doc](docs/06-landscape.md)). rmesh is the
container-native, privacy-first variant:

1. **Per-admin accountability** — admin actions are signed, attributable
   data, not uses of a shared network secret key.
2. **Encrypted group state** — the roster is ciphertext to DHT nodes and
   non-members, and the group key rotates on every membership change.
3. **No root, no TUN** — runs where mesh VPNs can't: containers, CI
   runners, shared hosts.
4. **Per-service, per-role ACLs** — enforced by every node at the
   destination, not host firewall rules.
5. **Transport privacy** — Veilid private routes hide who talks to whom,
   not just what they say.

## Docs

| Doc | Contents |
|---|---|
| [00 — Brainstorm decisions](docs/00-brainstorm-decisions.md) | Decision log + 2026-10 amendments — the source of truth |
| [01 — Vision](docs/01-vision.md) | What rmesh is, the Veilid-now/Iroh-later strategy, POC scope, principles |
| [02 — Architecture](docs/02-architecture.md) | Layer model, components, daemon/CLI shape, crate layout, trust model |
| [03 — POC Spec](docs/03-poc-spec.md) | Data schemas: DHT records, invites, envelopes, tunnels, validation rules |
| [04 — Protocols](docs/04-protocols.md) | Flows: create, invite/join, explicit sync, tunnels, revoke, routes |
| [05 — Roadmap](docs/05-roadmap.md) | Phases 0–5 to a demoable POC; post-POC: gossip, Iroh option, quorum, TUN |
| [06 — Landscape](docs/06-landscape.md) | Rayfish, MeshGuard, et al. — and the wedge rmesh occupies |

## The POC in one paragraph

A Rust daemon + CLI (`rmesh`) where you create a group on one machine, hand
a second machine an `rmesh://invite/…` token, and it joins over the live
Veilid network — membership validated cryptographically by the members
themselves, synced by explicit pull (no gossip yet), liveness by heartbeat
timeouts, with the group's roster encrypted end to end. Then
`rmesh expose postgres 127.0.0.1:5432` on one node and
`rmesh forward --to that-node:postgres --listen 127.0.0.1:8443` on another
gives you `psql` across clouds through an end-to-end-encrypted peer-to-peer
tunnel over private routes, governed by default-deny ACLs — and revoking a
member kills its access, live tunnels included. No servers of ours
anywhere, and no sudo anywhere.

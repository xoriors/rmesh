# rmesh

**rmesh** is a Tailscale-like private network without the VPN and without the
coordination server: machines across AWS, Azure, bare metal, and home form
**invitation-only private groups** inside a peer-to-peer mesh. Traffic flows
directly between peers (no VPN server hairpin), membership and ACLs are
signed data structures the peers themselves replicate and verify, and
cross-machine access works like SSH port forwarding through a local daemon.

The POC is built in Rust on [Veilid](https://veilid.com) (batteries-included
P2P: DHT, private routing, encryption), with a planned option to migrate the
transport to [Iroh](https://iroh.computer) + mainline DHT once custom routing
control matters. Status: **design phase** — implementation follows the docs.

## Docs

| Doc | Contents |
|---|---|
| [00 — Brainstorm decisions](docs/00-brainstorm-decisions.md) | Decision log from the original brainstorm — the source of truth |
| [01 — Vision](docs/01-vision.md) | What rmesh is, the Veilid-vs-Iroh strategy, POC scope, principles |
| [02 — Architecture](docs/02-architecture.md) | Layer model, components, daemon/CLI shape, crate layout, trust model |
| [03 — POC Spec](docs/03-poc-spec.md) | Data schemas: DHT records, invites, envelopes, tunnels, validation rules |
| [04 — Protocols](docs/04-protocols.md) | Flows: create, invite/join, explicit sync, tunnels, revoke, routes |
| [05 — Roadmap](docs/05-roadmap.md) | Phases 0–5 to a demoable POC; post-POC: gossip, Iroh, TUN, mobile |

## The POC in one paragraph

A Rust daemon + CLI (`rmesh`) where you create a group on one machine, hand a
second machine an `rmesh://invite/…` token, and it joins over the live Veilid
network — membership validated cryptographically by the members themselves,
synced by explicit pull (no gossip yet), liveness by heartbeat timeouts. Then
`rmesh expose postgres 127.0.0.1:5432` on one node and
`rmesh forward --to that-node:postgres --listen 127.0.0.1:8443` on another
gives you `psql` across clouds through an end-to-end-encrypted peer-to-peer
tunnel, governed by a default-deny ACL — and revoking a member kills its
access, live tunnels included. No servers of ours anywhere.

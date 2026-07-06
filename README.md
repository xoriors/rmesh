# rmesh

POC for **Telescale** — a decentralized private network layer built on
[Veilid](https://veilid.com): Tailscale-like groups with invitation-based
membership, explicit sync, and a routing + ACL layer, with no coordination
server and no accounts.

Status: **design phase**. Rust implementation to follow the docs below.

## Docs

| Doc | Contents |
|---|---|
| [01 — Vision](docs/01-vision.md) | What Telescale is, why Veilid, POC scope, design principles |
| [02 — Architecture](docs/02-architecture.md) | Layer model, components, crate layout, trust model |
| [03 — POC Spec](docs/03-poc-spec.md) | Data schemas: DHT records, invites, envelopes, validation rules |
| [04 — Protocols](docs/04-protocols.md) | Flows: create, invite/join, sync, send, revoke, route refresh |
| [05 — Roadmap](docs/05-roadmap.md) | Phases 0–5 to a demoable POC, then post-POC direction |

## The POC in one paragraph

A Rust node + CLI (`rmesh`) where you can create a group, hand someone a
`telescale://invite/…` link, have them join over the live Veilid network,
exchange end-to-end encrypted chat/key-value updates via explicit,
version-vectored sync, and revoke a member with cryptographic forward
exclusion — all with zero infrastructure operated by us.

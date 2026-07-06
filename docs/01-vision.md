# Telescale — Vision

## One-liner

**Telescale is a decentralized private network layer: Tailscale-like groups of
people and devices, with no coordination server, no accounts, and no company
in the middle.**

## The problem

Tailscale showed how good the UX of a private overlay network can be: install,
log in, and your devices see each other — with ACLs, key distribution, and NAT
traversal handled for you. But that UX is anchored to a **centralized
coordination plane**: a company-operated control server holds the device
directory, distributes keys, evaluates ACLs, and can (in principle) remove or
add members of your network. The data plane is end-to-end encrypted, but the
*membership and policy plane* is not yours.

Telescale asks: what does the same experience look like when the coordination
plane is **also** decentralized?

- **Membership** is a signed data structure replicated over a public DHT — not
  a row in a vendor's database.
- **Invitations** replace accounts: you join a group because an admin handed
  you a cryptographic invite, not because you logged into the same identity
  provider.
- **ACLs** are policy documents signed by the group's own keys and enforced by
  every member independently.
- **Transport** is a peer-to-peer network with NAT traversal and
  privacy-preserving routing — no rendezvous server you must trust.

## Why Veilid

Building the substrate — DHT, NAT traversal, relaying, onion-style private
routing, transport encryption — is years of work and not where Telescale's
value is. [Veilid](https://veilid.com) already provides exactly this layer,
open source and in Rust:

| Need | Veilid feature |
|---|---|
| Node identity | Cryptographic node keypairs (VLD0: Ed25519 / X25519 / XChaCha20-Poly1305 / BLAKE3) |
| Shared state without servers | DHT records with owner/writer keys and subkey schemas |
| Change propagation | DHT value watches |
| Direct member-to-member RPC | `AppCall` / `AppMessage` over routing contexts |
| Sender/receiver privacy | Safety routes + private routes (onion-like) |
| NAT traversal, relays, bootstrap | Built into veilid-core; public bootstrap or self-hosted |
| Local encrypted storage | TableStore / ProtectedStore |

Telescale is therefore an **application-layer protocol on top of Veilid**:
groups, invitations, membership, sync, and ACLs — the things Veilid
deliberately does not opine on.

## The POC (this repo, `rmesh`)

The proof of concept is a Rust node + CLI that demonstrates the full
membership lifecycle over the live Veilid network:

1. **Create a group** — generates the group's root of trust and publishes its
   DHT records.
2. **Invite** — admin mints a signed, expiring invite token (shareable as a
   link/QR).
3. **Join** — invitee redeems the invite over a Veilid private route; the
   admin verifies and adds them to the signed roster.
4. **Sync** — members explicitly sync group state (roster, ACLs, application
   data) via DHT watches + version-numbered records. No hidden gossip; sync is
   observable and deliberate.
5. **Send** — members exchange end-to-end encrypted application payloads
   (demo: chat messages / small key-value documents), with every message
   checked against the roster and ACL policy at its version.
6. **Revoke** — admin removes a member; the group data key rotates so the
   removed member is cryptographically excluded going forward.

What the POC intentionally does **not** do: overlay IP networking (TUN
devices, subnet routing), multi-device user identity, large file transfer,
mobile clients. Those are on the roadmap (see `05-roadmap.md`) once the
membership/sync/ACL core is proven.

## Design principles

1. **No privileged infrastructure.** Anything an "admin" does is expressed as
   signed data any member can verify. Admins are roles, not servers.
2. **Explicit sync.** State converges because members deliberately sync
   versioned records — never because of opaque background gossip. Every state
   transition is attributable to a signed update.
3. **Verify, don't trust.** Every roster change, ACL change, and message is
   signed and validated against the membership state at that version. A
   malicious relay or DHT node can withhold data, but never forge it.
4. **E2EE above the transport.** Veilid encrypts transport and DHT records;
   Telescale additionally encrypts group content with a group data key known
   only to current members, rotated on membership change.
5. **Small trusted computing base.** The POC's security-critical surface is
   the schema definitions and validation logic — kept in one crate,
   property-testable without a network.

## Naming

- **Telescale** — the project / protocol layer.
- **rmesh** — this repository and the POC node/CLI binary.
- **Veilid** — the underlying P2P network (external project, unaffiliated).

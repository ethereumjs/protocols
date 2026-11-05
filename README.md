# EthereumJS Protocols (`ethereumjs/protocols`)

Open wire protocols, subprotocol extensions, and binary data specifications designed within the **[EthereumJS](https://github.com/ethereumjs/ethereumjs-monorepo)** ecosystem and shared with the broader Ethereum community.

---

## Why This Repository Exists

As Ethereum evolves through the **Amsterdam** era and beyond — introducing **Block-Level Access Lists ([EIP-7928](https://eips.ethereum.org/EIPS/eip-7928))**, **Rolling-Window History Serving ([EIP-7642](https://eips.ethereum.org/EIPS/eip-7642))**, and **Altair/Electra Consensus Light Clients** — brand-new client architectures become possible that did not exist when legacy DevP2P specifications (`eth`, `snap/1`) were first written.

While building new EthereumJS libraries and nodes (such as **[`@ethereumjs/ethy`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/ethy)**, an execution-free, BAL-native State & Intelligence Node, and **[`@ethereumjs/beacon-light`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/beacon-light)**), we occasionally design wire protocols and subprotocol extensions that solve general peer-to-peer or state-distribution bottlenecks.

Whenever a protocol designed inside EthereumJS is clean, self-contained, and useful to other client teams, researchers, or AI-assisted implementers, we publish its specification here.

---

## Specification Design Philosophy (Human- & AI-First)

Traditional P2P wire specs are often terse packet tables that leave out cryptographic trust boundaries, edge-case encoding rules, or state-machine transitions. Every specification in `ethereumjs/protocols` is written to be **immediately implementable by both human protocol engineers and AI coding agents**:

1. **First-Principles Motivation & Security Proof:** Clear explanation of *why* the protocol exists, what cryptographic commitments it verifies, and what trust assumptions it makes (or eliminates).
2. **Exact Wire & Binary Layouts:** Unambiguous RLP / SSZ / fixed-byte schemas, including canonical scalar encoding rules (such as RLP zero `0x80` vs `0x00`) and framing invariants.
3. **End-to-End State Machines & Sync Selectors:** Concrete algorithms for capability negotiation, peer selection, timeout/backoff handling, and fallback behavior.
4. **Worked Byte-Level Examples & Integrator Checklists:** Step-by-step verification walkthroughs and pitfall tables so any client (TypeScript, Rust, Go, C#, Nim, Python) can interop on the first try.

---

## Protocol Catalog

| Domain | Protocol ID | Capability / Wire Tag | Status | Specification | Reference Implementation |
| --- | --- | --- | --- | --- | --- |
| **DevP2P (`RLPx`)** | **`ETHY/0`** | `["ethy", 0]` · ENR `"ethy"` | Draft (`v0`) | **[`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md)** | [`@ethereumjs/devp2p`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/devp2p) & [`@ethereumjs/ethy`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/ethy) |

---

## Featured Protocol: `ETHY/0` — Self-Proving Flat State & BAL Swarm Protocol

**[`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md)** specifies the **`ETHY/0`** DevP2P subprotocol (`["ethy", 0]`), designed for post-Amsterdam Ethereum nodes that hold verified flat state without maintaining an 80 GB database of interior Merkle Patricia Trie (`MPT`) branch nodes:

- **The Problem:** In `snap/1`, serving `AccountRange` or `StorageRanges` requires generating Merkle boundary proofs on the fly from 128 historical MPT roots, turning flat-state clients into pure `snap/1` consumers that cannot help onboard new peers.
- **The `ETHY/0` Solution:**
  1. **Self-Proving State Chunks:** When a node downloads and verifies a flat account/storage range chunk (`[originHash .. lastHash]`), it retains the ~4 KB left/right Merkle boundary proof (`proofNodes`) alongside the flat 136-byte `BinaryAccount` slab (`< 3%` disk overhead). Any `ETHY/0` node can re-serve that chunk from flat disk files with `O(1)` seeks, and the receiver verifies it cryptographically via standard `verifyMerkleRangeProof` against the chunk's `stateRoot`!
  2. **Historical Checkpoint + Compact `BAL` Tail (`#C + 1 .. #Head`):** Because EIP-7928 Block-Level Access Lists (`eth/71`) advance flat state deterministically without EVM execution or `GetTrieNodes` trie healing, an `ETHY/0` snapshot does not expire after 128 blocks (~25 minutes). Peers can seed a verified base snapshot from hours or days ago (`#C`) plus a batch-streamed `BAL_RANGE` tail up to the `@ethereumjs/beacon-light` verified head — enabling a **`1:100+` `snap/1` offload ratio**.
  3. **Built-In Epidemic Peer Exchange (`ETHY_PEX`) & ENR Discovery:** Includes native `GET_ETHY_PEERS` / `ETHY_PEERS` exchange and an EIP-778 ENR `"ethy"` key so sparse `ethy/0` nodes discover each other in `< 200ms` and serve chunks bidirectionally even across NATed outbound TCP connections.

---

## Contributing & Governance

All changes to specifications in this repository land via Pull Requests with clear rationale and, where applicable, matching updates in the reference implementation in [`ethereumjs/ethereumjs-monorepo`](https://github.com/ethereumjs/ethereumjs-monorepo).

## License

[MIT](./LICENSE) © EthereumJS Team

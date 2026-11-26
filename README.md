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
3. **End-to-End State Machines & Sync Selectors:** Concrete algorithms for capability negotiation, peer selection, custody reciprocity, and Byzantine fault handling.
4. **Worked Byte-Level Examples & Integrator Checklists:** Step-by-step verification walkthroughs and pitfall tables so any client (TypeScript, Rust, Go, C#, Nim, Python) can interop on the first try.

---

## Protocol Catalog

| Domain | Protocol ID | Capability / Wire Tag | Status | Specification | Reference Implementation |
| --- | --- | --- | --- | --- | --- |
| **DevP2P (`RLPx`)** | **`ETHY/0`** | `["ethy", 0]` · ENR `"ethy"` | Draft (`v0.4` — Unified Custody) | **[`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md)** | [`@ethereumjs/devp2p`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/devp2p) & [`@ethereumjs/ethy`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/ethy) |
| **DevP2P (`RLPx`)** | ~~`STATY/0`~~ | ~~`["staty", 0]`~~ | **Superseded** (merged into `ETHY/0` `v0.4`) | [`devp2p/STATY-0-SPEC.md`](./devp2p/STATY-0-SPEC.md) | Consolidated into `ETHY/0` `v0.4` (`0x0b`–`0x0e`) |

---

## Featured Protocol: Unified Custody `ETHY/0` (`v0.4`, Alongside `eth/71`)

Together with standard **`eth/71`** (live headers, blocks, tx broadcast, and rolling EIP-7928 BAL windows), **`ETHY/0` (`v0.4`)** delivers **one unified, custody-sharded, zero-`snap/1` state protocol** for post-Amsterdam Ethereum ([`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md)):

- **The Problem:** Replicating 100% of Ethereum world state (`~160+ GB` on Mainnet) takes hours, while ad-hoc `snap/1` light clients permanently depend on `snap/1` full nodes and create an asymmetric free-rider split.
- **The Unified Custody `ETHY/0` (`v0.4`) Solution:**
  1. **One Custody Dial (`--custody <1..16>`, default `2` = `1/8th`; `--full` = `16`; no `--custody 0`):** Every default Ethy node boots **live in 2–3 minutes** via rolling `eth/71` BALs, custodies **`2/16` (`1/8th`) of Ethereum (`~20–26 GB` Mainnet / `~3–4 GB` Sepolia)** across all three planes (`Accounts`, `Bytecode`, `Storage`) in the background, and flips its assigned buckets to Closed-World. Holding 2 buckets per node means **7 peers give 100% 1-hop coverage**, **`14–20` peers give 2× fault-tolerant redundancy** (zero DHT!), and **8 Ethy nodes form a self-sustaining swarm**.
  2. **Zero `snap/1` in Default Clients (Not Even Negotiated in `HELLO`):** Open-World cold misses resolve in 1 hop directly from bucket custodians (`0x0b`/`0x0c STATE_WITNESS` ~2 KB proofs or targeted `0x03`/`0x05` custody chunks folded forward with local `eth/71` BALs). Only a Day-0 `--full --snap-bootstrap` origin seeder ever mounts `snap/1` once to seed a fresh network.
  3. **`1/16th` Live Subtree MPTs & 512-Byte `[h_0..h_f]` Root Assembly (`0x0d`/`0x0e SUBTREE_ROOTS`):** Because Ethereum's state root is always a 16-way branch node `RLP([h_0..h_f, 0x])` where `h_b` depends exclusively on nibble bucket `b`, each custodian maintains a compact `1/16th` subtree trie updated solely from `eth/71` BALs (`< 2–5 ms/block`), gossips `h_b` to verify `keccak256(RLP([h_0..h_f, 0x])) === header.stateRoot` (with Byzantine quarantine), and generates **both `< 1 ms` live MPT witnesses and self-refreshing pivot chunks forever with `0` `snap/1` calls**.
  4. **Protocol-Level Custody Reciprocity & Bucket-Aware `ETHY_PEX`:** `0x09`/`0x0a` steers peers toward uncovered buckets while excluding `bucketsBitmask === 0` peers; servers protect active custodians and throttle/evict permanent non-custodians.

---

## Contributing & Governance

All changes to specifications in this repository land via Pull Requests with clear rationale and, where applicable, matching updates in the reference implementation in [`ethereumjs/ethereumjs-monorepo`](https://github.com/ethereumjs/ethereumjs-monorepo).

## License

[MIT](./LICENSE) © EthereumJS Team

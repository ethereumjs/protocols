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
| **DevP2P (`RLPx`)** | **`STATY/0`** | `["staty", 0]` · ENR `"staty"` | Draft (`v0.1`) | **[`devp2p/STATY-0-SPEC.md`](./devp2p/STATY-0-SPEC.md)** | [`@ethereumjs/devp2p`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/devp2p) & [`@ethereumjs/ethy`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/ethy) *(planned)* |
| **DevP2P (`RLPx`)** | **`ETHY/0`** | `["ethy", 0]` · ENR `"ethy"` | Draft (`v0.3`) | **[`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md)** | [`@ethereumjs/devp2p`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/devp2p) & [`@ethereumjs/ethy`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/ethy) |

---

## Featured Protocol Suite: `STATY/0` + `ETHY/0` (Alongside `eth/71`)

Together with standard **`eth/71`** (live headers, blocks, tx broadcast, and rolling EIP-7928 BAL windows), **`STATY/0`** and **`ETHY/0`** form a complementary two-tier post-Amsterdam state suite — each standing cleanly on its own while sharing one immutable `code/` store, one verified `bals/` archive, and one power-of-two block calendar (`32 / 256 / 1,024 / 4,096 / 8,192`):

### 1. `STATY/0` — BAL-Extended Active State & Witness Swarm ([`devp2p/STATY-0-SPEC.md`](./devp2p/STATY-0-SPEC.md))

- **The Problem:** Syncing 100% of Ethereum world state (`~160+ GB` on Mainnet) takes hours, while pure on-demand `snap/1` light clients leech from full nodes without giving back (and their MPT proofs expire after 128 blocks / ~25 minutes).
- **The `STATY/0` Solution:**
  1. **Multi-Day Reusable Witnesses via `eth/71` BALs:** By verifying a rolling `8,192`-block `(header, BAL)` window over standard `eth/71` (`STATY/0` carries **zero BAL messages on wire**), a `STATY/0` node folds BAL writes forward onto historical MPT inclusion/exclusion proofs — keeping them cryptographically valid at `#Head` even when hot contract `storageRoot`s mutate every block.
  2. **Deterministic Hot Epoch Bundles (`E = 256` blk, `D = 4,096` blk dedup):** `1–2` elected nodes per 51-minute micro-epoch mint proofs for the BAL-touched working set once for the entire swarm (`O(1)` `snap/1` load), letting joining nodes reach full verified JSON-RPC (`eth_call`, ENS, ERC-20, DeFi) in **3–5 minutes (`~3–10 GB` disk)**.
  3. **Symmetric Giving-Back:** Every `STATY/0` node seeds Hot Epoch Bundles (`EPOCH_PART`), cached cold point witnesses (`WITNESSES`), content-addressed bytecodes (`BYTECODES`), and its rolling `eth/71` BAL window.

### 2. `ETHY/0` — Self-Proving Full Flat State & BAL Swarm ([`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md))

- **The Problem:** In `snap/1`, serving `AccountRange` or `StorageRanges` requires generating Merkle boundary proofs on the fly from 128 historical MPT roots (`~80 GB` interior trie DB), turning flat-state clients into pure `snap/1` consumers.
- **The `ETHY/0` Solution:**
  1. **Self-Proving Full State (`Accounts`, `Storage`, `Bytecode`):** Retains compact (~3–5 KB) left/right Merkle boundary proofs alongside 136-byte `BinaryAccount` slabs and 64-byte `BinaryStorageSlot` slabs (`kind = 1` large contracts + `kind = 2` zero-proof batched small contracts), with explicit per-plane bitmasks (`bucketsBitmask`, `storageBucketsBitmask`, `codeBucketsBitmask`) in `STATUS`.
  2. **Historical Checkpoint + `BAL_RANGE` (`#C + 1 .. #Head`):** Peers seed a verified base snapshot from hours or days ago plus a batch-streamed `BAL_RANGE` tail up to the `@ethereumjs/beacon-light` verified head — enabling a **`1:100+` `snap/1` offload ratio** and **100% zero-leak read privacy** for `--full` nodes.
  3. **Built-In Epidemic Peer Exchange (`ETHY_PEX`) & ENR Discovery:** Native `GET_ETHY_PEERS` / `ETHY_PEERS` and ENR `"ethy"` discovery with bidirectional serving across NATed outbound TCP connections.

---

## Contributing & Governance

All changes to specifications in this repository land via Pull Requests with clear rationale and, where applicable, matching updates in the reference implementation in [`ethereumjs/ethereumjs-monorepo`](https://github.com/ethereumjs/ethereumjs-monorepo).

## License

[MIT](./LICENSE) © EthereumJS Team

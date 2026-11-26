# `ETHY/0` — Unified Custody State, Subtree-Proof & BAL Swarm Protocol (`devp2p`)

| Field | Value |
| --- | --- |
| **Protocol Identifier** | `ethy` |
| **Protocol Version** | `0` (`["ethy", 0]`) |
| **Transport Layer** | Ethereum DevP2P (`RLPx` v5 with Snappy framing) |
| **Message Space** | `0x00` – `0x0e` (`length = 15`) |
| **Discovery ENR Key** | `"ethy"` ([EIP-778](https://eips.ethereum.org/EIPS/eip-778) key `"ethy"`, value `rlp(0)` = `0x80`) |
| **Companion Protocols** | `eth/71` ([EIP-7642](https://eips.ethereum.org/EIPS/eip-7642), [EIP-7928](https://eips.ethereum.org/EIPS/eip-7928)), Consensus Light Client (`@ethereumjs/beacon-light`); optional Day-0 origin seeder bootstrap via `snap/1` (**zero `snap/1` in default clients**) |
| **Status** | Draft Specification (`v0.4` — Unified Custody Architecture, Amsterdam Era; supersedes standalone [`STATY/0`](./STATY-0-SPEC.md)) |
| **Authors** | EthereumJS Team |

---

## Table of Contents

1. [Executive Summary & Motivation](#1-executive-summary--motivation)
2. [Unified Custody Architecture (`16` Nibble Buckets, `--custody 2` Default, Zero `snap/1`)](#2-unified-custody-architecture-16-nibble-buckets---custody-2-default-zero-snap1)
3. [Cryptographic Trust Model & End-to-End Verification Invariants](#3-cryptographic-trust-model--end-to-end-verification-invariants)
4. [Capability Negotiation, ENR Discovery, Bucket-Aware `ETHY_PEX` & Custody Reciprocity](#4-capability-negotiation-enr-discovery-bucket-aware-ethy_pex--custody-reciprocity)
5. [Wire Encoding Conventions & Binary Data Layouts](#5-wire-encoding-conventions--binary-data-layouts)
6. [Message Specification (`0x00` – `0x0e`)](#6-message-specification-0x00--0x0e)
7. [Open-World Cold-Miss Resolution & BAL-Extended Witness Folding](#7-open-world-cold-miss-resolution--bal-extended-witness-folding)
8. [Live `1/16th` Subtree MPTs, 512-Byte `[h_0..h_f]` Root Assembly & Self-Refreshing Pivots](#8-live-116th-subtree-mpts-512-byte-h_0h_f-root-assembly--self-refreshing-pivots)
9. [Day-0 Origin Bootstrap, Sync Selection & Swarm Convergence](#9-day-0-origin-bootstrap-sync-selection--swarm-convergence)
10. [Worked Byte-Level Walkthroughs](#10-worked-byte-level-walkthroughs)
11. [Integrator Guide (For Human Engineers & AI Coding Agents)](#11-integrator-guide-for-human-engineers--ai-coding-agents)

---

## 1. Executive Summary & Motivation

With the introduction of **Block-Level Access Lists ([EIP-7928](https://eips.ethereum.org/EIPS/eip-7928) `BAL` on `eth/71`)** in the Amsterdam hardfork and **Consensus Light Clients (`Altair` / `Electra` / `Amsterdam` Sync Committee BLS verification)**, Ethereum nodes no longer need to re-execute every transaction or maintain an **80+ GB full-world Merkle Patricia Trie (`MPT`) database** to verify Ethereum state.

However, two structural failure modes remain if flat-state or light clients are designed naively:

1. **Full World State (`~160+ GB` on Mainnet) Is Heavy for Default Laptop Onboarding:** Replicating 100% of Ethereum accounts, bytecodes, and storage slots takes hours and ~160+ GB of disk before a node is usable.
2. **Ad-Hoc `snap/1` Light Clients Do Not Decouple From Full Nodes:** Clients that stay stateless and pull individual accounts/slots on demand from `snap/1` full nodes (`GetAccountRange` / `GetStorageRanges` / `GetTrieNodes`) permanently consume `snap/1` peer slots, cannot regenerate fresh proofs on their own, and create a free-rider asymmetry between light consumers and full-node seeders.

### What Unified Custody `ETHY/0` (`v0.4`) Solves

**`ETHY/0` (`["ethy", 0]`, `length = 15`)** unifies flat-state chunk seeding, 1-hop on-demand state witnesses, and `1/16th` live subtree MPT proof generation into **one symmetric, custody-sharded DevP2P protocol** that cuts Ethereum's dependency on `snap/1` full nodes completely:

- **One Protocol, One Custody Dial (`--custody <1..16>`, default `2`):**
  - Ethereum's state trie naturally partitions into **16 disjoint nibble buckets (`0x0..0xf` on `accountHash[0] >> 4`)**.
  - Every default `ETHY/0` node boots **live in 2–3 minutes** by syncing a rolling `eth/71` BAL window (`4,096..8,192` blocks), then custodies **`2/16` (`1/8th`) of Ethereum (`~20–26 GB` Mainnet / `~3–4 GB` Sepolia)** across all three planes (`Accounts`, `Bytecode`, `Storage`) in the background — while keeping **zero `snap/1` mounted in `HELLO`**!
  - Holding 2 buckets per node means **only 7 peers** give 100% 1-hop coverage, **`14–20` peers** give **2× fault-tolerant redundancy** across all 16 buckets (zero DHT hops!), and **just 8 Ethy nodes** form a completely self-sustaining swarm.
  - Power users can dial up to **`--full` (`--custody 16`)** for **100% Closed-World read privacy** (zero outbound queries ever). Low-disk devices can use **`--custody 1`**. **`--custody 0` is forbidden** — enforced at both the CLI and wire protocol layers via custody reciprocity (§ 4.4).
- **Two-Stage `snap/1` Independence (Option B → Option C in One Wire Protocol):**
  - **Stage 1 (Flat Custody Chunks — Immediate Zero-`snap/1` Cold Reads):** Custodians store verified flat binary slabs (`136B BinaryAccount`, `64B BinaryStorageSlot`) + compact Merkle boundary proofs (`.proof.idx`) and answer cold misses in **1 hop** via targeted `0x03 GET_PROVEN_ACCOUNT_CHUNK` / `0x05 GET_PROVEN_STORAGE_OR_CODE` + local BAL folding (also granting k-anonymous range privacy!).
  - **Stage 2 (`1/16th` Live Subtree MPT + 512-Byte `[h_0..h_f]` Root Assembly — Permanent Zero-`snap/1` Proofs & Refresh):** Because Ethereum's state trie root is always a 16-way branch node `RootNode = RLP([h_0, h_1, ..., h_f, 0x])` where `h_b` depends **exclusively** on bucket `b`, each custodian maintains a tiny `1/16th` subtree MPT updated solely from `eth/71` BALs (`< 2–5 ms/block`), gossips its 32-byte `h_b` hash (`0x0d`/`0x0e SUBTREE_ROOTS`) to assemble and verify `keccak256(RLP([h_0..h_f, 0x])) === header.stateRoot`, and serves **both `< 1 ms` ~2 KB live MPT witnesses (`0x0b`/`0x0c`) AND fresh self-proving pivot chunks** — with **`0` `snap/1` calls forever**!

---

## 2. Unified Custody Architecture (`16` Nibble Buckets, `--custody 2` Default, Zero `snap/1`)

```text
                       @ethereumjs/beacon-light
                    (trusted headers & stateRoots)
                                  │
               ┌──────────────────┴──────────────────┐
               ▼                                     ▼
     ┌──────────────────┐                  ┌───────────────────────────┐
     │     eth/71       │                  │       ETHY/0 (v0.4)       │
     │ (Standard EL)    │                  │   Unified Custody Swarm   │
     ├──────────────────┤                  ├───────────────────────────┤
     │ • Rolling BAL    │──── BALs feed ──►│ • 16 Nibble Buckets 0..f  │
     │   window (4k-8k) │    both state &  │ • Default --custody 2     │
     │ • Live head sync │    subtree MPTs  │   (1/8th; 7 peers=16/16)  │
     │ • Tx broadcast   │                  │ • 1-Hop Witnesses (0x0b)  │
     │                  │                  │ • Targeted Chunks (0x03)  │
     │                  │                  │ • Code & Storage (0x05)   │
     │                  │                  │ • [h_0..h_f] Gossip (0x0e)│
     └────────┬─────────┘                  └─────────────┬─────────────┘
              │                                          │
              └────────────────────┬─────────────────────┘
                                   ▼
                 Shared FlatStateManager + Shared code/
         ┌─────────────────────────┴─────────────────────────┐
         │ Local Custody Buckets (Closed-World: 0 leaks)     │
         │ Remote Buckets (Open-World: BAL -> Tombstone ->   │
         │   1-Hop ETHY/0 Witness/Chunk + Local BAL Fold)    │
         └───────────────────────────────────────────────────┘
            Default clients NEVER mount or dial snap/1!
```

### 2.1 Custody Tiers & Peer-Count Scaling

| Mode | Custody Buckets | Mainnet Disk (Est.) | Sepolia Disk (Est.) | Peers for 16/16 (1× / 2×) | Local Privacy | `snap/1` Mounted? |
| --- | --- | --- | --- | --- | --- | --- |
| `--custody 1` (low-disk) | `1 / 16` | ~11–14 GB | ~1.5–2 GB | 15 / 30 | 6.25% local | **No (0 slots)** |
| **`--custody 2` (Default)** | **`2 / 16` (`1/8th`)** | **~20–26 GB** | **~3–4 GB** | **7 / 14** | **12.5% local** | **No (0 slots)** |
| `--custody 4` | `4 / 16` (`1/4th`) | ~40–50 GB | ~6–8 GB | 3–4 / 7–8 | 25% local | **No (0 slots)** |
| `--full` (`--custody 16`) | `16 / 16` (All) | ~160+ GB | ~25 GB | 0 needed | **100% Closed-World** | **No** (unless Day-0 `--snap-bootstrap`) |
| ~~`--custody 0`~~ | **Forbidden** | — | — | — | — | Rejected at CLI & deprioritized on wire (§ 4.4) |

### 2.2 Per-Bucket Open-World vs. Closed-World State Machine (`FlatStateManager`)
A single `FlatStateManager` tracks completeness per nibble bucket `b = accountHash[0] >> 4` (`0..15`) via a 16-bit `closedBucketsMask`:

| Bucket State | Local Lookup Result | Action |
| --- | --- | --- |
| **Closed-World** (`(closedBucketsMask & (1 << b)) !== 0`) | Hit in flat DB | Return local value immediately (`0` network calls, 100% read privacy). |
| **Closed-World** | Miss in flat DB | Return empty account / `0x00..00` immediately (`0` network calls, 100% read privacy). |
| **Open-World** (`(closedBucketsMask & (1 << b)) === 0`) | Hit in flat DB (from rolling BALs or cached verified lookup) | Return local value immediately (`0` network calls). |
| **Open-World** | **Proven-Empty Tombstone** (verified MPT exclusion proof or `storageRoot === KECCAK256_RLP`, untouched in BALs since anchor) | Return empty account / `0x00..00` immediately (`0` network calls). |
| **Open-World** | **Unknown Miss** | Resolve in 1 hop from a bucket-`b` `ETHY/0` custodian (`0x0b GET_STATE_WITNESS` → `0x03`/`0x05` targeted chunk + local BAL fold). **If unresolved, throw `StateMissUnavailableError` (JSON-RPC `-32000`) — NEVER fabricate a false `0x0` balance or nonce!** |

As each assigned custody bucket `b` finishes syncing all three planes (`Accounts`, `Bytecode`, `Storage`) and BAL-replaying to `#Head`, bucket `b` flips atomically from **Open-World** to **Closed-World** (`closedBucketsMask |= (1 << b)`).

### 2.3 Shared Binary Block Calendar
All BAL retention, anchor root gossip, and snapshot generation horizons align on powers of two:

| Constant | Blocks | Wall Time (~12s/blk) | Purpose |
| --- | --- | --- | --- |
| `CANONICAL_PIVOT_GRID_BLOCKS` (`K`) | `32` | ~6.4 min | Fine pivot grid (1 Beacon Epoch) & fast anchor interval |
| `ANCHOR_EPOCH_BLOCKS` (`E`) | `256` | ~51.2 min | Canonical `[h_0..h_f]` root-branch assembly cadence |
| `MACRO_GENERATION_GRID_BLOCKS` (`G`) | `1,024` | ~3.4 h | Self-refreshing pivot chunk generation cadence |
| `ONBOARDING_BAL_WINDOW_BLOCKS` (`D`) | `4,096` | ~13.6 h | Default fresh-node rolling BAL window |
| `BAL_RETENTION_WINDOW_BLOCKS` (`2D`) | `8,192` | ~27.3 h | Default disk BAL archive & witness/chunk validity horizon |

### 2.4 On-Disk Storage Layout
```text
<networkDir>/
├── bals/                           # Verified contiguous (headerRlp, balRlp) rolling archive (4k-8k blks)
├── code/                           # Shared immutable content-addressed bytecode store (keccak256(code))
├── witnesses/                      # Bounded LRU cache of verified 0x0c STATE_WITNESS proofs
├── write-index/                    # Strictly local BAL Write-Index checkpoints (never wire-trusted)
├── tries/<0..f>/                   # Compact 1/16th live subtree MPTs for assigned custody buckets (Option C)
└── pivots/<basePivotStateRoot>/
    ├── accounts/<0..f>.bin + .proof.idx
    └── storage/<0..f>.bin  + .proof.idx
```

---

## 3. Cryptographic Trust Model & End-to-End Verification Invariants

An `ETHY/0` node treats every remote peer as an **untrusted data mule**. Every data-carrying message MUST pass its exact cryptographic predicate before committing to state or disk:

| Message | Mandatory Verification Predicate | On Failure |
| --- | --- | --- |
| **`0x08 BAL_RANGE`** | 1. `header.number === blockNumber` and `keccak256(headerRlp) === child.parentHash` linking up to `@ethereumjs/beacon-light` (or verified P2P head).<br>2. `keccak256(balRlp) === header.blockAccessListHash`. | Immediate disconnect (`SUBPROTOCOL_ERROR`) + peer quarantine. |
| **`0x04 PROVEN_ACCOUNT_CHUNK`** | 1. `(chunkBlockNumber, chunkStateRoot)` matches verified header at `chunkBlockNumber` (`localBalEarliestBlock <= chunkBlockNumber <= bestVerifiedHead` or bridged by BALs).<br>2. `binaryAccountsSlab.length % 136 === 0`; `accountHash` keys strictly ascending within `[originHash .. lastHash]`.<br>3. `verifyMerkleRangeProof(chunkStateRoot, originHash, lastHash, accountHashes, accountRlpValues, proofNodes)` succeeds.<br>4. If `chunkBlockNumber < targetBlock`, replay local verified BALs `(chunkBlockNumber + 1 .. targetBlock)` restricted to `[originHash .. lastHash]`. | Immediate disconnect (`SUBPROTOCOL_ERROR`) + peer quarantine. |
| **`0x06 PROVEN_STORAGE_OR_CODE`** (`kind = 0`) | `code.length <= MAX_CONTRACT_CODE_SIZE` and `keccak256(code)` matches a `codeHash` explicitly requested in `0x05`. | Immediate disconnect (`SUBPROTOCOL_ERROR`) + peer quarantine. |
| **`0x06 PROVEN_STORAGE_OR_CODE`** (`kind = 1`) | 1. `slotsSlab64B.length % 64 === 0`; `storageRoot` matches verified account's `storageRoot` at `chunkBlockNumber`.<br>2. `slotHash` keys strictly ascending within `[originSlotHash .. lastSlotHash]`; no all-zero `slotValueBE`.<br>3. `verifyMerkleRangeProof(storageRoot, originSlotHash, lastSlotHash, slotHashes, RLP(unpad(slotVal)), proofNodes)` succeeds.<br>4. Fold local verified BALs `(chunkBlockNumber + 1 .. #Head)`. | Immediate disconnect (`SUBPROTOCOL_ERROR`) + peer quarantine. |
| **`0x06 PROVEN_STORAGE_OR_CODE`** (`kind = 2`) | 1. Each `(accountHash, storageRoot)` was requested and matches a verified account.<br>2. `slotsSlab64B.length % 64 === 0`; strictly ascending `slotHash`; no zero values.<br>3. Rebuilding the complete storage MPT directly from `slotsSlab64B` yields `computedRoot === storageRoot` (or empty slab with `storageRoot === KECCAK256_RLP`). | Immediate disconnect (`SUBPROTOCOL_ERROR`) + peer quarantine. |
| **`0x0c STATE_WITNESS`** | 1. For every anchor group: `localBalEarliestBlock <= anchorBlock <= bestVerifiedHead` and `anchorStateRoot === verifiedHeader(anchorBlock).stateRoot`.<br>2. `proofNodes.length <= 4,096` and total decoded frame `<= 2 MiB`.<br>3. `accounts[]` strictly ascending by `accountHash`; `slots[]` strictly ascending by `(accountHash, slotHash)`; `slotValOrEmpty` canonical minimal BE (`<= 32` bytes, no leading `0x00`).<br>4. Build ONE shared read-only `Map<keccak256(nodeRlp), nodeRlp>` from `proofNodes[]`; verify every account leaf/exclusion against `anchorStateRoot` and every storage leaf/exclusion against its proven parent `storageRoot` at the **same** `anchorBlock`.<br>5. Fold strictly local verified BALs `(anchorBlock + 1 .. #Head)` (§ 7.2). | Immediate disconnect (`SUBPROTOCOL_ERROR`) + peer quarantine. |
| **`0x0e SUBTREE_ROOTS`** | 1. `entries.length <= 16`; `bucketIndex` in `0..15`, strictly ascending and unique; each `subtreeRootHash` length `32` or `0` (empty nibble).<br>2. Once all 16 nibble hashes `[h_0..h_f]` are collected for `anchorBlock`, verify:<br>`keccak256(RLP.encode([h_0, h_1, ..., h_f, 0x])) === verifiedHeader(anchorBlock).stateRoot`.<br>3. Any peer whose announced `h_b` disagrees with the consensus-proven `RootNode[b]` is proven Byzantine! | Disconnect (`SUBPROTOCOL_ERROR`) + quarantine Byzantine peer(s). |

**Empty / Miss Responses:** Returning an empty list or empty slab (`0x80`) when a peer does not hold a requested key/chunk/anchor is a valid miss and MUST NOT penalize the peer.

---

## 4. Capability Negotiation, ENR Discovery, Bucket-Aware `ETHY_PEX` & Custody Reciprocity

### 4.1 RLPx `HELLO` Capability Negotiation
- Subprotocol name: **`"ethy"`** (lowercase ASCII, 4 bytes: `0x65746879`).
- Subprotocol version: **`0`** (encoded canonically as `intToUnpaddedBytes(0)` = `0x80` in `HELLO`).
- Subprotocol message-ID space length: **`15`** (`0x00` through `0x0e`).
- Standard Ethereum clients (`Geth`, `Reth`, `Nethermind`, `Besu`, `Erigon`) silently ignore `["ethy", 0]` during `HELLO` intersection and mount `eth/71` normally.
- **Zero `snap/1` by Default:** Default `ETHY/0` clients MUST NOT include `["snap", 1]` in `HELLO`. Only an explicit Day-0 origin seeder (`--snap-bootstrap`) mounts `snap/1` until initial state finishes, then drops it permanently.

### 4.2 EIP-778 ENR Discovery Key (`"ethy"`)
Nodes actively serving `ETHY/0` (`bucketsBitmask !== 0`) SHOULD include the key `"ethy"` in their signed [EIP-778](https://eips.ethereum.org/EIPS/eip-778) ENR:
- **Key:** `"ethy"` (`0x65746879`)
- **Value:** `RLP.encode(0)` = `0x80` (highest supported version `0`).
- Nodes with `bucketsBitmask === 0` (still bootstrapping) SHOULD NOT advertise `"ethy"` in their ENR until their first custody bucket is verified and servable.

### 4.3 Bucket-Aware Epidemic Peer Exchange (`ETHY_PEX`, `0x09`/`0x0a`) & Bidirectional NAT Serving
To achieve **1-hop `16/16` coverage with 2× redundancy inside `14–20` peers** (zero DHT required):
1. Immediately after `ETHY/0 STATUS` (and periodically whenever any bucket lacks redundant custodians), a node sends `0x09 GET_ETHY_PEERS` with its `desiredBucketsMask` (the 16-bit mask of uncovered or under-redundant buckets).
2. When answering `0x09 GET_ETHY_PEERS`, the responder MUST:
   - **Exclude all peers with `bucketsBitmask === 0`** (non-custodians are never propagated in `ETHY_PEX`).
   - Fill ~75% of returned slots with active custodians matching `(peer.bucketsBitmask & desiredBucketsMask) !== 0` (ranking `trieBucketsBitmask` matches highest, then full 3-plane storage+code matches, then freshest `basePivotBlock`).
   - Fill ~25% of slots with random active custodians for mesh connectivity and anti-eclipse diversity.
3. **Bidirectional NAT Serving:** Once an `RLPx` TCP connection is open, `ETHY/0` is 100% symmetric — a home laptop behind NAT making outbound connections serves chunks, witnesses, bytecodes, BALs, and `h_b` roots over those outbound sockets without port forwarding.

### 4.4 Protocol-Level Custody Reciprocity (Anti-Free-Rider Enforcement)
Because `--custody 0` is forbidden and healthy swarms depend on symmetric custody contribution, `ETHY/0` servers enforce two-tier reciprocity based on the remote peer's verified `STATUS.bucketsBitmask`:

| Remote Peer Tier | Identification | Serving & Slot Policy |
| --- | --- | --- |
| **Active Custodian** | `bucketsBitmask !== 0` (serving `>= 1` bucket) | Full priority on `0x0b GET_STATE_WITNESS`, `0x03`/`0x05` reads, `0x0d` root queries; protected inbound `ETHY/0` peer slots; eligible for `ETHY_PEX` advertisement. |
| **Bootstrap Joiner / Non-Custodian** | `bucketsBitmask === 0` | Bounded low-priority quota for bulk onboarding (`0x01`/`0x03`/`0x05`) so fresh nodes can always bootstrap cleanly; **strictly throttled on ad-hoc `0x0b GET_STATE_WITNESS`**; **excluded from `ETHY_PEX`**; **evicted first** when inbound peer slots fill up. |

---

## 5. Wire Encoding Conventions & Binary Data Layouts

### 5.1 Canonical RLP Scalar Rule
All `ETHY/0` messages are RLP-encoded arrays compressed with Snappy (`RLPx` v5).
Every unsigned integer scalar (`uint8`, `uint16`, `uint32`, `uint64`, `uint256`) MUST use **canonical minimal big-endian encoding**:
- Strip all leading `0x00` bytes.
- Integer `0` MUST be encoded as the empty byte string `0x` (RLP byte `0x80`), **never** `0x00`.
- **Global Wire DoS Cap:** Receivers MUST reject any decompressed `ETHY/0` message exceeding `MAX_ETHY_MESSAGE_BYTES = 2 * 1024 * 1024` (2 MiB) and disconnect (`SUBPROTOCOL_ERROR`).

### 5.2 Fixed 136-Byte `BinaryAccount` Slab Row Layout (`binaryAccountsSlab`)
Used in `0x04 PROVEN_ACCOUNT_CHUNK` (`binaryAccountsSlab.length % 136 === 0`):

```text
Offset   Size    Field          Encoding
──────   ────    ─────────────  ───────────────────────────────────────────────────────
0..32    32 B    accountHash    keccak256(address) — strictly ascending across slab
32..40    8 B    nonce          uint64 big-endian (fixed 8 bytes)
40..72   32 B    balance        uint256 big-endian (fixed 32 bytes)
72..104  32 B    storageRoot    32-byte MPT storage root (KECCAK256_RLP if empty)
104..136 32 B    codeHash       32-byte bytecode hash (KECCAK256_NULL if EOA)
──────   ────
Total:  136 B per account row
```

### 5.3 Fixed 64-Byte `BinaryStorageSlot` Slab Row Layout (`slotsSlab64B`)
Used in `0x06 PROVEN_STORAGE_OR_CODE` (`kind = 1` and `kind = 2`; `slotsSlab64B.length % 64 === 0`):

```text
Offset   Size    Field          Encoding
──────   ────    ─────────────  ───────────────────────────────────────────────────────
0..32    32 B    slotHash       keccak256(slot32) — strictly ascending across slab
32..64   32 B    slotValueBE    uint256 big-endian (fixed 32 bytes; MUST be non-zero)
──────   ────
Total:   64 B per storage slot row
```

Stripping leading `0x00` bytes from `slotValueBE` and applying `RLP.encode(unpad(slotValueBE))` is a 100% lossless bijection with the canonical Ethereum storage MPT leaf value.

### 5.4 Deduplicated Multi-Proof Dictionary (`proofNodes: [bytes, ...]`)
Used in `0x0c STATE_WITNESS` (and boundary proofs in `0x04` / `0x06`):
- Contains each unique RLP-encoded MPT node (`branch`, `extension`, or `leaf`) **at most once**, covering both the state trie (`anchorStateRoot`) and all referenced storage tries (`storageRoot`).
- Under Option C, `proofNodes[0]` is the verified 530-byte `RootNode(#A) = RLP([h_0..h_f, 0x])`, followed by deduplicated subtree nodes from `BucketSubtreeTrie(b)`.
- **DoS Cap:** `proofNodes.length <= MAX_PROOF_NODES_PER_MSG` (`4,096`).

---

## 6. Message Specification (`0x00` – `0x0e`)

| Code | Name | Direction | Description |
| --- | --- | --- | --- |
| `0x00` | `STATUS` | Peer ↔ Peer | Handshake + live updates advertising pivot, per-plane 16-bit bitmasks (`bucketsBitmask`, `storageBucketsBitmask`, `codeBucketsBitmask`, `trieBucketsBitmask`), and BAL range. |
| `0x01` | `GET_BUCKET_MANIFEST` | Client → Server | Request ordered self-proving chunk descriptors for account bucket `0x0..0xf`. |
| `0x02` | `BUCKET_MANIFEST` | Server → Client | Returns chunk manifest for the requested bucket. |
| `0x03` | `GET_PROVEN_ACCOUNT_CHUNK` | Client → Server | Request a chunk by exact `originHash` OR targeted single-key lookup (`originHash <= targetHash <= lastHash`). |
| `0x04` | `PROVEN_ACCOUNT_CHUNK` | Server → Client | Returns 136B-packed `binaryAccountsSlab`, `proofNodes`, `chunkStateRoot`, and `chunkBlockNumber`. |
| `0x05` | `GET_PROVEN_STORAGE_OR_CODE` | Client → Server | Request bytecode batch (`kind = 0`), large-contract storage chunk (`kind = 1`), or batched complete-trie small contracts (`kind = 2`). |
| `0x06` | `PROVEN_STORAGE_OR_CODE` | Server → Client | Returns verified code blobs, 64B storage range + proof, or batched small-contract slabs. |
| `0x07` | `GET_BAL_RANGE` | Client → Server | Request up to `32` contiguous `(blockNumber, headerRlp, balRlp)` bundles starting at `startBlock`. |
| `0x08` | `BAL_RANGE` | Server → Client | Returns contiguous `(blockNumber, headerRlp, balRlp)` bundles. |
| `0x09` | `GET_ETHY_PEERS` | Peer → Peer | Request up to `maxPeers` active custodians (`ETHY_PEX`), optionally filtered by `desiredBucketsMask`. |
| `0x0a` | `ETHY_PEERS` | Peer → Peer | Returns custodian peer descriptors (never `bucketsBitmask === 0`). |
| `0x0b` | `GET_STATE_WITNESS` | Client → Server | Request 1-hop MPT inclusion/exclusion witnesses for `(accountHash, slotHashes[])` at `anchorBlock >= minAnchorBlock`. |
| `0x0c` | `STATE_WITNESS` | Server → Client | Returns verified/generated MPT witnesses grouped by `(anchorBlock, anchorStateRoot)` with deduplicated `proofNodes[]`. |
| `0x0d` | `GET_SUBTREE_ROOTS` | Peer → Peer | Request known 32-byte nibble subtree roots `h_b` for `anchorBlock` matching `bucketsBitmask`. |
| `0x0e` | `SUBTREE_ROOTS` | Peer ↔ Peer | Response (`reqId > 0`) or unsolicited gossip (`reqId = 0`) announcing `[[bucketIndex, subtreeRootHash32], ...]` at `anchorBlock`. |

---

### `0x00` `STATUS` (`v0.4` 12-Tuple)

Sent immediately on subprotocol mount and **re-sent whenever any bucket bitmask, `basePivotBlock`, or `balLatestBlock` changes**.

```text
[
  protocolVersion: uint,          // 0 (encoded as 0x80)
  chainId: uint,                  // e.g. 1 (Mainnet), 11155111 (Sepolia)
  genesisHash: bytes32,           // 32-byte execution genesis block hash
  basePivotBlock: uint64,         // Block number #C of the proven base snapshot (0 if none)
  basePivotStateRoot: bytes32,    // 32-byte stateRoot at #C (32 zero bytes if none)
  bucketsBitmask: uint16,         // Bit b (1 << b) is 1 iff account bucket b (0..15) is 100% proven & servable
  balEarliestBlock: uint64,       // Lowest contiguous block with servable (headerRlp + balRlp)
  balLatestBlock: uint64,         // Highest contiguous block with servable (headerRlp + balRlp)
  latestBlockHash: bytes32,       // 32-byte block hash at balLatestBlock (or 32 zero bytes)
  // --- v0.3 trailing per-plane completeness fields ---
  storageBucketsBitmask: uint16,  // Optional (default 0): bit b is 1 iff 100% of storage in bucket b is servable
  codeBucketsBitmask: uint16,     // Optional (default 0): bit b is 1 iff 100% of distinct codeHashes in bucket b are servable
  // --- v0.4 trailing live subtree-trie capability field ---
  trieBucketsBitmask: uint16      // Optional (default 0): bit b is 1 iff peer runs a live 1/16th subtree MPT for bucket b
]
```

**Validation & Backward Compatibility:**
- Disconnect (`SUBPROTOCOL_ERROR`) if `protocolVersion !== 0`, `chainId` mismatches, or `genesisHash` mismatches.
- Decoders MUST accept `payload.length >= 9`, defaulting omitted indices `9`, `10`, and `11` to `0`, and ignoring extra trailing fields beyond index `11`.
- Repeat `STATUS` messages MUST be accepted and update live peer metadata.
- A peer MUST set bit `b` in `trieBucketsBitmask` **only** after verifying its local `h_b(#A) === RootNode(#A)[b]` for a recent canonical anchor block!

---

### `0x01` `GET_BUCKET_MANIFEST` & `0x02` `BUCKET_MANIFEST`

**Request (`0x01`):** `[ requestId: uint64, bucketIndex: uint8 (0..15) ]`

**Response (`0x02`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8,
  chunks: [
    [
      originHash: bytes32,
      lastHash: bytes32,
      hasRight: uint8,            // 0 or 1
      chunkStateRoot: bytes32,
      chunkBlockNumber: uint64,
      accountCount: uint32,
      slabByteLength: uint32      // accountCount * 136
    ],
    ...
  ]
]
```
Miss response: `chunks = []`.

---

### `0x03` `GET_PROVEN_ACCOUNT_CHUNK` & `0x04` `PROVEN_ACCOUNT_CHUNK`

Supports **both** bulk bucket sync (exact `originHash` from `0x02 BUCKET_MANIFEST`) **and** 1-RTT Open-World targeted key lookup (Option B)!

**Request (`0x03`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8,             // 0..15
  targetOrOriginHash: bytes32     // Exact chunk originHash OR any target accountHash in bucketIndex
]
```

**Server Lookup Rule (`v0.4`):**
1. If a chunk in `bucketIndex` starts at exact `chunk.originHash === targetOrOriginHash`, return that chunk.
2. Otherwise, return the **unique** chunk in `bucketIndex` covering `targetOrOriginHash`:
   - `chunk.originHash <= targetOrOriginHash <= chunk.lastHash`, OR
   - The first chunk if `targetOrOriginHash < firstChunk.originHash` (proving non-existence at the left boundary of the bucket), OR
   - The last chunk if `targetOrOriginHash > lastChunk.lastHash && lastChunk.hasRight === 0` (proving non-existence at the right boundary of the bucket).
3. If `bucketIndex` is unavailable, reply with `binaryAccountsSlab = 0x80` and `proofNodes = []`.

**Response (`0x04`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8,
  originHash: bytes32,            // Actual chunk originHash (may be <= targetOrOriginHash)
  lastHash: bytes32,
  hasRight: uint8,                // 0 or 1
  chunkStateRoot: bytes32,
  chunkBlockNumber: uint64,
  binaryAccountsSlab: bytes,      // N * 136 bytes (§ 5.2; 0x80 on miss)
  proofNodes: [bytes, ...]        // RLP-encoded MPT boundary proof nodes
]
```

---

### `0x05` `GET_PROVEN_STORAGE_OR_CODE` & `0x06` `PROVEN_STORAGE_OR_CODE`

Transfers shared contract bytecodes (`kind = 0`), chunked large-contract storage ranges (`kind = 1`), or batched complete-trie small contracts (`kind = 2`).

**Request (`0x05`):**
- `kind = 0` (Bytecode Batch): `[ requestId, 0 (0x80), codeHashes: [bytes32, ...] (1..64), maxResponseBytes (<= 524,288) ]`
- `kind = 1` (Large-Contract Storage Range / Targeted Slot Chunk): `[ requestId, 1, accountHash: bytes32, storageRoot: bytes32, targetOrOriginSlotHash: bytes32, maxResponseBytes (<= 524,288) ]`
  - Matching `0x03`, if no stored chunk starts at exact `targetOrOriginSlotHash`, the server returns the covering storage chunk satisfying `originSlotHash <= targetOrOriginSlotHash <= lastSlotHash` (or boundary chunk proving non-existence).
- `kind = 2` (Batched Complete-Trie Small Contracts): `[ requestId, 2, accounts: [[accountHash: bytes32, storageRoot: bytes32], ...] (1..64), maxResponseBytes (<= 524,288) ]`

**Response (`0x06`):**
- `kind = 0`: `[ requestId, 0 (0x80), codes: [bytes, ...] ]` (misses omitted).
- `kind = 1`: `[ requestId, 1, accountHash, storageRoot, chunkBlockNumber, originSlotHash, lastSlotHash, hasRight, slotsSlab64B, proofNodes ]` (`slotsSlab64B = 0x80` on miss).
- `kind = 2`: `[ requestId, 2, entries: [[accountHash, storageRoot, slotsSlab64B], ...] ]` (misses or multi-chunk contracts omitted so client falls back to `kind = 1`).

---

### `0x07` `GET_BAL_RANGE` & `0x08` `BAL_RANGE`

**Request (`0x07`):** `[ requestId: uint64, startBlock: uint64, count: uint8 (1..32) ]`

**Response (`0x08`):** `[ requestId: uint64, blocks: [[blockNumber: uint64, headerRlp: bytes, balRlp: bytes], ...] ]`

---

### `0x09` `GET_ETHY_PEERS` & `0x0a` `ETHY_PEERS` (`ETHY_PEX`)

**Request (`0x09`):**
```text
[
  requestId: uint64,
  maxPeers: uint8,                // 1..32 (recommended default: 16)
  // --- v0.4 optional trailing bucket-filter mask (backward-compatible with v0.2/v0.3 2-tuple) ---
  desiredBucketsMask: uint16      // Optional (default 0 = any buckets): prioritize custodians covering these bits
]
```

**Response (`0x0a`):**
```text
[
  requestId: uint64,
  peers: [
    [
      peerId: bytes,              // 64-byte uncompressed secp256k1 pubkey (no 0x04 prefix)
      address: bytes,             // UTF-8 IPv4/IPv6 string
      tcpPort: uint16,
      udpPort: uint16,
      basePivotBlock: uint64,
      bucketsBitmask: uint16,     // MUST be non-zero (non-custodians excluded!)
      lastSeenAgeSec: uint32,
      // --- v0.4 optional trailing trie capability mask ---
      trieBucketsBitmask: uint16  // Optional (default 0)
    ],
    ...
  ]
]
```

---

### `0x0b` `GET_STATE_WITNESS` & `0x0c` `STATE_WITNESS`

Requests and returns compact, deduplicated MPT inclusion/exclusion multi-proofs for specific `(accountHash, slotHashes[])` keys anchored at `anchorBlock >= minAnchorBlock`. Servers answer purely from their live `BucketSubtreeTrie` (Option C, `< 1 ms`, ~2 KB) or local `witnesses/` cache (Option B) — **servers MUST NEVER block on upstream `snap/1` queries**.

**Request (`0x0b GET_STATE_WITNESS`):**
```text
[
  requestId: uint64,
  minAnchorBlock: uint64,         // Client's localBalEarliestBlock (reject older anchors)
  maxResponseBytes: uint64,       // Soft cap clamped to <= 524,288
  queries: [                      // 1..64 accounts; total slotHashes across all queries <= 256
    [
      accountHash: bytes32,       // keccak256(address)
      slotHashes: [bytes32, ...]  // 0..256 keccak256(slot32) keys to prove under this account
    ],
    ...
  ]
]
```

**Response (`0x0c STATE_WITNESS`):**
```text
[
  requestId: uint64,
  groups: [                       // Grouped by anchorBlock so proofNodes deduplicate within each anchor
    [
      anchorBlock: uint64,
      anchorStateRoot: bytes32,
      accounts: [                 // Strictly ascending by accountHash
        [
          accountHash: bytes32,
          accountRlpOrEmpty: bytes // Canonical 4-tuple account RLP, OR 0x80 = proven non-existent account
        ],
        ...
      ],
      slots: [                    // Strictly ascending by (accountHash, slotHash)
        [
          accountHash: bytes32,   // Parent account MUST appear in accounts[] of the same group
          slotHash: bytes32,
          slotValOrEmpty: bytes   // Minimal big-endian uint256 (1..32B, no leading 0x00), OR 0x80 = proven zero slot
        ],
        ...
      ],
      proofNodes: [bytes, ...]    // Deduplicated MPT nodes (<= 4,096 nodes) covering accounts[] & slots[]
    ],
    ...
  ]
]
```
- **Miss / Throttle Response:** Unfound keys are omitted cleanly (`groups = []` if nothing matches).

---

### `0x0d` `GET_SUBTREE_ROOTS` & `0x0e` `SUBTREE_ROOTS`

Exchanges the 32-byte nibble subtree root hashes `h_b` (`b in 0..15`) at canonical anchor blocks so every node in the swarm can assemble and cryptographically verify `RootNode(#A) = RLP([h_0..h_f, 0x])` against `verifiedHeader(#A).stateRoot`.

**Request (`0x0d GET_SUBTREE_ROOTS`):**
```text
[
  requestId: uint64,              // Non-zero request ID
  anchorBlock: uint64,            // Canonical anchor block #A (e.g. multiple of 32 or 256)
  bucketsBitmask: uint16          // Bitmask of desired nibble hashes (0 = all known at anchorBlock)
]
```

**Response / Gossip (`0x0e SUBTREE_ROOTS`):**
```text
[
  requestId: uint64,              // Matches 0x0d requestId, OR 0 (0x80) for unsolicited gossip announcement
  anchorBlock: uint64,
  entries: [                      // 0..16 entries, strictly ascending & unique by bucketIndex
    [
      bucketIndex: uint8,         // 0..15
      subtreeRootHash: bytes      // 32-byte keccak256 hash h_b, OR 0x80 if nibble b is empty
    ],
    ...
  ]
]
```

---

## 7. Open-World Cold-Miss Resolution & BAL-Extended Witness Folding

### 7.1 1-Hop Resolution Escalation Ladder (Zero `snap/1` by Default)
When `FlatStateManager` encounters an Unknown Miss on an Open-World key in bucket `b = accountHash[0] >> 4`:

1. **Step 1 — 1-Hop `ETHY/0` Witness (`0x0b GET_STATE_WITNESS`):**
   - Route coalesced misses (`10–25 ms` micro-batch window) to connected peers covering bucket `b` (preferring peers with `trieBucketsBitmask & (1 << b)`).
   - Verify returned `0x0c STATE_WITNESS` against `verifiedHeader(anchorBlock).stateRoot`, fold local BALs `(anchorBlock + 1 .. #Head)`, commit values / proven-empty tombstones, and cache in `witnesses/`.
2. **Step 2 — 1-Hop `ETHY/0` Targeted Custody Chunk (`0x03` / `0x05` — Option B):**
   - If Step 1 did not resolve the key (e.g. custodian serves flat slabs without a live subtree trie yet):
     - **Account miss:** Send `0x03 GET_PROVEN_ACCOUNT_CHUNK(b, accountHash)` to a bucket-`b` custodian with `basePivotBlock >= localBalEarliestBlock`. Verify `verifyMerkleRangeProof`. Presence proves the account at `chunkBlockNumber`; absence within a verified covering range proves non-existence (commit Proven-Empty Tombstone if untouched in local BALs since `chunkBlockNumber`)!
     - **Storage miss:** Once the account's `storageRoot` is known, if `storageRoot === KECCAK256_RLP` immediately commit a Proven-Empty Tombstone; otherwise query `0x05` (`kind = 2` or targeted `kind = 1`) from a custodian with `storageBucketsBitmask & (1 << b)`, verify proof, and fold local BALs.
3. **Step 3 — Honest Failure (No `snap/1` Fallback on Default Clients):**
   - If no connected custodian covers bucket `b`, default clients MUST NOT dial `snap/1`. `FlatStateManager` throws `StateMissUnavailableError` → JSON-RPC returns `-32000` (never a fabricated `0x0`).
   - Only when a client was explicitly launched with `--snap-bootstrap` AND the caller's policy is `'full'` (never `'swarm-only'` autonomous AI probes) may it fall back to `snap/1`.

### 7.2 Strictly Local BAL Write-Index & Per-Field BAL Folding
> **Critical Security Invariant:** A node MUST build its BAL Write-Index (`lastWriteBlock` for accounts and `keccak256(slot32)` storage slots) **exclusively from locally verified `(headerRlp, balRlp)` bundles** — never from remote peers.

Given a cryptographically verified account/slot state at `anchorBlock` (`localBalEarliestBlock <= anchorBlock <= #Head`):
1. **Per-Field Account Folding (EIP-7928):** Starting from proven `(nonce_B, balance_B, storageRoot_B, codeHash_B)` (or empty account if excluded) at `anchorBlock`, fold local verified BALs across `[anchorBlock + 1 .. #Head]` independently per field (`balanceChanges`, `nonceChanges`, `codeChanges`, respecting EIP-161 deletion) so a balance-only BAL preserves proven `nonce_B` and `codeHash_B`.
2. **Storage Slot Folding (EIP-6780 Invariance):** If `lastSlotWriteBlock(accountHash, slotHash) > anchorBlock`, the verified BAL `postValue` already in `FlatStateManager` wins (NEVER overwrite live state with an older proof!). Otherwise, the proven value (or proven-empty tombstone) at `anchorBlock` is **cryptographically valid at `#Head`** — even if the contract's `storageRoot` mutated thousands of times on unrelated slots since `anchorBlock`!

---

## 8. Live `1/16th` Subtree MPTs, 512-Byte `[h_0..h_f]` Root Assembly & Self-Refreshing Pivots

### 8.1 Why `1/16th` Subtree Tries Eliminate `snap/1` Permanently
In Ethereum's 16-way Hexary Merkle Patricia Trie, the top node at `header.stateRoot` is always a 17-element branch node:

```text
RootNode(#A) = RLP([ h_0, h_1, h_2, ..., h_b, ..., h_f, 0x ])    (~530 bytes)
keccak256(RootNode(#A)) === header(#A).stateRoot
```

Crucially, **`h_b` depends exclusively on accounts and contract storage slots whose `accountHash[0] >> 4 === b`**. Zero writes outside bucket `b` ever affect `h_b`!

- **Incremental BAL Maintenance (`< 2–5 ms/block`):** On every verified `eth/71` BAL block `#H`, a custodian holding bucket `b` filters BAL writes to `keccak256(address)[0] >> 4 === b` (~1/16th of block writes), updates the local storage MPT for touched contracts in bucket `b` to obtain exact post-block `storageRoot`s, and updates bucket `b`'s account subtree MPT to obtain exact `h_b(#H)` — with **zero EVM execution** and **zero `snap/1`**.
- **512-Byte Root Assembly & Byzantine Quarantine:** At every canonical anchor block `#A` (e.g. `E = 256` blocks), custodians broadcast `(anchorBlock = #A, [[b, h_b(#A)]])` via `0x0e SUBTREE_ROOTS` and pull missing nibbles via `0x0d GET_SUBTREE_ROOTS`. Every node assembles `RootNode(#A) = RLP([h_0..h_f, 0x])` and checks:
  ```text
  keccak256(RootNode(#A)) === verifiedHeader(#A).stateRoot
  ```
  Because `keccak256` is collision-resistant, **it is impossible to forge a valid `RootNode(#A)`**. If conflicting `h_b` candidates exist for a nibble, the unique valid combination identifies and immediately **quarantines** any peer that gossiped a forged `h_b`.
- **Continuous Self-Audit:** Every custodian checks its local `h_b(#A) === RootNode(#A)[b]`. A match cryptographically proves zero local state or BAL drift and unlocks bit `(1 << b)` in `STATUS.trieBucketsBitmask`. A mismatch immediately clears `trieBucketsBitmask` bit `b` so corrupt proofs are never served.

### 8.2 Live `< 1 ms` Witness Serving & Self-Refreshing Macro-Generation Chunks
Once `RootNode(#A)` is verified:
1. **Live `0x0c STATE_WITNESS`:** Custodian `b` generates standard Ethereum MPT inclusion/exclusion proofs from its `BucketSubtreeTrie(b)` at anchor `#A`, prepends `RootNode(#A)` as `proofNodes[0]`, deduplicates nodes, and replies in `< 1 ms` (~2 KB) — 100% verifiable against `header(#A).stateRoot`!
2. **Zero-`snap/1` Pivot Chunk Refresh:** At every macro-generation boundary (`G = 1,024` / `D = 4,096`), each custodian exports brand-new self-proving flat chunks (`accounts/<b_hex>.bin` + `.proof.idx` with `RootNode(#A)` prepended to boundary proofs, and `storage/<b_hex>.bin` + `.proof.idx`) directly from its `BucketSubtreeTrie(b)`, verifies them locally, and atomically advances `basePivotBlock = #A` in `ETHY/0 STATUS`!

---

## 9. Day-0 Origin Bootstrap, Sync Selection & Swarm Convergence

### 9.1 Day-0 → Steady-State Swarm Lifecycle
1. **Day-0 Origin Seeder (`1` `--full --snap-bootstrap` node):**
   - To launch `ETHY/0` on a network with zero existing `ETHY/0` peers, a single operator runs `ethy --full --snap-bootstrap`.
   - Once synced, that `--full` node holds all 16 buckets + all 16 subtree tries, computes all 16 `[h_0..h_f]` locally with **zero `snap/1`**, broadcasts `0x0e SUBTREE_ROOTS`, drops `snap/1` from `HELLO` permanently, and seeds all joining `--custody 2` nodes over `ETHY/0`.
2. **Early Growth (`2–7` Nodes):**
   - New default `--custody 2` nodes mount **zero `snap/1`**, sync rolling `eth/71` BALs (live in 2–3 min), pull their 2 custody buckets from the origin seeder (or peers), and begin serving their 2 buckets + `h_b` roots immediately.
3. **Self-Sustaining Threshold (`8+` `--custody 2` Nodes):**
   - Once all 16 nibble buckets are covered by independent `--custody 2` custodians, **even if the original Day-0 `--full` seed node shuts down**, the swarm continues assembling `RootNode(#A)`, answering 100% of cold reads in 1 hop, and refreshing macro-generation pivot chunks **forever with `0` `snap/1` calls**.

### 9.2 Rarest-First Striping, Cross-Snapshot Composition & Contiguous BAL Serving
- **Rarest-First Bucket Striping:** Multi-bucket joiners prioritize buckets with lowest seeder availability (breaking ties by `keccak256(localNodeId || bucketIndex)`) to avoid flash-crowd contention on bucket `0x0`.
- **Contiguous BAL Bridge Guarantee:** A seeder MUST honestly advertise its true contiguous `balEarliestBlock` and MUST NOT be selected as a standalone chunk source if `balEarliestBlock > basePivotBlock + 1` unless another connected peer (or local `eth/71` BAL archive) bridges `[basePivotBlock + 1 .. #Head]`.
- **Cross-Snapshot Composition:** Because buckets `0x0..0xf` are disjoint and every chunk carries `(chunkBlockNumber, chunkStateRoot)`, a joiner MAY source different buckets from seeders at different pivot generations as long as verified BALs bridge `min(chunkBlockNumber) + 1 .. #Head`.

---

## 10. Worked Byte-Level Walkthroughs

### 10.1 `0x00 STATUS` (`v0.4` 12-Tuple — `--custody 2` Node Serving Buckets `0x2` and `0xa` with Live Subtree Tries)
```ts
[
  new Uint8Array(0),                      // protocolVersion = 0 (canonical 0x80)
  bigIntToUnpaddedBytes(11155111n),       // chainId = Sepolia (0xaa36a7)
  genesisHash32,                          // 32 bytes
  bigIntToUnpaddedBytes(12062720n),       // basePivotBlock (multiple of 256)
  basePivotStateRoot32,                   // 32 bytes
  intToUnpaddedBytes(0x0404),             // bucketsBitmask: bits 2 and 10 (0x0404)
  bigIntToUnpaddedBytes(12058624n),       // balEarliestBlock (4,096-block archive)
  bigIntToUnpaddedBytes(12062800n),       // balLatestBlock
  latestBlockHash32,                      // 32 bytes
  intToUnpaddedBytes(0x0404),             // storageBucketsBitmask: bits 2 and 10
  intToUnpaddedBytes(0x0404),             // codeBucketsBitmask: bits 2 and 10
  intToUnpaddedBytes(0x0404),             // trieBucketsBitmask: live 1/16th subtree tries ready!
]
```

### 10.2 1-Hop Cold Read via `0x0b GET_STATE_WITNESS` → `0x0c STATE_WITNESS`
1. **Node B** needs `(accountHash, [slotHash])` where `accountHash[0] >> 4 === 0x2` and its local BAL window starts at `12,058,624`.
2. Node B sends **`0x0b GET_STATE_WITNESS`** to Node A (whose `trieBucketsBitmask & 0x0004 !== 0`):
   ```ts
   [
     bigIntToUnpaddedBytes(42n),           // requestId
     bigIntToUnpaddedBytes(12058624n),     // minAnchorBlock
     intToUnpaddedBytes(524288),           // maxResponseBytes (512 KiB)
     [[accountHash32, [slotHash32]]],      // queries
   ]
   ```
3. **Node A** reads its `BucketSubtreeTrie(0x2)` at anchor `#12,062,720`, prepends verified `RootNode(#12,062,720)`, deduplicates `proofNodes[]` (~2 KB), and returns **`0x0c STATE_WITNESS`**.
4. **Node B** runs single-pass `verifyStateMultiProof` against `verifiedHeader(12062720).stateRoot`, folds local BALs `12,062,721 .. 12,062,800` via its local `BalWriteIndex`, and commits the consensus-proven value (or Proven-Empty Tombstone) to `FlatStateManager` — **total RTT ~30 ms, `0` `snap/1` calls**.

### 10.3 `0x0e SUBTREE_ROOTS` Unsolicited Anchor Gossip
```ts
[
  new Uint8Array(0),                      // requestId = 0 (unsolicited gossip; canonical 0x80)
  bigIntToUnpaddedBytes(12062720n),       // anchorBlock #A
  [
    [intToUnpaddedBytes(2), h2_bytes32],  // bucket 0x2 -> 32-byte h_2(#A)
    [intToUnpaddedBytes(10), hA_bytes32], // bucket 0xa -> 32-byte h_a(#A)
  ],
]
```

---

## 11. Integrator Guide (For Human Engineers & AI Coding Agents)

| Invariant | Requirement |
| --- | --- |
| **1. Canonical RLP Zero (`0x80`)** | Never encode integer `0` (`protocolVersion`, `bucketIndex = 0`, `hasRight = 0`, `kind = 0`, gossip `requestId = 0`, `desiredBucketsMask = 0`) as `0x00`. Always use `new Uint8Array(0)` → RLP `0x80`. |
| **2. Snappy Framing & 2 MiB Cap** | All payloads on `RLPx` v5 are Snappy-compressed; reject any decompressed frame `> 2 MiB` or `proofNodes.length > 4,096` immediately with `SUBPROTOCOL_ERROR`. |
| **3. Repeat `STATUS` & `v0.2`/`v0.3` Interop** | Accept repeat `0x00 STATUS` messages and `payload.length >= 9` (default omitted `storageBucketsBitmask`, `codeBucketsBitmask`, `trieBucketsBitmask` to `0`). Only flip `trieBucketsBitmask` bit `b` after verifying `h_b === RootNode[b]`. |
| **4. Targeted `0x03` & `0x05 (kind = 1)` Lookup** | When no chunk starts at exact `targetOrOriginHash`, return the unique covering chunk (`origin <= target <= last`, or boundary chunk proving exclusion) so Open-World clients resolve cold misses in 1 RTT. |
| **5. Strict False-Zero Safety** | Never return `0x0` balance/nonce/slot on an unresolved Open-World miss. Only Closed-World buckets and verified Proven-Empty Tombstones may return empty/zero. |
| **6. Strictly Local BAL Write-Index** | Build write-index exclusively from locally verified `(headerRlp, balRlp)` bundles. Reject any witness/chunk with `anchorBlock < localBalEarliestBlock`. |
| **7. Zero `snap/1` Default & Custody Reciprocity** | Never mount `snap/1` on default clients (reserve for explicit `--snap-bootstrap` origin seeders). Enforce custody reciprocity: never advertise `bucketsBitmask === 0` peers in `ETHY_PEX`, throttle them on `0x0b`, and evict them first on full inbound slots. |
| **8. Byzantine `h_b` Quarantine** | Never trust `0x0e SUBTREE_ROOTS` until `keccak256(RLP([h_0..h_f, 0x])) === verifiedHeader(anchorBlock).stateRoot`. Immediately disconnect and quarantine any peer whose announced `h_b` disagrees with the verified `RootNode`. |

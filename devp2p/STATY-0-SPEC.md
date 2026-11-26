> **Superseded / Merged Notice (`2026-11-26`):** `STATY/0` has been **consolidated into Unified Custody [`ETHY/0` `v0.4`](./ETHY-0-SPEC.md)** so Ethy operates a single symmetric custody-sharded protocol (`--custody <1..16>`, default `2` = 1/8th, zero `snap/1` in default clients) instead of two competing protocols where cold misses remained tethered to `snap/1`. `STATY/0`'s multi-proof witness format (`0x0b`/`0x0c`), deduplicated `proofNodes[]`, strictly local BAL write-index, per-field BAL witness folding, Proven-Empty Tombstones, and shared `code/` store all live on directly inside [`ETHY-0-SPEC.md` (`v0.4`)](./ETHY-0-SPEC.md). This document is retained for historical reference.

# `STATY/0` — BAL-Extended Active State & Witness Swarm Protocol (`devp2p`)

| Field | Value |
| --- | --- |
| **Protocol Identifier** | `staty` |
| **Protocol Version** | `0` (`["staty", 0]`) |
| **Transport Layer** | Ethereum DevP2P (`RLPx` v5 with Snappy framing) |
| **Message Space** | `0x00` – `0x0a` (`length = 11`) |
| **Discovery ENR Key** | `"staty"` ([EIP-778](https://eips.ethereum.org/EIPS/eip-778) key `"staty"`, value `rlp(0)` = `0x80`) |
| **Companion Protocols** | `eth/71` ([EIP-7642](https://eips.ethereum.org/EIPS/eip-7642), [EIP-7928](https://eips.ethereum.org/EIPS/eip-7928)), [`ETHY/0`](./ETHY-0-SPEC.md) (optional full flat-state companion), `snap/1`, Consensus Light Client (`@ethereumjs/beacon-light`) |
| **Status** | **Superseded** — Merged into [`ETHY/0` `v0.4`](./ETHY-0-SPEC.md) (Historical Draft `v0.1`) |
| **Authors** | EthereumJS Team |

---

## Table of Contents

1. [Executive Summary & Motivation](#1-executive-summary--motivation)
2. [Protocol Suite Boundaries (`eth/71` · `STATY/0` · `ETHY/0`) & Shared Architecture](#2-protocol-suite-boundaries-eth71--staty0--ethy0--shared-architecture)
3. [Cryptographic Trust Model & BAL-Extended Witness Soundness](#3-cryptographic-trust-model--bal-extended-witness-soundness)
4. [Shared Block Calendar & Deterministic Hot Epoch Bundles (`E = 256`, `D = 4,096`)](#4-shared-block-calendar--deterministic-hot-epoch-bundles-e--256-d--4096)
5. [Capability Negotiation, ENR Discovery & `STATY_PEX`](#5-capability-negotiation-enr-discovery--staty_pex)
6. [Wire Encoding Conventions & Multi-Proof Packaging](#6-wire-encoding-conventions--multi-proof-packaging)
7. [Message Specification (`0x00` – `0x0a`)](#7-message-specification-0x00--0x0a)
8. [Client Sync, On-Demand Resolution & `O(1)` Epoch Minter Election](#8-client-sync-on-demand-resolution--o1-epoch-minter-election)
9. [Worked Byte-Level Walkthrough](#9-worked-byte-level-walkthrough)
10. [Integrator Guide (For Human Engineers & AI Coding Agents)](#10-integrator-guide-for-human-engineers--ai-coding-agents)

---

## 1. Executive Summary & Motivation

Post-Amsterdam Ethereum clients face a fundamental tension between **user onboarding speed** and **P2P network sustainability**:

1. **Full World State (`~160+ GB` on Mainnet)** — Syncing every account, every bytecode, and every historical storage slot across Ethereum takes hours and tens to hundreds of gigabytes of disk, even over an MPT-free flat-state swarm like [`ETHY/0`](./ETHY-0-SPEC.md). Syncing only accounts without code and storage leaves JSON-RPC (`eth_call`, ENS resolution, ERC-20 `balanceOf`, Uniswap quotes) unusable.
2. **Pure On-Demand `snap/1` Leeching Does Not Scale** — Clients that stay stateless and pull individual accounts/slots on demand from `snap/1` full nodes (`GetAccountRange` / `GetStorageRanges` / `GetTrieNodes`) onboard instantly, but create an asymmetric parasite problem:
   - In pre-Amsterdam Ethereum, an MPT proof for a hot contract (e.g., `USDC`, `WETH`, `ENS`, Uniswap pools) expires in **12 seconds** because `storageRoot` mutates every block, and `snap/1` servers prune historical state roots after **128 blocks (~25.6 minutes)**.
   - Every light client repeatedly hits full nodes for the same active state while contributing **nothing** back to the network.

### The Core Insight: EIP-7928 BALs Turn Ephemeral MPT Proofs into Reusable Multi-Day Witnesses

With **Block-Level Access Lists ([EIP-7928](https://eips.ethereum.org/EIPS/eip-7928) `BAL` on `eth/71`)**, every verified block publishes an exhaustive, header-committed (`header.blockAccessListHash`) record of:
- Every post-execution state mutation (`balanceChanges`, `nonceChanges`, `codeChanges`, `storageChanges`), AND
- Every storage slot read during block execution (`storageReads`).

This unlocks two properties that were impossible before Amsterdam:

1. **Multi-Day Witness Reuse Across Mutating `storageRoot`s:** Once a client verifies an MPT inclusion or exclusion proof for `(accountHash)` or `(accountHash, slotHash)` anchored at a past block `#B`, and holds the verified contiguous `(headerRlp, balRlp)` stream from `#B + 1 .. #Head`, **folding the BAL stream forward yields the exact consensus-proven value at `#Head`** — even if the contract's `storageRoot` mutated 10,000 times since `#B`.
2. **Deterministic Active-State Discovery Without Oracle Trust:** Because `BAL`s record every address and storage slot touched on-chain, the entire network independently agrees on the exact **Active Working Set** of every 256-block micro-epoch (~51 minutes) — typically `~0.1% – 1%` of world state, yet serving `> 95%` of real-world MetaMask, ENS, token, and DeFi RPC queries locally with zero network round-trips.

### What `STATY/0` Solves

**`STATY/0` (`["staty", 0]`)** is a standalone DevP2P subprotocol that turns lightweight BAL-native nodes into a **self-sustaining Active State & Witness Swarm**:

- **Live Verified JSON-RPC in 3–5 Minutes (`~3–10 GB` Disk):** A joining node downloads the recent `13.6–27.3 h` rolling BAL window over standard `eth/71`, downloads the corresponding **Hot Epoch Bundles** (`0x01`–`0x04`) and referenced **Bytecodes** (`0x07`–`0x08`) over `STATY/0`, verifies every proof locally against `@ethereumjs/beacon-light`, and immediately answers full verified `eth_call`, ENS, ERC-20, and DeFi queries locally.
- **Amortized `O(1)` Upstream `snap/1` Footprint:** Only `1–2` elected nodes per 256-block micro-epoch pull proofs for newly-read keys from `snap/1` and publish them into the `STATY/0` swarm. Cold misses are resolved first from `STATY/0` peers (`GET_WITNESSES`, `0x05`–`0x06`), falling back to `snap/1` only on a global swarm miss — after which the fetched witness is cached and immediately re-served to the swarm.
- **Every Node Gives Back:** Every `STATY/0` node serves:
  1. Verified **Hot Epoch Bundles** (`EPOCH_PART`),
  2. Its cached **Ad-Hoc Point Witnesses** (`WITNESSES`),
  3. Its immutable content-addressed **Contract Bytecodes** (`BYTECODES`), and
  4. Its verified rolling `8,192`-block `(header, BAL)` window over standard **`eth/71`** (`GetBlockHeaders` + `GetBlockAccessLists`).
- **Clean Co-Design with `ETHY/0`:** `STATY/0` stands completely on its own as the default fast-onboarding protocol, and composes seamlessly with [`ETHY/0`](./ETHY-0-SPEC.md) for `--full` nodes that want instant `STATY/0` usability while replicating 100% of world state (`~160 GB`) in the background for zero-leak read privacy and Tier-2 seeding.

---

## 2. Protocol Suite Boundaries (`eth/71` · `STATY/0` · `ETHY/0`) & Shared Architecture

`STATY/0` is intentionally scoped to avoid duplicating existing Ethereum wire protocols:

| Layer | Protocol | Capability | BAL Handling | State Scope | Typical Footprint |
| --- | --- | --- | --- | --- | --- |
| **Live Chain & BAL Stream** | **`eth/71`** | `["eth", 71]` | Canonical source & server for raw headers, bodies, receipts, and **rolling BAL windows** (`GetBlockAccessLists`) | Execution diffs only | `~0.4–0.8 GB` (`8,192` blk BAL window) |
| **Active State & Witness Swarm** | **`STATY/0`** | `["staty", 0]` | **Zero BAL messages on wire** — consumes verified `eth/71` BALs; maintains strictly local `write-index` | Deterministic **Hot Epoch Bundles** + P2P **Ad-Hoc Point Witnesses** + shared **Bytecodes** | `~3–10 GB`, live in `3–5 min` |
| **Full Flat-State Swarm** | **`ETHY/0`** | `["ethy", 0]` | Keeps `GET_BAL_RANGE` (`0x07`/`0x08`) solely to batch-bridge `#Pivot → #Head` without full block/receipt serving | 100% world state: 16 **Account Buckets** + 16 **Storage Buckets** + shared **Bytecodes** | `~160+ GB` Mainnet, zero-leak privacy |

```text
                       ┌───────────────────────────────────────────────┐
                       │    Consensus Light Client (@beacon-light)     │
                       │   BLS Sync Committee -> Verified Head Hash    │
                       └───────────────────────┬───────────────────────┘
                                               │ Anchor Truth
              ┌────────────────────────────────┼────────────────────────────────┐
              ▼                                ▼                                ▼
┌──────────────────────────┐     ┌──────────────────────────┐     ┌──────────────────────────┐
│          eth/71          │     │         STATY/0          │     │          ETHY/0          │
│ • Live headers & tx pool │     │ • Hot Epoch Bundles      │     │ • 16 Account Buckets     │
│ • Rolling 8,192-blk BALs │────►│ • P2P Point Witnesses    │     │ • 16 Storage Buckets     │
│ • Serves BALs back to    │     │ • Content-Addressed Code │     │ • Content-Addressed Code │
│   eth/71 peers           │     │ • Zero BAL wire overlap  │     │ • #Pivot->#Head BAL tail │
└─────────────┬────────────┘     └─────────────┬────────────┘     └─────────────┬────────────┘
              │                                │                                │
              └────────────────────────────────┼────────────────────────────────┘
                                               ▼
                  ┌─────────────────────────────────────────────────────────┐
                  │         Shared Storage & Unified FlatStateManager       │
                  │  code/ (immutable) · bals/ (verified) · write-index/    │
                  │  Per-Bucket Resolution:                                 │
                  │   • ETHY bucket complete  -> Closed-World (0 leaks)     │
                  │   • STATY-only bucket     -> Open-World (Flat ->        │
                  │     Tombstone -> STATY/0 GET_WITNESSES -> snap/1 cache) │
                  └─────────────────────────────────────────────────────────┘
```

### 2.1 Shared On-Disk Directory Layout (`STATY/0` + `ETHY/0`)

Implementations supporting either or both protocols SHOULD share the immutable bytecode store and verified rolling BAL archive:

```text
data/<chain>/
├── code/                         # SHARED: keccak256(code) -> raw bytecode (immutable, cross-pivot)
├── bals/                         # SHARED: verified contiguous (headerRlp, balRlp) rolling archive
├── staty/
│   ├── epochs/<epochNumber>/     # Verified Hot Epoch Bundle manifests + shard parts (0x0..0xf)
│   ├── witnesses/                # Cached verified ad-hoc point witnesses (keyed by hash + anchorBlock)
│   └── write-index/              # STRICTLY LOCAL index of last-modified block per account field / slot
└── ethy/                         # Present when ETHY/0 (--full) is enabled
    └── pivots/<stateRoot>/
        ├── accounts/<0..f>.bin + .proof.idx
        └── storage/<0..f>.bin  + .proof.idx
```

### 2.2 Unified `FlatStateManager` Semantics (Open-World vs. Closed-World)

A single `FlatStateManager` serves both `STATY/0`-only nodes and hybrid `STATY/0 + ETHY/0` (`--full`) nodes by tracking completeness per high nibble bucket `b = accountHash[0] >> 4` (`0..15`):

| Bucket `b` State | Local Key Status | EVM / RPC Action |
| --- | --- | --- |
| **Closed-World** (`ETHY/0` accounts & storage complete for bucket `b`) | Present in flat DB | Return local value immediately (`0` network queries, 100% read privacy) |
| **Closed-World** | Absent from flat DB | Return empty account / `0x00..00` immediately (provably absent) |
| **Open-World** (`STATY/0` active, `ETHY/0` bucket `b` not complete) | Present in flat DB (from BAL, Hot Epoch Bundle, or cached witness) | Return local value immediately (`0` network queries) |
| **Open-World** | **Proven-Empty Tombstone** (verified MPT exclusion proof or `storageRoot === KECCAK256_RLP`, untouched in BALs since `anchorBlock`) | Return empty account / `0x00..00` immediately (`0` network queries) |
| **Open-World** | **Unknown Miss** (neither value nor tombstone) | Query `STATY/0 GET_WITNESSES` → on global swarm miss, fetch single-key proof from `snap/1`, verify, fold BALs, commit to `FlatStateManager`, and cache in `staty/witnesses/` for P2P re-serving! |

In `--full` mode, as `ETHY/0` finishes each bucket `b` in the background, bucket `b` flips atomically from **Open-World** to **Closed-World** with zero downtime to live RPC or head following.

---

## 3. Cryptographic Trust Model & BAL-Extended Witness Soundness

A `STATY/0` client treats every `STATY/0` peer as an **untrusted data mule**. Every account, storage slot, non-existence tombstone, and bytecode received over `STATY/0` MUST pass cryptographic verification before committing to state.

### 3.1 Anchor Truth (`@ethereumjs/beacon-light` + Contiguous `eth/71` BAL Chain)

1. **Consensus Finality & Head:** The client obtains the validator-signed execution `blockHash` and `stateRoot` via `@ethereumjs/beacon-light` ($\ge 342/512$ Sync Committee BLS12-381 aggregate signature + SSZ Merkle branch).
2. **Contiguous Verified Header + BAL Window (`[localBalEarliestBlock .. #Head]`):**
   - Every header `#B` in `[localBalEarliestBlock .. #Head]` is linked backward via `keccak256(headerRlp) === childHeader.parentHash` from the CL-verified head.
   - Every BAL `balRlp` in `[localBalEarliestBlock + 1 .. #Head]` is verified against its execution header:
     ```text
     keccak256(balRlp) === header(#B).blockAccessListHash
     ```
   - Note: `localBalEarliestBlock` is the earliest block whose **stateRoot** is anchored in the verified header chain AND for which **every subsequent block** `localBalEarliestBlock + 1 .. #Head` has a verified `headerRlp` and `balRlp`.

### 3.2 Theorem: BAL-Extended Witness Soundness

Let $B$ be any block with `localBalEarliestBlock <= B <= #Head` and $R_B = \text{header}(B).\text{stateRoot}$.

1. **Account Soundness (Per-Field BAL Folding):**
   - An MPT proof against $R_B$ for `accountHash = keccak256(address)` cryptographically proves either:
     - **Inclusion:** the complete 4-tuple $(\text{nonce}_B, \text{balance}_B, \text{storageRoot}_B, \text{codeHash}_B)$ at block $B$, OR
     - **Exclusion:** the account did not exist at block $B$ (proven empty: $\text{nonce}_B = 0, \text{balance}_B = 0, \text{storageRoot}_B = \text{KECCAK256\_RLP}, \text{codeHash}_B = \text{KECCAK256\_NULL}$).
   - **Important EIP-7928 Property (Partial Account Updates):** An EIP-7928 BAL records per-field post-values (`balanceChanges`, `nonceChanges`, `codeChanges`). A block that only modifies an account's `balance` does **not** repeat its `nonce` or `codeHash`.
   - Therefore, forwarding the proven account from $B$ to `#Head` folds each field independently over the verified BAL stream `[B + 1 .. #Head]` (identical to `consumeBAL` restricted to `address`):
     - `balance_Head` = last `balanceChanges.postBalance` in `[B + 1 .. #Head]`, else `balance_B`
     - `nonce_Head`   = last `nonceChanges.postNonce` in `[B + 1 .. #Head]`, else `nonce_B`
     - `codeHash_Head` = `keccak256(last codeChanges.newCode)` in `[B + 1 .. #Head]`, else `codeHash_B`
     - If EIP-161 emptiness applies at the last mutation (e.g., same-transaction create + `SELFDESTRUCT`), the account is deleted (proven-empty tombstone).
   - The resulting `(nonce_Head, balance_Head, codeHash_Head)` is **100% consensus-equivalent** to reading the state trie at `#Head`.

2. **Storage Slot Soundness (Independent of Later `storageRoot` Mutations):**
   - Let `storageRoot_B` be the account's proven storage root at block $B$ (from step 1).
   - An MPT proof against `storageRoot_B` for `slotHash = keccak256(slot32)` proves either:
     - **Inclusion:** `slotVal_B` (non-zero `uint256`), OR
     - **Exclusion** (or trivial emptiness when `storageRoot_B === KECCAK256_RLP`): `slotVal_B = 0`.
   - Under post-Cancun **EIP-6780**, `SELFDESTRUCT` cannot destroy or wipe the storage of any contract created in an earlier block ($\le B$). Every modification to `(address, slot)` in blocks `[B + 1 .. #Head]` MUST appear explicitly as a `storageChanges` entry with its complete 32-byte `postValue` in that block's verified BAL.
   - Therefore:
     - If `(address, slot)` has any `storageChanges` entry in `[B + 1 .. #Head]`, the last `postValue` from the verified BALs is the exact value at `#Head` (the historical witness at $B$ MUST NOT overwrite it).
     - Otherwise (no `storageChanges` in `[B + 1 .. #Head]`), `slotVal_B` (including proven zero `0`) is **provably the exact value at `#Head`**, regardless of how many times other slots mutated `storageRoot` between $B$ and `#Head`!

### 3.3 Security Invariant: Write-Indices Are Strictly Local (Never Wire-Trusted)

> **CRITICAL RULE:** `STATY/0` NEVER accepts a peer-supplied "write-index" or "unchanged since block `#B`" claim over the wire.
>
> If a client trusted a remote write-index without verifying the underlying BALs, a malicious peer could serve a genuine 2-day-old MPT proof showing Alice had `1,000,000 USDC` and omit yesterday's transfer from the write-index.

Every `STATY/0` node builds its `staty/write-index/` **purely from locally verified `eth/71` BALs**, and enforces:

```text
ACCEPT witness at anchorBlock B  <=>  localBalEarliestBlock <= B <= bestVerifiedHead
                                      AND anchorStateRoot === verifiedHeader(B).stateRoot
                                      AND verifyMptProof(...) === true
```

- A freshly synced node starts with `localBalEarliestBlock ≈ #Head - 8,192` (~27.3 hours of witness horizon).
- A long-lived node that keeps verifying live `eth/71` BALs retains its local write-index history over days or weeks, allowing cached cold witnesses to remain cryptographically valid for its entire uptime at near-zero storage cost.

### 3.4 Exact Verification Predicates & Peer Reputation

Every data-carrying `STATY/0` response MUST satisfy its predicate before committing to state or cache:

| Message | Verification Predicate | On Failure |
| --- | --- | --- |
| `0x04 EPOCH_PART` | 1. `anchorBlock === epochNumber * 256` and `localBalEarliestBlock <= anchorBlock <= bestVerifiedHead`.<br>2. `anchorStateRoot === verifiedHeader(anchorBlock).stateRoot`.<br>3. `proofNodes.length <= MAX_PROOF_NODES_PER_MSG` (`4,096`) and total proof bytes `<= MAX_RESPONSE_BYTES` (`512 KB`).<br>4. Account entries sorted strictly ascending by `accountHash`; slot entries sorted strictly ascending by `(accountHash, slotHash)`.<br>5. Every `(accountHash, accountRlpOrEmpty)` verifies against `anchorStateRoot` using `proofNodes` (inclusion if non-empty, exclusion if `0x80`).<br>6. Every `(accountHash, slotHash, slotValOrEmpty)` has a verified `storageRoot` at the same `anchorBlock` (present in this message's `accounts[]` or already locally proven at `anchorBlock`), `slotValOrEmpty` is canonical minimal big-endian (`<= 32` bytes, no leading `0x00`), and verifies against `storageRoot` (if `0x80`: exclusion proof or `storageRoot === KECCAK256_RLP`; if non-empty: inclusion of `RLP.encode(slotValOrEmpty)`). | **Disconnect (`SUBPROTOCOL_ERROR`) + quarantine peer** (e.g., 24h ban). Discard entire message. |
| `0x06 WITNESSES` | For each group in `groups[]`: same checks (1)–(6) as `EPOCH_PART` (without requiring `anchorBlock % 256 === 0`, only `minAnchorBlock <= anchorBlock <= bestVerifiedHead` and `anchorBlock >= localBalEarliestBlock`). | **Disconnect (`SUBPROTOCOL_ERROR`) + quarantine peer**. Discard entire message. |
| `0x08 BYTECODES` | For every `code` in `codes[]`: `code.length <= MAX_CONTRACT_CODE_SIZE` (`24,576` bytes, or active EIP-7907 / fork cap) AND `keccak256(code)` matches a `codeHash` requested in the corresponding `GET_BYTECODES` message. | **Disconnect (`SUBPROTOCOL_ERROR`) + quarantine peer**. Discard offending blob(s). |
| Any miss / empty list | `EPOCH_MANIFEST.shards = []`, `EPOCH_PART.accounts = []`, `WITNESSES.groups = []`, `BYTECODES.codes = []` (or subset of requested keys). | **Valid miss (zero penalty).** Do not disconnect; retry remaining keys with another peer. |

---

## 4. Shared Block Calendar & Deterministic Hot Epoch Bundles (`E = 256`, `D = 4,096`)

`STATY/0` and `ETHY/0` align on one shared power-of-two block calendar so BAL retention, epoch boundaries, and snapshot refresh horizons never clash:

| Constant | Value (Blocks) | Mainnet Time | Protocol Role |
| --- | --- | --- | --- |
| `CANONICAL_PIVOT_GRID_BLOCKS` (`K`) | `32` | ~6.4 min | `ETHY/0` fine `SNAP/1` pivot grid (1 Beacon Epoch) |
| **`STATY_MICRO_EPOCH_BLOCKS` (`E`)** | **`256`** | **~51.2 min** | **`STATY/0` Hot Epoch Bundle cadence** (`epoch = blockNumber / 256`, `anchorBlock = epoch * 256`) |
| `MACRO_GENERATION_GRID_BLOCKS` (`G`) | `1,024` (`4E`) | ~3.4 h | `ETHY/0` macro-generation grid |
| **`STATY_MACRO_DAY_BLOCKS` (`D`)** | **`4,096`** (`16E`) | **~13.6 h** | **`STATY/0` deterministic dedup reset boundary** (`macroDay = epoch / 16`) & `ETHY/0` `HARD_PREFER_SNAP_AGE` |
| `DEFAULT_BAL_WINDOW_BLOCKS` (`2D`) | `8,192` (`32E`) | ~27.3 h | Default rolling verified BAL archive (`16..32` active micro-epochs = Macro-Day `d - 1` + current Macro-Day `d`) |

### 4.1 Deterministic Micro-Epoch Key-Set Rule

For micro-epoch `e` covering blocks `[(e - 1) * 256 + 1 .. e * 256]` with `anchorBlock = e * 256`:

1. **Finality Buffer:** Minting of epoch `e` begins only after `bestVerifiedHead >= anchorBlock + EPOCH_FINALITY_DELAY_BLOCKS` (default **`8` blocks** $\approx 96\text{s}$), protecting against shallow reorgs while leaving `~120` blocks of headroom inside `snap/1`'s 128-block state-root retention window.
2. **Macro-Day Dedup Base (`dedupBaseEpoch`):**
   - Normally `dedupBaseEpoch = Math.floor(e / 16) * 16` (start of Macro-Day `d`), with `dedupBaseBlock = (dedupBaseEpoch - 1) * 256 + 1`.
   - If the minter knows an earlier micro-epoch inside `[dedupBaseEpoch .. e - 1]` failed to mint across the swarm, it MAY set `dedupBaseEpoch = e` (self-contained fallback) and advertise that value in `EPOCH_MANIFEST`.
3. **Accounts Included in Epoch `e` (`accountSet(e)`):**
   - Collect every `address` appearing in any `BAL(b)` for `b ∈ [(e - 1) * 256 + 1 .. e * 256]` (whether read-only OR partially written — because EIP-7928 BALs record per-field deltas, proving the full 4-tuple `[nonce, balance, storageRoot, codeHash]` at `anchorBlock` guarantees all untouched fields are known!).
   - Exclude any `address` that already appeared in any `BAL(b')` for an earlier micro-epoch in `[dedupBaseEpoch .. e - 1]` (already proven in an earlier bundle of the same Macro-Day!).
   - For every remaining `address`, compute `accountHash = keccak256(address)` and include its MPT inclusion/exclusion proof against `header(anchorBlock).stateRoot`.
4. **Storage Slots Included in Epoch `e` (`slotSet(e)`):**
   - Collect every `(address, slot)` appearing in `BAL(b).storageReads` for `b ∈ [(e - 1) * 256 + 1 .. e * 256]`.
   - Exclude any `(address, slot)` that had a `storageChanges` write in any `BAL(b')` for `b' ∈ [dedupBaseBlock .. anchorBlock]` (its post-execution 32-byte value is already 100% known from the verified BALs alone!).
   - Exclude any `(address, slot)` already proven in an earlier micro-epoch `[dedupBaseEpoch .. e - 1]`.
   - For every remaining `(address, slot)`, include its MPT inclusion/exclusion proof against `account.storageRoot` at `anchorBlock`.
   - **Self-Contained Part Rule:** If an epoch part includes a storage slot for `accountHash`, that part MUST also include the `accountHash` account entry (and its account proof nodes) at `anchorBlock` even if `accountHash` was already seen in an earlier micro-epoch of the Macro-Day — guaranteeing every single `EPOCH_PART` is **100% independently verifiable** in parallel worker threads!
5. **Sharding & Chunking (`16` Nibble Shards × `<= 256 KB` Parts):**
   - Partition `accountSet(e)` and `slotSet(e)` into 16 shards by high nibble `shardIndex = accountHash[0] >> 4` (`0x0..0xf`) — matching `ETHY/0`'s 16 account buckets.
   - Within each shard, sort keys strictly ascending by `(accountHash, slotHash)` and slice into self-proving parts targeting `<= 256 KB` encoded size (`<= 512 KB` hard wire cap).
   - Deduplicate all MPT branch/extension/leaf nodes within each part into a single `proofNodes[]` dictionary (~40–60% bandwidth savings over naive per-key proof concatenation).
6. **Bytecodes:**
   - Bytecodes are **never duplicated inside `EPOCH_PART`**. Once a client verifies an account in `EPOCH_PART` whose `codeHash !== KECCAK256_NULL` is missing from shared `code/`, it fetches the code in batches via `GET_BYTECODES (0x07)` (or `ETHY/0` `kind = 0`).

### 4.2 Why Macro-Day Dedup Works So Well

- During micro-epoch `0` of Macro-Day `d`, hot constants (`USDC`, `USDT`, `WETH`, `ENS Registry`, Chainlink price feeds, Uniswap router/pool immutables) are minted once (~15–30 MB).
- During micro-epochs `1..15` of the same Macro-Day, only **newly touched** addresses and cold slots are minted (~2–8 MB per 51-minute epoch).
- At the next Macro-Day boundary (`epoch % 16 === 0`), dedup resets cleanly so joining nodes NEVER need more than `Macro-Day (d - 1)` + current `Macro-Day d` (`16..32` micro-epochs = `13.6..27.3 h`) to capture the entire recent active state!

---

## 5. Capability Negotiation, ENR Discovery & `STATY_PEX`

### 5.1 RLPx `HELLO` Capability Negotiation
- Subprotocol name: **`"staty"`** (lowercase ASCII, 5 bytes: `0x7374617479`).
- Subprotocol version: **`0`** (encoded as `intToBytes(0)` in `HELLO` capabilities).
- Subprotocol message-ID space length: **`11`** (`0x00` through `0x0a`).
- Nodes MAY negotiate `["eth", 71]`, `["staty", 0]`, `["ethy", 0]`, and `["snap", 1]` simultaneously on the same RLPx connection. Standard Ethereum clients that do not implement `staty/0` ignore `["staty", 0]` during `HELLO` capability intersection.

### 5.2 EIP-778 ENR Discovery Key (`"staty"`)
Nodes supporting `STATY/0` SHOULD include the key `"staty"` in their signed [EIP-778](https://eips.ethereum.org/EIPS/eip-778) Ethereum Node Record (ENR):
- **Key:** `"staty"` (ASCII `0x7374617479`)
- **Value:** RLP-encoded highest supported `staty` subprotocol version (`uint`). For `STATY/0`, `RLP.encode(0)` is `0x80` (empty byte string).
- Discovery v4/v5 and EIP-1459 DNS discovery prioritize peers advertising `"staty"` into reserved `STATY/0` peer slots (recommended `minStatyPeers = 6`).

### 5.3 Epidemic Peer Exchange (`STATY_PEX`) & NAT Bidirectional Serving
- Immediately after `STATY/0 STATUS (0x00)` handshake, peers exchange `GET_STATY_PEERS (0x09)` / `STATY_PEERS (0x0a)` (up to `32` active peers, ~75% freshest-epoch / highest-coverage peers and ~25% random active peers for graph expansion).
- Because DevP2P `RLPx` TCP streams are symmetric once opened, NATed residential laptops making outbound TCP connections serve `EPOCH_PART`, `WITNESSES`, `BYTECODES`, and `eth/71` BALs over those outbound sockets without requiring port forwarding.

---

## 6. Wire Encoding Conventions & Multi-Proof Packaging

### 6.1 Canonical RLP Scalar Rule
All `STATY/0` messages are RLP-encoded arrays compressed with Snappy (standard DevP2P base protocol `v5`).
Every unsigned integer scalar (`uint8`, `uint16`, `uint32`, `uint64`, `uint256`) in `STATY/0` RLP tuples MUST use **canonical minimal big-endian encoding**:
- Leading `0x00` bytes MUST be stripped.
- The integer `0` MUST be encoded as the empty byte array `new Uint8Array(0)` (RLP byte `0x80`), **never** `0x00` (`Uint8Array([0])`).

### 6.2 Canonical Key & Value Encoding Inside Proofs
- `accountHash`: fixed `32` bytes (`keccak256(address20)`).
- `slotHash`: fixed `32` bytes (`keccak256(slot32)`).
- `accountRlpOrEmpty`:
  - Empty bytes (`0x80`) = **proven non-existence** at `anchorStateRoot`.
  - Non-empty bytes = canonical Ethereum MPT account leaf RLP `[nonce, balance, storageRoot, codeHash]`.
- `slotValOrEmpty`:
  - Empty bytes (`0x80`) = **proven zero slot** (exclusion proof against `storageRoot`, or `storageRoot === KECCAK256_RLP`).
  - Non-empty bytes (`1..32` bytes, no leading `0x00`) = non-zero `uint256` slot value. In the Ethereum storage MPT, the leaf value verified by `verifyProof` is `RLP.encode(slotValOrEmpty)`.

### 6.3 Deduplicated Multi-Proof Dictionary (`proofNodes: [bytes, ...]`)
Within a single `EPOCH_PART (0x04)` or a single anchor group in `WITNESSES (0x06)`:
- `proofNodes` contains each unique RLP-encoded MPT node (`branch`, `extension`, or `leaf`) **at most once**, covering both the state trie (`anchorStateRoot`) and any referenced storage tries (`storageRoot`).
- The receiver builds a temporary read-only `Map<bytes32, bytes>` mapping `keccak256(nodeRlp) -> nodeRlp` (for nodes `>= 32` bytes; inline `< 32`-byte RLP children are already embedded inside their parent node), and runs standard MPT proof lookup from `anchorStateRoot` / `storageRoot` against that map.
- **DoS Bounds:** Receivers MUST reject any message where `proofNodes.length > 4,096` or total decoded payload exceeds `MAX_STATY_MESSAGE_BYTES = 2 * 1024 * 1024` (2 MiB).

---

## 7. Message Specification (`0x00` – `0x0a`)

| Code | Name | Direction | Description |
| --- | --- | --- | --- |
| `0x00` | `STATUS` | Peer ↔ Peer | Initial handshake + **repeatable live updates** advertising servable micro-epochs, active minter status, witness horizon, and bytecode inventory hint. |
| `0x01` | `GET_EPOCH_MANIFEST` | Client → Server | Request the 16-shard manifest for micro-epoch `epochNumber`. |
| `0x02` | `EPOCH_MANIFEST` | Server → Client | Returns anchor block/root, `dedupBaseEpoch`, and per-shard part counts. |
| `0x03` | `GET_EPOCH_PART` | Client → Server | Request one self-proving part `(epochNumber, shardIndex, partIndex)`. |
| `0x04` | `EPOCH_PART` | Server → Client | Returns sorted proven accounts, proven storage slots, and deduplicated `proofNodes[]`. |
| `0x05` | `GET_WITNESSES` | Client → Server | Request cached ad-hoc MPT witnesses for specific `(accountHash, slotHashes[])` pairs with `anchorBlock >= minAnchorBlock`. |
| `0x06` | `WITNESSES` | Server → Client | Returns cached witnesses grouped by `(anchorBlock, anchorStateRoot)` with deduplicated `proofNodes[]`. |
| `0x07` | `GET_BYTECODES` | Client → Server | Request a batch of content-addressed contract bytecodes (`1..64` `codeHashes`). |
| `0x08` | `BYTECODES` | Server → Client | Returns matching raw contract bytecodes (misses omitted). |
| `0x09` | `GET_STATY_PEERS` | Peer → Peer | Request up to `maxPeers` active `STATY/0` peers (`STATY_PEX`). |
| `0x0a` | `STATY_PEERS` | Peer → Peer | Returns active `STATY/0` peer descriptors with epoch coverage hints. |

---

### `0x00` `STATUS`

Sent immediately upon mounting `STATY/0`, and **re-sent whenever `latestEpoch`, `epochMask`, `mintingEpoch`, or `witnessMinAnchorBlock` changes** (push-based swarm state updates without polling).

```text
[
  protocolVersion: uint,          // 0 (encoded as 0x80)
  chainId: uint,                  // e.g. 1 (Mainnet), 11155111 (Sepolia)
  genesisHash: bytes32,           // 32-byte execution genesis block hash
  earliestEpoch: uint64,          // Oldest micro-epoch e servable by this peer (0 if none)
  latestEpoch: uint64,            // Newest micro-epoch e completely servable by this peer (0 if none)
  epochMask: uint64,              // Bit i (0..63) is 1 iff micro-epoch (latestEpoch - i) is 100% servable
  mintingEpoch: uint64,           // Micro-epoch currently being minted via SNAP/1 (0 if idle; O(1) suppression)
  witnessMinAnchorBlock: uint64,  // Lowest anchorBlock among cached ad-hoc witnesses (0 if none)
  witnessCountHint: uint32,       // Approximate count of cached ad-hoc witnesses servable via GET_WITNESSES
  codeCountHint: uint32           // Approximate count of contract bytecodes servable via GET_BYTECODES
]
```

**Validation Rules:**
- If `protocolVersion !== 0`, `chainId` mismatches local chain ID, or `genesisHash` mismatches local genesis hash, the receiver MUST disconnect with `DISCONNECT_REASON.SUBPROTOCOL_ERROR`.
- `STATUS` MAY be sent multiple times per session. Peers MUST accept repeat `STATUS` frames as live updates (and ignore any trailing future extension fields beyond index `9`).
- When `latestEpoch > 0`, bit `0` of `epochMask` MUST be `1` (representing `latestEpoch` itself); bit `i` (`0 <= i <= 63`) covers up to `64` micro-epochs (`16,384` blocks $\approx 54.6\text{h}$).

---

### `0x01` `GET_EPOCH_MANIFEST` & `0x02` `EPOCH_MANIFEST`

Queries a peer for the shard layout and metadata of Hot Epoch Bundle `epochNumber`.

**Request (`0x01 GET_EPOCH_MANIFEST`):**
```text
[
  requestId: uint64,
  epochNumber: uint64             // Micro-epoch index (anchorBlock = epochNumber * 256)
]
```

**Response (`0x02 EPOCH_MANIFEST`):**
```text
[
  requestId: uint64,
  epochNumber: uint64,
  anchorBlock: uint64,            // MUST equal epochNumber * 256 (0 if epoch unavailable)
  anchorStateRoot: bytes32,       // 32-byte stateRoot at anchorBlock (32 zero bytes if unavailable)
  dedupBaseEpoch: uint64,         // First micro-epoch of the dedup window (typically floor(e/16)*16)
  shards: [
    [
      shardIndex: uint8,          // 0..15 (high nibble of accountHash)
      partCount: uint16,          // Number of self-proving parts in this shard (0 if shard empty)
      accountCount: uint32,       // Total account proofs in this shard
      slotCount: uint32,          // Total storage slot proofs in this shard
      totalBytes: uint32          // Total approximate wire bytes across all parts of this shard
    ],
    ...
  ]
]
```
- If the peer does not have `epochNumber`, it MUST reply with `anchorBlock = 0`, `anchorStateRoot = ZERO32`, `dedupBaseEpoch = 0`, `shards = []` (never hang or disconnect).
- **Receiver Check:** Verify `anchorBlock === epochNumber * 256` and `anchorStateRoot === verifiedHeader(anchorBlock).stateRoot` before requesting parts.

---

### `0x03` `GET_EPOCH_PART` & `0x04` `EPOCH_PART`

Requests and transfers one independently verifiable shard part of a Hot Epoch Bundle.

**Request (`0x03 GET_EPOCH_PART`):**
```text
[
  requestId: uint64,
  epochNumber: uint64,
  shardIndex: uint8,              // 0..15
  partIndex: uint16               // 0 .. partCount - 1
]
```

**Response (`0x04 EPOCH_PART`):**
```text
[
  requestId: uint64,
  epochNumber: uint64,
  shardIndex: uint8,
  partIndex: uint16,
  anchorBlock: uint64,            // epochNumber * 256
  anchorStateRoot: bytes32,       // Verified stateRoot at anchorBlock
  accounts: [
    [
      accountHash: bytes32,       // Strictly ascending; high nibble MUST equal shardIndex
      accountRlpOrEmpty: bytes    // Canonical 4-elem RLP account, or 0x80 (proven non-existence)
    ],
    ...
  ],
  slots: [
    [
      accountHash: bytes32,       // Sorted ascending by (accountHash, slotHash); MUST appear in accounts[] above
      slotHash: bytes32,
      slotValOrEmpty: bytes       // Minimal BE uint256 (1..32B, no leading 0x00), or 0x80 (proven zero)
    ],
    ...
  ],
  proofNodes: [bytes, ...]        // Deduplicated RLP-encoded MPT nodes covering accounts[] and slots[]
]
```
- **Miss Response:** If the part is unavailable, reply with `accounts = []`, `slots = []`, `proofNodes = []`.
- **Verification:** Run § 3.4 `EPOCH_PART` predicate, then fold verified BALs `(anchorBlock + 1 .. #Head)` per § 3.2 and persist the verified part under `staty/epochs/<epochNumber>/` so this node can immediately re-serve it!

---

### `0x05` `GET_WITNESSES` & `0x06` `WITNESSES`

Requests cached point witnesses (from either `staty/witnesses/` or `staty/epochs/`) for arbitrary cold accounts and storage slots. Servers MUST answer purely from local cache/disk — servers MUST NEVER block on upstream `snap/1` queries to answer `GET_WITNESSES`.

**Request (`0x05 GET_WITNESSES`):**
```text
[
  requestId: uint64,
  minAnchorBlock: uint64,         // Client's localBalEarliestBlock; server MUST NOT return anchorBlock < minAnchorBlock
  maxResponseBytes: uint32,       // Soft response budget in bytes (clamped to <= 524,288)
  queries: [                      // 1..64 accounts; total slotHashes across all queries <= 256
    [
      accountHash: bytes32,
      slotHashes: [bytes32, ...]  // Empty [] = account witness only; non-empty = account + listed slots
    ],
    ...
  ]
]
```

**Response (`0x06 WITNESSES`):**
```text
[
  requestId: uint64,
  groups: [                       // Grouped by anchorBlock so proofNodes deduplicate within each anchor
    [
      anchorBlock: uint64,        // MUST satisfy minAnchorBlock <= anchorBlock
      anchorStateRoot: bytes32,
      accounts: [
        [accountHash: bytes32, accountRlpOrEmpty: bytes],
        ...
      ],
      slots: [
        [accountHash: bytes32, slotHash: bytes32, slotValOrEmpty: bytes],
        ...
      ],
      proofNodes: [bytes, ...]
    ],
    ...
  ]
]
```
- **Partial / Miss Semantics:** Any queried `accountHash` or `slotHash` not found locally with `anchorBlock >= minAnchorBlock` is simply omitted. If nothing matches, reply with `groups = []`.
- **Self-Contained Anchor Groups:** If a group includes a slot for `accountHash`, that group MUST include the `accountHash` entry in `accounts[]` at the same `anchorBlock` so the receiver can verify `storageRoot` in one pass.
- **Note for `ETHY/0` Hybrid Servers:** A node running both `ETHY/0` and `STATY/0` may answer `GET_WITNESSES` from its `staty/epochs/` and `staty/witnesses/` caches (it still does NOT need an 80 GB MPT database!).

---

### `0x07` `GET_BYTECODES` & `0x08` `BYTECODES`

Content-addressed contract bytecode transfer, backed by the shared immutable `code/` directory.

**Request (`0x07 GET_BYTECODES`):**
```text
[
  requestId: uint64,
  codeHashes: [bytes32, ...],     // 1..64 distinct 32-byte keccak256 codeHashes
  maxResponseBytes: uint32        // Soft budget (clamped to <= 524,288)
]
```

**Response (`0x08 BYTECODES`):**
```text
[
  requestId: uint64,
  codes: [bytes, ...]             // Raw contract bytecodes matching a subset of requested codeHashes (misses omitted)
]
```
- **Verification:** For every `code` in `codes[]`, compute `h = keccak256(code)`. If `h` was not in the request's `codeHashes` set, disconnect (`SUBPROTOCOL_ERROR`) and quarantine peer. Otherwise write `code` to shared `code/` indexed by `h`.

---

### `0x09` `GET_STATY_PEERS` & `0x0a` `STATY_PEERS` (`STATY_PEX`)

Exchanges recently active `STATY/0` peer endpoints for fast swarm mesh discovery.

**Request (`0x09 GET_STATY_PEERS`):**
```text
[
  requestId: uint64,
  maxPeers: uint8                 // 1..32 (recommended default: 16)
]
```

**Response (`0x0a STATY_PEERS`):**
```text
[
  requestId: uint64,
  peers: [
    [
      peerId: bytes,              // 64-byte uncompressed secp256k1 public key (no 0x04 prefix)
      address: bytes,             // UTF-8 IPv4/IPv6 string
      tcpPort: uint16,
      udpPort: uint16,
      latestEpoch: uint64,
      epochMask: uint64,
      witnessCountHint: uint32,
      lastSeenAgeSec: uint32
    ],
    ...
  ]
]
```

---

## 8. Client Sync, On-Demand Resolution & `O(1)` Epoch Minter Election

### 8.1 Fresh Onboarding Flow (`3–5 min` to Live Verified RPC)

```text
1. Consensus Anchor  ──► @ethereumjs/beacon-light verifies finalized & head blockHash (#Head)
2. eth/71 BAL Tail   ──► Download & verify contiguous (headerRlp, balRlp) for:
                         startBlock = (currentMacroDay - 1) * 4,096  ..  #Head
                         (16..32 micro-epochs = 4,096..8,192 blocks; ~350–600 MB)
                         -> Populate local staty/write-index/ + apply BAL writes to FlatStateManager
                         -> Immediately serve this rolling BAL window back to eth/71 peers!
3. STATY/0 Hot Sync  ──► For each micro-epoch e in [(d-1)*16 .. latestClosedEpoch] (rarest-first):
                         • GET_EPOCH_MANIFEST(e) -> verify anchorStateRoot against local header
                         • Stripe GET_EPOCH_PART(e, shard 0..15, part) across STATY/0 peers
                         • Verify MPT multiproofs, fold local BALs (anchorBlock+1..#Head),
                           commit to FlatStateManager, save to staty/epochs/<e>/
                         • Flip bit in outbound STATY/0 STATUS.epochMask immediately!
4. Bytecode Batch    ──► Collect missing codeHashes from verified accounts -> GET_BYTECODES (0x07)
                         -> Verify keccak256(code) === codeHash -> persist in shared code/
5. ★ Live & Seeding  ──► Full verified JSON-RPC (eth_call, ENS, DeFi) + continuous live eth/71 BALs
                         + optional background ETHY/0 sync (--full mode)
```

### 8.2 Cold-Miss Resolution Pipeline (`eth_call` / `eth_getStorageAt` / `eth_getBalance`)

When a local EVM execution or JSON-RPC query hits an **Unknown Miss** in an Open-World bucket:

1. **Coalesce:** Batch concurrent misses (e.g., an `eth_call` touching 12 slots of a cold contract) into a single query set.
2. **Step 1 — `STATY/0` Swarm Fan-Out (`GET_WITNESSES`, `0x05`):**
   - Fan out to `2–3` connected `STATY/0` peers with highest `witnessCountHint` / `epochMask` coverage, setting `minAnchorBlock = localBalEarliestBlock`.
   - Verify any returned groups (§ 3.4), fold local BALs (`anchorBlock + 1 .. #Head`), commit to `FlatStateManager`, and cache in `staty/witnesses/`.
3. **Step 2 — `snap/1` Single-Key Fallback (Global Swarm Miss Only):**
   - Only for keys still unresolved after Step 1, issue targeted `snap/1` `GetAccountRange(limit = accountHash)` / `GetStorageRanges(origin = slotHash, limit = slotHash)` / `GetByteCodes` anchored at a recent verified block `B` (`#Head - B <= 64`).
   - Verify the returned MPT proof against `header(B).stateRoot`, fold BALs `(B + 1 .. #Head)`, commit to `FlatStateManager`, AND **persist the verified witness in `staty/witnesses/`** (and bump `witnessCountHint` in `STATY/0 STATUS`) so every subsequent node querying that cold contract is served directly by `STATY/0`!

### 8.3 Decentralized `O(1)` Micro-Epoch Minter Election

Every `256` blocks (~51.2 minutes), when `bestVerifiedHead >= e * 256 + 8` and no connected peer advertises `latestEpoch >= e` or `mintingEpoch === e`:

1. **Hash-Lottery Order-Statistic Delay** (identical mathematical mechanism to [`ETHY/0` § 8.3](./ETHY-0-SPEC.md#83-two-tier-freshness-horizons--amortized-o1-decentralized-minter-election)):
   - Each eligible `STATY/0` node (synced, connected to `>= 1` `snap/1` peer, and holding verified BALs for `[(dedupBaseEpoch - 1) * 256 + 1 .. e * 256]`) computes uniform $u \in (0, 1)$ from `keccak256(localNodeId || bigIntToBytes(e))` and local swarm size estimate $\hat{N} = \max(1, \text{knownActiveStatyPeers} + 1)$:
     $$\Delta t = \text{clamp}\!\left(W_{\min} + \left(-\ln(1 - u)\right) \cdot \frac{\hat{N} \cdot T_{\text{target}}}{\lambda_{\text{target}}} \cdot w,\; W_{\min},\; W_{\max}\right)$$
   - Recommended parameters: $W_{\min} = 10\text{s}$, $T_{\text{target}} = 45\text{s}$, $\lambda_{\text{target}} = 1.5$, $W_{\max} = 300\text{s}$, uptime/reachability weight $w \in [0.25, 4.0]$.
   - Regardless of whether the swarm has `10` or `50,000` nodes, only **`1–2` nodes** win the lottery and query `snap/1` for epoch `e`!
2. **Suppression & Anti-Fake-Minter Liveness Guard:**
   - The winning node broadcasts `STATY/0 STATUS` with `mintingEpoch = e` while pulling proofs for `accountSet(e)` and `slotSet(e)` from `snap/1` at `anchorBlock = e * 256`.
   - Other nodes suppress their local timer **only if** `e * 256 <= bestVerifiedHead` AND the announcing peer finishes and publishes `latestEpoch >= e` (verified via `GET_EPOCH_MANIFEST(e)`) within `STATY_MINT_LIVENESS_TIMEOUT_MS = 180,000ms` (3 minutes).
   - If the winner stalls or disconnects, suppression expires and the next-ranked volunteer takes over well before `snap/1`'s 128-block (`~25.6 min`) state-root cliff.
3. **Cache Retention & Pruning:**
   - By default, nodes retain `staty/epochs/` for the active `2D` (`32` micro-epochs) rolling window and prune older epoch directories automatically.
   - `staty/witnesses/` uses an LRU disk budget (recommended default: `512 MB` – `2 GB`) and automatically evicts any witness with `anchorBlock < localBalEarliestBlock`.
   - Shared `code/` is immutable and never pruned while referenced by active state (entire Ethereum mainnet distinct contract bytecode is `< 15 GB`, and active working-set bytecode is `< 1.5 GB`).

---

## 9. Worked Byte-Level Walkthrough

Suppose **Node A** has verified micro-epoch `e = 47,120` (`anchorBlock = 47,120 * 256 = 12,062,720`, `dedupBaseEpoch = 47,104`), and **Node B** joins at `#Head = 12,062,740` after downloading `eth/71` BALs for `[12,058,624 .. 12,062,740]`:

1. **Node A sends `0x00 STATUS`:**
   ```ts
   [
     new Uint8Array(0),                      // protocolVersion = 0 (canonical 0x80)
     bigIntToUnpaddedBytes(11155111n),       // chainId = Sepolia (0xaa36a7)
     genesisHash32,                          // 32 bytes
     bigIntToUnpaddedBytes(47088n),          // earliestEpoch = 47,088
     bigIntToUnpaddedBytes(47120n),          // latestEpoch = 47,120
     bigIntToUnpaddedBytes(0x1ffffffffn),    // epochMask (33 contiguous micro-epochs ready)
     new Uint8Array(0),                      // mintingEpoch = 0 (idle)
     bigIntToUnpaddedBytes(12058624n),       // witnessMinAnchorBlock
     intToUnpaddedBytes(4200),               // witnessCountHint
     intToUnpaddedBytes(18500),              // codeCountHint
   ]
   ```

2. **Node B queries `0x01 GET_EPOCH_MANIFEST` for `e = 47,120`:**
   ```ts
   [
     bigIntToUnpaddedBytes(1n),              // requestId = 1
     bigIntToUnpaddedBytes(47120n),          // epochNumber = 47,120
   ]
   ```

3. **Node A replies `0x02 EPOCH_MANIFEST`**, Node B confirms `anchorBlock === 12_062_720n` and `equalsBytes(anchorStateRoot, verifiedHeader(12_062_720n).stateRoot)`, then requests `0x03 GET_EPOCH_PART(reqId=2, epoch=47120, shardIndex=0x0, partIndex=0)`.

4. **Node B receives `0x04 EPOCH_PART`** containing an account `accountHash` (whose `codeHash` is `USDC`) and a read-only storage slot `(accountHash, slotHash, slotValOrEmpty = 0x06)` (`decimals() = 6`):
   - Builds temporary proof map from `proofNodes[]`.
   - Verifies `accountRlp` against `anchorStateRoot` at `accountHash` → extracts `storageRoot`.
   - Verifies `RLP.encode(hexToBytes('0x06'))` against `storageRoot` at `slotHash`.
   - Checks local `staty/write-index/`: `(accountHash, slotHash)` had **no** `storageChanges` in verified BALs `[12,062,721 .. 12,062,740]`.
   - Commits `decimals = 6` to `FlatStateManager` as provably valid at `#12,062,740`, requests missing `codeHash` via `0x07 GET_BYTECODES`, and persists the verified part in `staty/epochs/47120/` to begin seeding to peers!

---

## 10. Integrator Guide (For Human Engineers & AI Coding Agents)

| Invariant | Requirement |
| --- | --- |
| **1. Canonical RLP Zero (`0x80`)** | Never encode integer `0` (`protocolVersion = 0`, `shardIndex = 0`, `partIndex = 0`, `mintingEpoch = 0`) as `Uint8Array([0])` (`0x00`). Always use minimal unpadded big-endian bytes (`new Uint8Array(0)` → `0x80`). |
| **2. Zero BAL Transport in `STATY/0`** | Do **not** add BAL transfer messages to `STATY/0`. Pull and serve rolling `(headerRlp, balRlp)` windows over standard `eth/71` (`GetBlockHeaders`, `GetBlockAccessLists`). |
| **3. Local Write-Index Only** | Never trust peer-supplied write-indices. Reject any `EPOCH_PART` or `WITNESSES` group with `anchorBlock < localBalEarliestBlock` or `anchorStateRoot !== verifiedHeader(anchorBlock).stateRoot`. |
| **4. Per-Field Account Folding** | Because EIP-7928 BALs emit per-field deltas (`balanceChanges`, `nonceChanges`, `codeChanges`), include every touched account in `accountSet(e)` (not only read-only accounts) and fold each field independently across `[anchorBlock + 1 .. #Head]`. |
| **5. Proven-Empty Tombstones** | Persist verified MPT exclusion proofs (`accountRlpOrEmpty = 0x80` or `slotValOrEmpty = 0x80`) as **proven-empty tombstones** in `FlatStateManager` so repeated EVM reads of zero slots never re-trigger network queries. |
| **6. Non-Blocking `GET_WITNESSES` Server** | Answer `GET_WITNESSES (0x05)` strictly from local `staty/witnesses/` and `staty/epochs/` caches. Never synchronously proxy a `STATY/0` request to `snap/1` (prevents amplification attacks). |
| **7. Repeat `STATUS` Allowed** | Do **not** disconnect peers sending repeat `STATY/0 STATUS (0x00)` messages; they advertise newly completed epochs, active `mintingEpoch` suppression, and growing witness/code caches. |
| **8. Shared `code/` Store** | Store verified bytecodes in a single content-addressed `code/` directory shared between `STATY/0` and `ETHY/0` so upgrading to `--full` never re-downloads bytecode. |

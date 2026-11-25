# `ETHY/0` — Self-Proving Flat State & BAL Swarm Protocol (`devp2p`)

| Field | Value |
| --- | --- |
| **Protocol Identifier** | `ethy` |
| **Protocol Version** | `0` (`["ethy", 0]`) |
| **Transport Layer** | Ethereum DevP2P (`RLPx` v5 with Snappy framing) |
| **Message Space** | `0x00` – `0x0a` (`length = 11`) |
| **Discovery ENR Key** | `"ethy"` ([EIP-778](https://eips.ethereum.org/EIPS/eip-778) key `"ethy"`, value `rlp(0)` = `0x80`) |
| **Companion Protocols** | `eth/69–71` ([EIP-7642](https://eips.ethereum.org/EIPS/eip-7642), [EIP-7928](https://eips.ethereum.org/EIPS/eip-7928)), [`STATY/0`](./STATY-0-SPEC.md) (Active State & Witness Swarm), `snap/1`, Consensus Light Client (`@ethereumjs/beacon-light`) |
| **Status** | Draft Specification (`v0.3` — Amsterdam Era) |
| **Authors** | EthereumJS Team |

---

## Table of Contents

1. [Executive Summary & Motivation](#1-executive-summary--motivation)
2. [Architectural Overview: How `ETHY/0` Eliminates the 80 GB MPT Server Requirement](#2-architectural-overview-how-ethy0-eliminates-the-80-gb-mpt-server-requirement)
3. [Cryptographic Trust Model & End-to-End Verification Chain](#3-cryptographic-trust-model--end-to-end-verification-chain)
4. [Capability Negotiation, ENR Discovery & `ETHY_PEX`](#4-capability-negotiation-enr-discovery--ethy_pex)
5. [Wire Encoding Conventions & Binary Data Layouts](#5-wire-encoding-conventions--binary-data-layouts)
6. [Message Specification (`0x00` – `0x0a`)](#6-message-specification-0x00--0x0a)
7. [Sync Source Selection & Fallback Algorithm (`ETHY/0` vs `SNAP/1`)](#7-sync-source-selection--fallback-algorithm-ethy0-vs-snap1)
8. [Swarm Convergence, Amortized `O(1)` Snapshot Refresh & Long-Lived Seeding](#8-swarm-convergence-amortized-o1-snapshot-refresh--long-lived-seeding)
9. [Co-Design & Interop with `STATY/0` (Two-Protocol Suite)](#9-co-design--interop-with-staty0-two-protocol-suite)
10. [Worked Byte-Level Walkthrough](#10-worked-byte-level-walkthrough)
11. [Integrator Guide (For Human Engineers & AI Coding Agents)](#11-integrator-guide-for-human-engineers--ai-coding-agents)

---

## 1. Executive Summary & Motivation

With the introduction of **Block-Level Access Lists ([EIP-7928](https://eips.ethereum.org/EIPS/eip-7928) `BAL` on `eth/71`)** in the Amsterdam hardfork and **Consensus Light Clients (`Altair` / `Electra` / `Amsterdam` Sync Committee BLS verification)**, a new class of lightweight, self-sovereign Ethereum node has become possible:

- Instead of maintaining an **80+ GB database of interior Merkle Patricia Trie (`MPT`) branch nodes** or burning CPU re-executing every transaction of every block, a **BAL-Native Flat-State Node** (such as [`@ethereumjs/ethy`](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/ethy)) syncs the flat account/storage leaves once, verifies consensus finality via the 512-validator Sync Committee (`@ethereumjs/beacon-light`), and advances its flat state block-by-block in `< 5ms` by verifying `header.blockAccessListHash` and applying the block's post-execution `BAL` (`consumeBAL`).
- Because it holds 100% of the flat Ethereum world state (`Accounts`, `Bytecode`, and `Storage`) locally on disk alongside an on-demand EVM (`@ethereumjs/evm`), it achieves **100% read privacy** (`eth_getBalance`, `eth_getStorageAt`, `eth_call` never leave the user's machine) and **censorship-resistant P2P transaction broadcasting** at a fraction of the hardware footprint of a traditional full node.

### The Asymmetric `snap/1` Bottleneck

However, relying solely on legacy **`snap/1`** for initial state onboarding creates two structural problems:

1. **Flat-State Nodes Cannot Serve `snap/1` (`GetAccountRange` / `GetStorageRanges` / `GetTrieNodes`):**
   - In `snap/1`, a server must dynamically generate left and right **Merkle boundary proofs** (`proof: bytes[]`) for whatever arbitrary `rootHash` and `[originHash .. limitHash]` slice a client requests, and must answer random-access `GetTrieNodes` healing queries.
   - Doing so requires storing 128 historical Merkle Patricia Trie roots and all interior branch nodes (`~80 GB`). A flat-state node that discards interior trie nodes after verifying incoming range proofs cannot serve `snap/1` range requests.
2. **`snap/1` Has No Partial-Capability Advertisement (`STATUS`):**
   - `snap/1` has no `STATUS` handshake message. Every node that mounts `snap/1` to *request* ranges appears identical to a full `snap/1` server.
3. **The 128-Block (~25 Minute) State-Root Cliff:**
   - Standard `snap/1` peers prune historical state roots after 128 blocks (~25.6 minutes). A snapshot from 3 hours ago cannot be queried over `snap/1`.

### What `ETHY/0` Solves

**`ETHY/0` (`["ethy", 0]`)** is a purpose-built DevP2P subprotocol that turns flat-state nodes into a **self-sustaining, BitTorrent-style peer-to-peer state and BAL swarm**:

- **Zero MPT Database Needed to Seed State:** During its own initial sync, an Ethy node saves the small (~3–5 KB) left/right Merkle boundary proof (`proofNodesRlp`) that accompanied each verified `128 KB` flat account/storage slab (`< 3%` disk overhead). It can then re-serve that exact **Self-Proving Chunk** (`binaryAccountsSlab + proofNodesRlp + chunkStateRoot`) directly from flat files with `O(1)` disk seeks!
- **100% Trustless Verification:** A joining node does **not** trust the serving `ETHY/0` peer. It verifies every received chunk using standard `verifyMerkleRangeProof` against `chunkStateRoot`, and verifies `chunkStateRoot` against the block header / BAL chain anchored by `@ethereumjs/beacon-light`.
- **Multi-Day Snapshot Longevity via `BAL_RANGE`:** Because `eth/71` BALs deterministically advance flat state forward, an `ETHY/0` seeder can serve a verified base snapshot `#C` from hours or days ago together with a batch-streamed **`BAL_RANGE` (`#C + 1 .. #Head`)**, offloading `snap/1` peers by **1:100 or more**.
- **Immediate Partial Seeding (`bucketsBitmask`):** Via `ETHY/0 STATUS`, a node advertises a 16-bit `bucketsBitmask` (`0x0000..0xffff`) for nibbles `0x0..0xf`. As soon as a syncing node finishes bucket `0x0` (6.25% of sync), it can immediately seed bucket `0x0` to other joining peers!

---

## 2. Architectural Overview: How `ETHY/0` Eliminates the 80 GB MPT Server Requirement

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│ 1. INITIAL SEEDING (1 in N Nodes Syncs via SNAP/1 + ETH/71 BALs)                  │
│                                                                                   │
│   Geth / Reth (snap/1) ──AccountRange(R_k, [origin..last], leaves, proofNodes)──► │
│                                                                                   │
│   Ethy Seeder (Node A):                                                           │
│     • Verifies verifyMerkleRangeProof(R_k, origin, last, leaves, proofNodes)      │
│     • Appends 136B BinaryAccount rows to  accounts/<b_hex>.bin                    │
│     • Appends ~4 KB proof sidecar entry to accounts/<b_hex>.proof.idx             │
│     • Persists verified (headerRlp, balRlp) in heal/<blockNumber>.bal             │
│     • Sets bit (1 << b) in ETHY/0 STATUS bucketsBitmask as each bucket finishes!  │
└─────────────────────────────────────────┬─────────────────────────────────────────┘
                                          │
                                          ▼  ETHY/0 Swarm (100x Offload, Zero MPT DB)
┌───────────────────────────────────────────────────────────────────────────────────┐
│ 2. SWARM ONBOARDING (Joining Ethy Node B Syncs from Ethy Node A / Swarm)          │
│                                                                                   │
│   Node A ──0x00 STATUS(basePivot=#C, bucketsBitmask=0xffff, balRange=#C..#H)────► │
│   Node B ──0x01 GET_BUCKET_MANIFEST(bucket=0x0)─────────────────────────────────► │
│   Node A ──0x02 BUCKET_MANIFEST(chunks=[{origin, last, R_k, #B_k}, ...])────────► │
│   Node B ──0x03 GET_PROVEN_ACCOUNT_CHUNK(bucket=0x0, origin)────────────────────► │
│   Node A ──0x04 PROVEN_ACCOUNT_CHUNK(slab136B, proofNodesRlp, R_k, #B_k)────────► │
│   Node B ──0x07 GET_BAL_RANGE(startBlock=#C+1, count=32)────────────────────────► │
│   Node A ──0x08 BAL_RANGE([(num, headerRlp, balRlp), ...])──────────────────────► │
│                                                                                   │
│   Node B Verifies Locally:                                                        │
│     1. Header + BAL chain (#B_k .. #H) matches CL-verified head (@ethereumjs/cl)  │
│     2. verifyMerkleRangeProof(R_k, origin, last, slab136B, proofNodesRlp) == true │
│     3. Applies BALs (#B_k + 1 .. #H) via consumeBAL -> Reaches ★ Live Head!       │
│     4. Node B also saves <b_hex>.proof.idx and becomes a new ETHY/0 Seeder!       │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Cryptographic Trust Model & End-to-End Verification Chain

An `ETHY/0` client treats all `ETHY/0` peers as **untrusted data mules**. Every byte received over `ETHY/0` must pass cryptographic verification before being committed to state:

### 3.1 Anchor Truth (`@ethereumjs/beacon-light` + Header Chain)
1. **Consensus Layer Head & Finality:** The client obtains the validator-signed execution `blockHash` and `stateRoot` at the head/finalized slot via `@ethereumjs/beacon-light` ($\ge 342 / 512$ Sync Committee BLS12-381 aggregate signature + SHA-256 SSZ Merkle branch).
2. **Contiguous Header + BAL Chain (`#B_min .. #Head`):**
   - Every block header in the BAL tail is verified via `keccak256(headerRlp) === childHeader.parentHash`, linking deterministically up to the CL-verified `blockHash` (or best verified P2P head when operating in `p2p_only` mode).
   - Every Block-Level Access List (`balRlp`) for block `#B` is verified against its header commitment:
     `keccak256(balRlp) === header.blockAccessListHash`

### 3.2 Per-Chunk State Proof Verification (`PROVEN_ACCOUNT_CHUNK`)
Because an initial `snap/1` sync with rolling pivots (`WP-04k`) may step its pivot root every 16 blocks ($R_0$ at $\#P_0$, $R_1$ at $\#P_1$, $\dots$), each self-proving chunk carries the exact `(chunkBlockNumber, chunkStateRoot)` against which its boundary proof was generated:
1. The receiver checks that `(chunkBlockNumber, chunkStateRoot)` matches the verified header at `chunkBlockNumber` in the header chain (`chunkBlockNumber <= basePivotBlock`).
2. The receiver decodes `binaryAccountsSlab` ($N \times 136$ bytes), checks that `accountHash` keys are strictly ascending within `[originHash .. lastHash]`, and reconstructs each account's canonical 4-element RLP body `[nonce, balance, storageRoot, codeHash]`.
3. The receiver runs standard Merkle Patricia Trie range proof verification:
   ```ts
   const hasRight = await verifyMerkleRangeProof(
     chunkStateRoot,
     originHash,
     lastHash,
     accountHashes,
     accountRlpValues,
     proofNodes,
   )
   ```
4. If `chunkBlockNumber < basePivotBlock` (because the seeder stepped its pivot during its initial sync), the receiver replays the verified BALs for `(chunkBlockNumber + 1) .. basePivotBlock` restricted to `[originHash .. lastHash]`, bringing the chunk to `basePivotBlock` (`#C`).

### 3.3 Contract Bytecode & Storage Verification (`PROVEN_STORAGE_OR_CODE`)
1. **Bytecode Batch (`kind = 0`):** For every returned bytecode blob `code`, the receiver verifies `code.length <= MAX_CONTRACT_CODE_SIZE` and `keccak256(code)` matches a `codeHash` requested in the corresponding `GET_PROVEN_STORAGE_OR_CODE` message (referenced by a verified account's `codeHash`). Verified blobs are persisted in the immutable, cross-pivot shared `code/` directory.
2. **Chunked Large-Contract Storage Range (`kind = 1`):**
   - Check `slotsSlab64B.length % 64 === 0` and verify `storageRoot` matches the account's verified 32-byte `storageRoot` at `chunkBlockNumber` (linked to the verified header chain).
   - Decode each 64-byte `BinaryStorageSlot` row (`32B slotHash + 32B valueBE`, § 5.3), verify `slotHash` keys are strictly ascending within `[originSlotHash .. lastSlotHash]` and no `valueBE` is all-zero, reconstruct each canonical MPT leaf value `RLP.encode(unpad(valueBE))`, and run `verifyMerkleRangeProof(storageRoot, originSlotHash, lastSlotHash, slotHashes, slotRlpValues, proofNodes)`.
   - If `chunkBlockNumber < basePivotBlock`, replay verified BALs for `(chunkBlockNumber + 1) .. basePivotBlock` on slots within `[originSlotHash .. lastSlotHash]`.
3. **Batched Complete-Trie Small Contracts (`kind = 2`):**
   - For each returned `[accountHash, storageRoot, slotsSlab64B]` entry, check `slotsSlab64B.length % 64 === 0` and that `(accountHash, storageRoot)` was requested and matches a verified account.
   - Rebuild the complete storage MPT root directly from `slotsSlab64B` (zero `proofNodes` needed) and verify `computedRoot === storageRoot` (or `slotsSlab64B.length === 0 && storageRoot === KECCAK256_RLP`).
4. **Failure Policy (All Messages):** Any failed Merkle range proof, root mismatch, unsorted/duplicate key, unsolicited bytecode hash, or header/BAL hash mismatch MUST trigger immediate disconnect (`DISCONNECT_REASON.SUBPROTOCOL_ERROR`) and peer quarantine (e.g., 24h ban). Empty / partial responses indicating a local miss are valid and MUST NOT penalize the peer.

---

## 4. Capability Negotiation, ENR Discovery & `ETHY_PEX`

### 4.1 RLPx `HELLO` Capability Negotiation
- Subprotocol name: **`"ethy"`** (lowercase ASCII, 4 bytes: `0x65746879`).
- Subprotocol version: **`0`** (encoded as `intToBytes(0)` in `HELLO` capabilities).
- Subprotocol message-ID space length: **`11`** (`0x00` through `0x0a`).
- Standard Ethereum clients (`Geth`, `Reth`, `Nethermind`, `Besu`, `Erigon`) that do not implement `ethy/0` simply ignore `["ethy", 0]` during `HELLO` capability intersection and mount `eth/69–71` + `snap/1` normally.

### 4.2 EIP-778 ENR Discovery Key (`"ethy"`)
Nodes supporting `ETHY/0` SHOULD include the key `"ethy"` in their signed [EIP-778](https://eips.ethereum.org/EIPS/eip-778) Ethereum Node Record (ENR):
- **Key:** `"ethy"` (ASCII `0x65746879`)
- **Value:** RLP-encoded highest supported `ethy` subprotocol version (`uint`). For `ETHY/0`, `RLP.encode(0)` is `0x80` (empty byte string).
- When a node discovers a candidate peer via discovery v4 (`ENRResponse`), discovery v5, or an EIP-1459 DNS tree whose ENR contains `"ethy"`, it marks that peer as `ethyCapable = true` and prioritizes dialing it into reserved `ETHY/0` peer slots.

### 4.3 Built-In Epidemic Peer Exchange (`ETHY_PEX`) & Bidirectional NAT Serving
Because early `ETHY/0` deployments may represent a small fraction of the global DHT, `ETHY/0` embeds a lightweight **Peer Exchange (`GET_ETHY_PEERS` / `ETHY_PEERS`)** directly in the subprotocol:
1. Immediately after completing an `ETHY/0 STATUS` handshake with any peer, a node sends `GET_ETHY_PEERS (0x09)`.
2. The remote peer replies with `ETHY_PEERS (0x0a)`, listing up to `32` recently seen `ETHY/0` peers along with their last advertised `basePivotBlock` and `bucketsBitmask`.
3. **Bidirectional Serving Over Outbound TCP Sockets (NAT Traversal):**
   - In DevP2P `RLPx`, once a TCP connection is established, subprotocol streams are **100% symmetric and bidirectional** regardless of which side initiated the TCP `SYN`.
   - Therefore, an `ETHY/0` node running on a home laptop behind a NAT router (without port forwarding) that makes outbound TCP connections to other `ETHY/0` peers **can serve `PROVEN_ACCOUNT_CHUNK` and `BAL_RANGE` responses over those outbound TCP sockets** just as effectively as a public-IP node!

---

## 5. Wire Encoding Conventions & Binary Data Layouts

### 5.1 Canonical RLP Scalar Rule
All `ETHY/0` messages are RLP-encoded arrays compressed with Snappy (standard DevP2P base protocol `v5`).
Every unsigned integer scalar (`uint8`, `uint16`, `uint64`, `uint256`) in `ETHY/0` RLP tuples MUST use **canonical minimal big-endian encoding**:
- Leading `0x00` bytes MUST be stripped.
- The integer `0` MUST be encoded as the empty byte array `new Uint8Array(0)` (RLP byte `0x80`), **never** `0x00` (`Uint8Array([0])`).

### 5.2 Fixed 136-Byte `BinaryAccount` Slab Row Layout (`binaryAccountsSlab`)
To avoid per-account RLP list decoding and V8/heap allocation overhead during high-speed state transfer (>100,000 accounts/sec), `PROVEN_ACCOUNT_CHUNK (0x04)` transmits accounts as a flat packed byte array (`binaryAccountsSlab: Uint8Array`) whose byte length is a multiple of **`136` bytes** (`32`-byte `accountHash` + `104`-byte `BinaryAccount` from `@ethereumjs/util`):

```text
Offset   Size    Field          Encoding
──────   ────    ─────────────  ───────────────────────────────────────────────────────
0..32    32 B    accountHash    keccak256(address) — strictly ascending across slab!
32..40    8 B    nonce          uint64 big-endian (fixed 8 bytes)
40..72   32 B    balance        uint256 big-endian (fixed 32 bytes)
72..104  32 B    storageRoot    32-byte MPT storage root (KECCAK256_RLP if empty)
104..136 32 B    codeHash       32-byte bytecode hash (KECCAK256_NULL if EOA)
──────   ────
Total:  136 B per account row
```

### 5.3 Fixed 64-Byte `BinaryStorageSlot` Slab Row Layout (`slotsSlab64B`)
Matching the packed `BinaryAccount` slab convention, `PROVEN_STORAGE_OR_CODE (0x06)` (`kind = 1` and `kind = 2`) transmits contract storage entries as a flat packed byte array (`slotsSlab64B: Uint8Array`) whose byte length is a multiple of **`64` bytes**:

```text
Offset   Size    Field          Encoding
──────   ────    ─────────────  ───────────────────────────────────────────────────────
0..32    32 B    slotHash       keccak256(slot32) — strictly ascending across slab!
32..64   32 B    slotValueBE    uint256 big-endian (fixed 32 bytes; MUST be non-zero)
──────   ────
Total:   64 B per storage slot row
```

Because zero-valued storage slots never exist in Ethereum's storage MPT, stripping leading `0x00` bytes from `slotValueBE` and applying `RLP.encode(unpad(slotValueBE))` is a **100% lossless bijection** with the canonical MPT leaf value.

### 5.4 On-Disk Proof Sidecar & Shared Store Layout
An `ETHY/0` implementation that ingests verified slabs stores them alongside compact proof sidecars and shares immutable bytecodes and verified BALs with [`STATY/0`](./STATY-0-SPEC.md):
- **Shared Immutable Bytecode Store (`code/`):** Content-addressed by `keccak256(code)`; never duplicated across pivot directories.
- **Shared Verified BAL Archive (`bals/`):** Contiguous `(headerRlp, balRlp)` bundles serving both `ETHY/0 GET_BAL_RANGE` and `eth/71`.
- **Account Proof Sidecars (`pivots/<stateRoot>/accounts/<nibble>.proof.idx`):**
  - Magic Header (8 bytes): `0x4554485950524631` (ASCII `"ETHY"` + `"PRF1"`).
  - Each chunk entry is `[ uint32BE(recordByteLength), RLP.encode([ slabByteOffset, slabByteLength, originHash, lastHash, hasRight, chunkStateRoot, chunkBlockNumber, proofNodes ]) ]`.
- **Storage Slabs & Large-Contract Proof Sidecars (`pivots/<stateRoot>/storage/<nibble>.bin` + `.proof.idx`):**
  - Small contracts whose complete storage trie fits in a single response (`kind = 2`) require **zero** proof sidecar bytes (their complete `slotsSlab64B` hashes directly to `storageRoot`!).
  - Only multi-chunk large contracts (`kind = 1`, where `hasRight = 1` or `originSlotHash > 0`) append their ~3–5 KB left/right boundary proof to `storage/<nibble>.proof.idx`.

---

## 6. Message Specification (`0x00` – `0x0a`)

| Code | Name | Direction | Description |
| --- | --- | --- | --- |
| `0x00` | `STATUS` | Peer ↔ Peer | Initial handshake + live updates advertising snapshot root, per-plane 16-bit bucket masks (`bucketsBitmask`, `storageBucketsBitmask`, `codeBucketsBitmask`), and servable BAL range. |
| `0x01` | `GET_BUCKET_MANIFEST` | Client → Server | Request the ordered list of self-proving chunk descriptors for account bucket `0x0..0xf`. |
| `0x02` | `BUCKET_MANIFEST` | Server → Client | Returns the chunk manifest for the requested bucket. |
| `0x03` | `GET_PROVEN_ACCOUNT_CHUNK` | Client → Server | Request one self-proving `BinaryAccount` slab + Merkle boundary proof by `(bucketIndex, originHash)`. |
| `0x04` | `PROVEN_ACCOUNT_CHUNK` | Server → Client | Returns the 136B-packed `binaryAccountsSlab`, `proofNodes`, `chunkStateRoot`, and `chunkBlockNumber`. |
| `0x05` | `GET_PROVEN_STORAGE_OR_CODE` | Client → Server | Request bytecode batches (`kind = 0`), a chunked large-contract storage range (`kind = 1`), or batched complete-trie small contracts (`kind = 2`). |
| `0x06` | `PROVEN_STORAGE_OR_CODE` | Server → Client | Returns verified contract code blobs, packed 64B storage range + boundary proof, or batched complete-trie small-contract slabs. |
| `0x07` | `GET_BAL_RANGE` | Client → Server | Request up to 32 contiguous `(blockNumber, headerRlp, balRlp)` bundles starting at `startBlock`. |
| `0x08` | `BAL_RANGE` | Server → Client | Returns contiguous `(blockNumber, headerRlp, balRlp)` bundles for post-pivot BAL healing / catch-up. |
| `0x09` | `GET_ETHY_PEERS` | Peer → Peer | Request up to `maxPeers` recently active `ETHY/0` peers (`ETHY_PEX`). |
| `0x0a` | `ETHY_PEERS` | Peer → Peer | Returns peer descriptors `[peerId, ip, tcpPort, udpPort, basePivotBlock, bucketsBitmask, ageSec]`. |

---

### `0x00` `STATUS`

Sent immediately upon mounting the `ETHY/0` subprotocol, and **re-sent whenever `bucketsBitmask`, `storageBucketsBitmask`, `codeBucketsBitmask`, or `balLatestBlock` changes** (so connected peers learn immediately when a new account/storage/code bucket becomes servable or the BAL tail advances).

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
  // --- v0.3 trailing per-plane completeness fields (backward-compatible with v0.2 9-tuple decoders) ---
  storageBucketsBitmask: uint16,  // Optional (default 0): bit b is 1 iff 100% of storage in bucket b is servable
  codeBucketsBitmask: uint16      // Optional (default 0): bit b is 1 iff 100% of distinct codeHashes in bucket b are servable
]
```

**Validation & Compatibility Rules:**
- If `protocolVersion !== 0`, `chainId` mismatches local chain ID, or `genesisHash` mismatches local genesis hash, the receiver MUST disconnect with `DISCONNECT_REASON.SUBPROTOCOL_ERROR` (or `USELESS_PEER`).
- **Backward Compatibility (`v0.2` ↔ `v0.3`):** Decoders MUST accept `payload.length >= 9`, defaulting omitted trailing fields `storageBucketsBitmask` (index `9`) and `codeBucketsBitmask` (index `10`) to `0` (and ignoring any future fields beyond index `10`). A `v0.3` peer MUST set `storageBucketsBitmask` / `codeBucketsBitmask` bits **only** when 100% of that plane for bucket `b` is verified and servable — eliminating any ambiguity between accounts-only peers and full-state seeders.
- Unlike `eth/69` `STATUS` (which may only be sent once), **`ETHY/0 STATUS` MAY be sent multiple times during a session** to push updated bucket bitmasks and `balLatestBlock` values without polling!

---

### `0x01` `GET_BUCKET_MANIFEST` & `0x02` `BUCKET_MANIFEST`

Queries a peer for the list of self-proving chunks that make up account bucket `bucketIndex` (`0..15`, corresponding to `accountHash` high nibble `0x0..0xf`).

**Request (`0x01 GET_BUCKET_MANIFEST`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8              // 0..15
]
```

**Response (`0x02 BUCKET_MANIFEST`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8,
  chunks: [
    [
      originHash: bytes32,        // Request origin passed to verifyMerkleRangeProof
      lastHash: bytes32,          // Last accountHash in this chunk
      hasRight: uint8,            // 1 if more keys exist in trie to the right of lastHash, else 0
      chunkStateRoot: bytes32,    // 32-byte stateRoot against which proofNodes were verified
      chunkBlockNumber: uint64,   // Pivot block number corresponding to chunkStateRoot
      accountCount: uint32,       // Number of 136-byte rows in this chunk
      slabByteLength: uint32      // Byte length of binaryAccountsSlab (accountCount * 136)
    ],
    ...
  ]
]
```
- If the server does not have bucket `bucketIndex` available, it MUST reply with `chunks = []` (never hang or disconnect).

---

### `0x03` `GET_PROVEN_ACCOUNT_CHUNK` & `0x04` `PROVEN_ACCOUNT_CHUNK`

Requests and transfers one self-proving account chunk identified by `(bucketIndex, originHash)`.

**Request (`0x03 GET_PROVEN_ACCOUNT_CHUNK`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8,             // 0..15
  originHash: bytes32             // 32-byte chunk originHash from BUCKET_MANIFEST (or cursor origin)
]
```

**Response (`0x04 PROVEN_ACCOUNT_CHUNK`):**
```text
[
  requestId: uint64,
  bucketIndex: uint8,
  originHash: bytes32,
  lastHash: bytes32,
  hasRight: uint8,                // 0 or 1
  chunkStateRoot: bytes32,
  chunkBlockNumber: uint64,
  binaryAccountsSlab: bytes,      // N * 136 bytes (see § 5.2)
  proofNodes: [bytes, ...]        // RLP-encoded MPT boundary proof nodes for verifyMerkleRangeProof
]
```
- **Empty / Miss Response:** If the peer does not hold the requested chunk, it replies with `binaryAccountsSlab = 0x80` (empty bytes) and `proofNodes = []`.
- **Receiver Verification:**
  1. Check `binaryAccountsSlab.length % 136 === 0`.
  2. Expand each 136-byte row into `(accountHash, accountRlp)` and run `verifyMerkleRangeProof(chunkStateRoot, originHash, lastHash, accountHashes, accountRlpValues, proofNodes)`.
  3. If `chunkBlockNumber < basePivotBlock`, replay verified BALs for `(chunkBlockNumber + 1) .. basePivotBlock` on hashes within `[originHash .. lastHash]`.

---

### `0x05` `GET_PROVEN_STORAGE_OR_CODE` & `0x06` `PROVEN_STORAGE_OR_CODE`

Transfers content-addressed contract code blobs (`kind = 0`), a chunked proven storage range for a large contract (`kind = 1`), or a high-throughput batch of complete-trie small contracts (`kind = 2`).

> **Why no `GET_STORAGE_MANIFEST` message is needed:** Unlike account ranges (whose chunk boundaries are unknown ahead of time), the receiver already learns every contract's exact `(accountHash, storageRoot, codeHash)` from its verified account bucket. It can therefore immediately batch small contracts with `kind = 2` and cursor-stream large contracts with `kind = 1` (`originSlotHash` → `lastSlotHash + 1` until `hasRight = 0`), keeping `ETHY/0` at `length = 11` with zero breaking changes.

**Request (`0x05 GET_PROVEN_STORAGE_OR_CODE`):**
- When `kind = 0` (Bytecode Batch, `1..64` `codeHashes` — backed by shared `code/`):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 0 (encoded as 0x80)
    codeHashes: [bytes32, ...],   // 1..64 distinct 32-byte keccak256 codeHashes
    maxResponseBytes: uint64      // Soft budget (clamped to <= 524,288)
  ]
  ```
- When `kind = 1` (Chunked Large-Contract Storage Range for one `accountHash`):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 1
    accountHash: bytes32,
    storageRoot: bytes32,         // Expected 32-byte storageRoot from verified account
    originSlotHash: bytes32,      // 0x00..00 on first request; previous lastSlotHash + 1 thereafter
    maxResponseBytes: uint64      // Soft budget (clamped to <= 524,288)
  ]
  ```
- When `kind = 2` (Batched Complete-Trie Small Contracts — `v0.3`, `1..64` accounts):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 2
    accounts: [                   // 1..64 contracts whose complete storage tries are requested
      [accountHash: bytes32, storageRoot: bytes32],
      ...
    ],
    maxResponseBytes: uint64      // Soft budget (clamped to <= 524,288)
  ]
  ```

**Response (`0x06 PROVEN_STORAGE_OR_CODE`):**
- When `kind = 0` (Bytecode Batch):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 0 (encoded as 0x80)
    codes: [bytes, ...]           // Misses omitted; receiver verifies keccak256(code) === requestedCodeHash
  ]
  ```
- When `kind = 1` (Chunked Large-Contract Storage Range):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 1
    accountHash: bytes32,
    storageRoot: bytes32,         // 32-byte storageRoot against which proofNodes were verified
    chunkBlockNumber: uint64,     // Pivot block number corresponding to storageRoot
    originSlotHash: bytes32,
    lastSlotHash: bytes32,
    hasRight: uint8,              // 1 if more slots exist to the right of lastSlotHash, else 0
    slotsSlab64B: bytes,          // N * 64 bytes packed BinaryStorageSlot rows (see § 5.3; 0x80 on miss)
    proofNodes: [bytes, ...]      // Empty [] when complete storage trie is returned from origin 0x00..00
  ]
  ```
- When `kind = 2` (Batched Complete-Trie Small Contracts — zero `proofNodes`):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 2
    entries: [                    // Misses or multi-chunk contracts omitted (client retries omitted via kind = 1)
      [
        accountHash: bytes32,
        storageRoot: bytes32,
        slotsSlab64B: bytes       // Complete storage trie packed as N * 64B rows; receiver checks computedRoot === storageRoot
      ],
      ...
    ]
  ]
  ```
- **Miss / Split Fallback:** In `kind = 2`, if a requested contract is missing or too large to fit as a complete trie within `maxResponseBytes`, the server simply omits it from `entries[]`. The client falls back to `kind = 1` cursor-streaming for omitted large contracts without penalizing the peer.

---

### `0x07` `GET_BAL_RANGE` & `0x08` `BAL_RANGE`

Streams a contiguous batch of verified execution block headers and their EIP-7928 Block-Level Access Lists (`BAL`) in a single round-trip (up to `32` blocks per request) so a joining node can rapidly heal from `basePivotBlock` (`#C`) up to `#Head`.

**Request (`0x07 GET_BAL_RANGE`):**
```text
[
  requestId: uint64,
  startBlock: uint64,             // First block number requested (e.g. #C + 1)
  count: uint8                    // Max blocks requested (1..32)
]
```

**Response (`0x08 BAL_RANGE`):**
```text
[
  requestId: uint64,
  blocks: [
    [
      blockNumber: uint64,
      headerRlp: bytes,           // RLP-encoded execution BlockHeader
      balRlp: bytes               // Canonical RLP-encoded BlockLevelAccessList (empty bytes if none)
    ],
    ...
  ]
]
```
- **Receiver Verification:**
  1. Decode `headerRlp`, verify `header.number === blockNumber` and `header.parentHash === prevBlockHash`.
  2. Decode `balRlp` (`createBlockLevelAccessListFromRLP(balRlp)`) and verify `equalsBytes(bal.hash(), header.blockAccessListHash)`.
  3. Apply via `stateManager.consumeBAL(bal.toJSON(), { shouldApplyAccount })`.

---

### `0x09` `GET_ETHY_PEERS` & `0x0a` `ETHY_PEERS` (`ETHY_PEX`)

Exchanges known `ETHY/0` peer endpoints and their snapshot capabilities to form a self-healing swarm mesh without requiring centralized discovery servers.

**Request (`0x09 GET_ETHY_PEERS`):**
```text
[
  requestId: uint64,
  maxPeers: uint8                 // 1..32 (recommended default: 16)
]
```

**Response (`0x0a ETHY_PEERS`):**
```text
[
  requestId: uint64,
  peers: [
    [
      peerId: bytes,              // 64-byte secp256k1 uncompressed public key (without 0x04 prefix)
      address: bytes,             // UTF-8 IPv4/IPv6 string (e.g. "203.0.113.10")
      tcpPort: uint16,
      udpPort: uint16,
      basePivotBlock: uint64,
      bucketsBitmask: uint16,
      lastSeenAgeSec: uint32      // Seconds since this peer was last active
    ],
    ...
  ]
]
```

---

## 7. Sync Source Selection & Fallback Algorithm (`ETHY/0` vs `SNAP/1`)

Because an Ethy client speaks both **`ETHY/0`** and **`SNAP/1`**, it must decide cleanly at startup whether to onboard its base state from the **`ETHY/0` Swarm** or from **`SNAP/1`** without getting stuck or thrashing between incompatible approaches.

### 7.1 When Is an `ETHY/0` Peer Considered "Usable" (`isUsableEthySeeder`)?
A connected `ETHY/0` peer $P$ is usable for state onboarding if:
1. For account sync: $P.\text{bucketsBitmask} \neq 0$ (it holds at least one complete, proven account bucket). For storage or bytecode sync, the scheduler filters peers by `P.storageBucketsBitmask & (1 << b)` and `P.codeBucketsBitmask & (1 << b)` respectively (preferring peers advertising full 3-plane completeness, while still allowing `kind = 0` bytecode queries against any peer that shares `code/`), AND
2. **Contiguous BAL Bridge Guarantee:** Unless $P.\text{basePivotBlock} \ge \text{bestPeerHead}$ (zero tail needed) or another connected peer covers the missing gap, $P$ MUST advertise a contiguous BAL tail anchored directly to its base snapshot:
   ```text
   P.balEarliestBlock <= P.basePivotBlock + 1
   ```
   This applies to **all** seeders, including full `bucketsBitmask = 0xffff` seeders. A peer whose local BAL history has a gap (`balEarliestBlock > basePivotBlock + 1`) MUST NOT be selected as a standalone seeder, and servers MUST honestly report their true contiguous `balEarliestBlock`.
3. Either:
   - $P$'s `basePivotBlock` (`#C`) is within the hard BAL tail ceiling (`#Head - #C <= MAX_ETHY_BAL_TAIL_BLOCKS`, default `64,800` blocks $\approx 9$ days), OR
   - The local node is explicitly started with `--sync-mode ethy` (e.g. local 2-node testing).

### 7.2 Deterministic 3-Mode Startup Ruleset (`--sync-mode auto | ethy | snap`)

```text
                        ┌───────────────────────────────┐
                        │ Node Starts (Fresh / No Disk) │
                        └───────────────┬───────────────┘
                                        │
                   ┌────────────────────┼────────────────────┐
                   ▼                    ▼                    ▼
          --sync-mode ethy      --sync-mode auto     --sync-mode snap
                   │                    │                    │
                   ▼                    ▼                    ▼
         Wait for usable        Wait 2,500ms (or up   Anchor canonical
         ETHY/0 peer &          to 10,000ms while     K=32 pivot on
         sync via ETHY/0        "ethy" dials active)  SNAP/1 (while
         exclusively               │          │       seeding ETHY/0!)
                        Found! ◄───┘          └───► Timeout (0 ETHY peers)
                          │                                  │
                          ▼                                  ▼
                Lock transport = 'ethy'            Lock transport = 'snap'
                Sync proven chunks +               Sync via SNAP/1 + BALs
                BAL_RANGE (#C+1..#H)               (early preemption if fresh
                                                   ETHY/0 joins < 5% done!)
```

1. **Immediate Selection on Direct Connect (`--connect` / `peers.json` VIP):**
   - If a usable `ETHY/0` peer connects and sends a valid `ETHY/0 STATUS` before the grace window expires, the node immediately locks `syncTransport = 'ethy'` (zero extra wait).
2. **Adaptive Discovery Grace Window & Early `SNAP -> ETHY` Preemption in `auto` Mode:**
   - When zero `"ethy"` ENR / `PeerBook` candidates are in flight, the node waits at most **`2,500ms`** after its first `ETH STATUS` before locking `SNAP/1`.
   - When `"ethy"` ENR or cached `ETHY/0` dials are actively in flight, the node MAY extend grace up to **`10,000ms`** so low-density public DHTs do not prematurely force `SNAP/1`.
   - If a node locks `SNAP/1`, is still early in account sync (`< 5%` complete, zero finished buckets), and a fresh usable `ETHY/0` seeder (`#Head - basePivotBlock <= SOFT_REFRESH_AGE_BLOCKS`) connects, the client SHOULD cancel the early `SNAP/1` pass and switch cleanly to `ETHY/0`.
3. **Bucket-Granular Seeding Regardless of Sync Mode:**
   - Even when a node syncs via `SNAP/1` (`syncTransport = 'snap'`), it writes `accounts/<nibble>.proof.idx` for every verified slab and flips bit `1 << b` in its outbound `ETHY/0 STATUS` `bucketsBitmask` as each bucket `0x0..0xf` completes — immediately becoming an `ETHY/0` seeder for subsequent nodes!

---

## 8. Swarm Convergence, Amortized `O(1)` Snapshot Refresh & Long-Lived Seeding

All mechanisms in this section operate purely over **`ETHY/0` (`0x00`–`0x0a`) with zero breaking wire-protocol changes**, using the existing `STATUS` (`basePivotBlock`, `basePivotStateRoot`, `bucketsBitmask`, `balEarliestBlock`, `balLatestBlock`) and `ETHY_PEERS` fields.

### 8.1 Canonical Pivot Grid (`K = 32`) & Macro-Generation Grid (`G = 1,024`)
To prevent snapshot fragmentation across independent `SNAP/1` bootstrappers:
1. **Fine Grid (`CANONICAL_PIVOT_GRID_BLOCKS = 32` — 1 Beacon Epoch):**
   - Every `SNAP/1` anchor block and rolling-pivot step MUST align to a multiple of `32` (`blockNumber % 32 == 0`):
     - If `head % 32 <= 24`: snap backward immediately to `head - (head % 32)` (`0` wait; leaves $\ge 104$ blocks inside standard 128-block `snap/1` windows).
     - If `head % 32 > 24`: wait $1..7$ blocks for `head + (32 - head % 32)` while continuing to accept incoming `ETHY/0` seeders.
2. **Macro-Generation Grid (`MACRO_GENERATION_GRID_BLOCKS = 1,024` $\approx 3.4\text{h}$):**
   - Scheduled swarm-wide generation refreshes target multiples of `1,024` blocks ($B_{\text{gen}} = \lfloor H / 1024 \rfloor \cdot 1024$, which is always on the `K = 32` grid) so the entire swarm converges onto at most `1–2` active generations at any time.

### 8.2 Contiguous BAL Tail Serving Invariant (Long-Lived & Intermittent Nodes)
Because flat-state nodes do not retain an 80 GB interior MPT database, live `eth/71` BAL updates advance `FlatStateManager` while the on-disk `.bin` + `.proof.idx` files remain cryptographically bound to `basePivotBlock`.
- **Contiguous Disk Archive:** An `ETHY/0` seeder MUST persist verified `(blockNumber, headerRlp, balRlp)` bundles continuously during `SNAP_SYNCING`, `BAL_CATCHUP`, **and** `HEAD_FOLLOWING` (up to a bounded disk retention budget, recommended `MAX_SERVABLE_BAL_TAIL_BLOCKS = 8,192` $\approx 27.3\text{h}$) rather than relying solely on a 64-block in-memory `RollingBlockStore`.
- **No Premature Pruning:** During rolling-pivot `SNAP/1` sync, a client MUST NOT prune persisted BAL bundles at or above $\min_{b}(\text{chunkBlockNumber}_b) + 1$ while any bucket still serves chunks verified at that earlier block.
- **Honest Gap Reporting:** If local disk retention evicts blocks above `basePivotBlock + 1`, the server MUST advertise its true contiguous `balEarliestBlock` in `ETHY/0 STATUS` so joiners never encounter a mid-tail hole.

### 8.3 Two-Tier Freshness Horizons & Amortized `O(1)` Decentralized Minter Election
A naive hard age threshold (e.g., *"if freshest `ETHY/0` pivot $> 2,048$ blocks old, run `SNAP/1`"*) creates a **synchronized `O(N)` `SNAP/1` thundering herd** whenever a swarm sharing generation $G_0$ crosses the cliff simultaneously. `ETHY/0` clients prevent this via two-tier horizons plus decentralized hash-lottery election:

```text
  basePivotBlock (#G_0)
  │◄── Fresh Swarm Window ──►│◄── Pre-Emptive O(1) Election ──►│◄── Hard Fallback ──►│
  0                     1,536 blk (5.1h)                  4,096 blk (13.6h)     64,800 blk (9d)
                        T_soft                            T_hard                T_max
  • Joiners: 100% ETHY/0     • Joiners: STILL 100% ETHY/0      • Auto joiners enter  • Hard reject
  • Seeders: hold G_0        • Online seeders run O(1) hash    • jittered SNAP/1       unless explicit
                               lottery to mint G_1 in          • election if no G_1    --sync-mode ethy
                               background BEFORE G_0 expires!    exists or in-flight
```

1. **Three Horizons:**
   - **`SOFT_REFRESH_AGE_BLOCKS = 1,536`** ($\approx 5.1\text{h}$): $G_0$ remains fully usable for all joining nodes while eligible online nodes begin decentralized $O(1)$ election to mint the next macro-generation $B_{\text{gen}}$ proactively.
   - **`HARD_PREFER_SNAP_AGE_BLOCKS = 4,096`** ($\approx 13.6\text{h}$): If no newer or verified in-flight generation exists beyond `4,096` blocks, joining `auto` nodes also participate in jittered `SNAP/1` election.
   - **`MAX_ETHY_BAL_TAIL_BLOCKS = 64,800`** ($\approx 9\text{d}$): Hard cap beyond which a snapshot is unusable in `auto` mode.
2. **Deterministic Hash-Lottery Order-Statistic Delay:**
   - Each eligible node computes a deterministic uniform score $u \in (0, 1)$ from `keccak256(localNodeId || bigIntToBytes(B_gen))`, estimates local swarm size $\hat{N} = \max(1, \text{knownActiveEthyPeers} + 1)$ from connected `ETHY/0` peers and `ETHY_PEX`, and applies an uptime/reachability weight $w \in [0.25, 4.0]$ (favoring long-uptime, non-NAT, well-connected nodes over newly woken laptops):
     $$\Delta t = \text{clamp}\!\left(W_{\min} + \left(-\ln(1 - u)\right) \cdot \frac{\hat{N} \cdot T_{\text{target}}}{\lambda_{\text{target}}} \cdot w,\; W_{\min},\; W_{\max}\right)$$
   - Because the minimum of $N$ independent $\text{Exp}(\lambda / N)$ variables is distributed as $\text{Exp}(\lambda)$ independent of $N$, **only $O(1)$ nodes (typically `1–2`) fire per generation** whether the swarm has `10` or `10,000` nodes!
3. **Suppression & Adversarial Fake-Minter Defense:**
   - When the winning volunteer begins minting $B_{\text{gen}}$, it advertises `basePivotBlock = B_gen, basePivotStateRoot = root(B_gen), bucketsBitmask = 0x0000` (and flips bits `1 << b` as buckets finish). Connected peers and `ETHY_PEX` gossip propagate this announcement so other nodes cancel their pending `SNAP/1` timers.
   - **Security Invariant (Anti-Suppression Attack):** A peer claiming `basePivotBlock >= B_gen` with `bucketsBitmask = 0x0000` MUST ONLY suppress local election if:
     1. `basePivotBlock <= bestVerifiedHead` and (for direct peers) `basePivotStateRoot` matches the verified canonical execution header at `basePivotBlock`, **AND**
     2. The peer reaches `bucketsBitmask > 0` (with at least one verified `BUCKET_MANIFEST` / chunk) within `MINT_LIVENESS_TIMEOUT_MS` (recommended **`300,000ms`** / 5 minutes).
   - If the announcing minter crashes or stalls without producing a verified bucket within the liveness timeout, suppression expires automatically and the next-ranked volunteer takes over.

### 8.4 Zero-Downtime Background Shadow Re-Seeding
To prevent long-lived or intermittently opened nodes (e.g., laptops opened every 2–3 days) from permanently falling out of `ETHY/0` state serving:
1. **`ETHY/0`-First Shadow Upgrade (Zero `SNAP/1` Load):** When a node in `HEAD_FOLLOWING` detects that its local `basePivotBlock` exceeds `SOFT_REFRESH_AGE_BLOCKS` (or its local BAL tail has a gap) and a newer verified `ETHY/0` generation $G_{\text{new}}$ exists in the swarm, it SHOULD download $G_{\text{new}}$'s self-proving chunks over **`ETHY/0`** into a shadow directory (`pivots/<newStateRoot>/`), preserve contiguous BALs `[G_new + 1 .. lastVerifiedBlock]`, atomically flip its active serving directory and `ETHY/0 STATUS`, and prune the superseded pivot directory — with zero downtime to live RPC or head following.
2. **Elected `SNAP/1` Shadow Mint:** Only when no newer `ETHY/0` generation exists and no verified minter is in flight does the node participate in the § 8.3 hash-lottery to perform a background `SNAP/1` shadow mint.

### 8.5 Rarest-First Striping, Cross-Snapshot Composition & Mesh Health
1. **Rarest-First + Node-Permuted Bucket Order:** Joining nodes MUST NOT fetch buckets in fixed ascending `0x0..0xf` order (which causes all flash-crowd leechers to hammer bucket `0x0` on a single origin seeder). Instead, clients SHOULD prioritize buckets with the **lowest availability count** across connected seeders and break ties with a per-node permutation (`keccak256(localNodeId || bucketIndex)`), turning single-seeder flash crowds into $O(\log N)$ BitTorrent-style cross-seeding.
2. **Cross-Snapshot Bucket Composition:** Because each bucket `0x0..0xf` is disjoint in `accountHash` space and every chunk carries `(chunkBlockNumber, chunkStateRoot)`, a joining node MAY source different buckets from seeders on different generations (`bucketPivotBlock[b]`), provided the connected swarm covers contiguous `BAL_RANGE` blocks from $\min_b(\text{bucketPivotBlock}[b]) + 1$ through `#Head`, filtering `consumeBALBatch` per bucket for blocks $\le \max_b(\text{bucketPivotBlock}[b])$.
3. **Freshness-Biased `ETHY_PEX` & Reserved `ETHY/0` Peer Slots:**
   - When answering `GET_ETHY_PEERS (0x09)`, nodes SHOULD fill ~75% of returned entries with the freshest-pivot, highest-`bucketsBitmask`, recently active peers and ~25% with random active peers (fast generation propagation without hub collapse).
   - Multi-protocol nodes SHOULD reserve a minimum quota of peer slots (recommended `minEthyPeers = 4`) for `ETHY/0` peers so non-`ETHY/0` peers cannot evict the `ETHY/0` mesh.
   - Seeders SHOULD apply fair bounded upload concurrency (`maxConcurrentChunkReads`) so overloaded residential nodes degrade smoothly instead of timing out all leechers.

### 8.6 Wire-Compatibility Summary (`ETHY/0`)

| Mechanism | `ETHY/0` Wire Field Used | Breaking Change? |
| --- | --- | --- |
| Canonical Grid (`K = 32`) & Macro Grid (`G = 1,024`) | `STATUS.basePivotBlock`, `PROVEN_ACCOUNT_CHUNK.chunkBlockNumber` | **None** |
| Contiguous BAL Continuity Guard | `STATUS.balEarliestBlock`, `STATUS.balLatestBlock` | **None** |
| `O(1)` Minter Election Suppression | `STATUS.basePivotBlock` + `basePivotStateRoot` + `bucketsBitmask = 0x0000` & `ETHY_PEERS` | **None** |
| Rarest-First & Cross-Snapshot Striping | `STATUS.bucketsBitmask`, `BUCKET_MANIFEST`, `BAL_RANGE` | **None** |
| Background Shadow Re-Seeding | Repeat `STATUS (0x00)` with updated `basePivotBlock` / `basePivotStateRoot` | **None** |
| Per-Plane Completeness (`v0.3`) | Optional trailing `STATUS` fields `storageBucketsBitmask` (idx `9`), `codeBucketsBitmask` (idx `10`) | **None** (defaults to `0` on `v0.2` peers) |
| High-Throughput Storage Seeding (`v0.3`) | `PROVEN_STORAGE_OR_CODE` `kind = 1` (64B slab) & `kind = 2` (batched small contracts) | **None** (`length = 11` unchanged) |

---

## 9. Co-Design & Interop with `STATY/0` (Two-Protocol Suite)

`ETHY/0` is a complete, standalone full-state replication protocol, and is simultaneously co-designed with **[`STATY/0`](./STATY-0-SPEC.md)** (the BAL-Extended Active State & Witness Swarm) as a two-tier post-Amsterdam state suite:

| Mode | Protocols Mounted | Time to Live Verified RPC | Disk Footprint (Mainnet) | Privacy & Network Role |
| --- | --- | --- | --- | --- |
| **Default (`STATY/0`)** | `eth/71` + `STATY/0` | **`3–5 min`** | `~3–10 GB` | Active state local, cold misses via `STATY/0` witnesses; seeds Hot Epoch Bundles, witnesses, `code/`, and `eth/71` BALs. |
| **Full (`--full`)** | `eth/71` + `STATY/0` + `ETHY/0` | **`3–5 min`** (via `STATY/0`, while `ETHY/0` completes in background) | `~160+ GB` | **100% read privacy** (zero external queries on any bucket once closed-world) + Tier-2 MPT-free `ETHY/0` full-state seeder. |

### Shared Primitives Between `ETHY/0` and `STATY/0`
1. **Shared Block Calendar:** Both protocols align on powers of two — `K = 32` (`ETHY/0` fine pivot), `E = 256` (`STATY/0` micro-epoch), `G = 1,024` (`ETHY/0` macro-generation), `D = 4,096` (`STATY/0` Macro-Day dedup reset & `ETHY/0` `HARD_PREFER_SNAP_AGE`), and `2D = 8,192` (default rolling verified BAL archive).
2. **Shared Immutable `code/` & Verified `bals/` Directories:** Bytecode verified over `STATY/0 GET_BYTECODES (0x07/0x08)` is immediately servable over `ETHY/0 PROVEN_STORAGE_OR_CODE (kind = 0)` and vice versa; verified `(headerRlp, balRlp)` bundles in `bals/` serve both `eth/71` and `ETHY/0 GET_BAL_RANGE (0x07/0x08)`.
3. **Per-Bucket Open-World → Closed-World Upgrade:** Because both protocols partition state by `accountHash` high nibble `0x0..0xf`, a `--full` node running `FlatStateManager` starts live in Open-World `STATY/0` mode and atomically flips each bucket `b` to Closed-World (zero network lookups, 100% private) as `ETHY/0` finishes accounts, code, and storage for bucket `b`.

---

## 10. Worked Byte-Level Walkthrough

Suppose **Node A (Seeder)** has completed bucket `0x0` (accounts, storage, and bytecode) at `basePivotBlock = 12,062,000` (`0xb80d70`) with `bucketsBitmask = 0x0001`, `storageBucketsBitmask = 0x0001`, `codeBucketsBitmask = 0x0001`, and BAL tail `12,062,000 .. 12,062,016`:

1. **Node A sends `0x00 STATUS` (`v0.3`):**
   ```ts
   [
     new Uint8Array(0),                      // protocolVersion = 0 (canonical 0x80)
     bigIntToUnpaddedBytes(11155111n),       // chainId = Sepolia (0xaa36a7)
     genesisHash32,                          // 32 bytes
     bigIntToUnpaddedBytes(12062000n),       // basePivotBlock = 12,062,000
     basePivotStateRoot32,                   // 32 bytes
     intToUnpaddedBytes(0x0001),             // bucketsBitmask = 0x01 (account bucket 0x0 ready)
     bigIntToUnpaddedBytes(12062000n),       // balEarliestBlock
     bigIntToUnpaddedBytes(12062016n),       // balLatestBlock
     latestBlockHash32,                      // 32 bytes
     intToUnpaddedBytes(0x0001),             // storageBucketsBitmask = 0x01 (storage bucket 0x0 ready)
     intToUnpaddedBytes(0x0001),             // codeBucketsBitmask = 0x01 (code bucket 0x0 ready)
   ]
   ```
2. **Node B (Joiner) requests the manifest for bucket `0x0` (`0x01 GET_BUCKET_MANIFEST`):**
   ```ts
   [
     bigIntToUnpaddedBytes(1n),              // requestId = 1
     new Uint8Array(0),                      // bucketIndex = 0 (canonical 0x80)
   ]
   ```
3. **Node A replies with `0x02 BUCKET_MANIFEST`** listing the chunks in `0.proof.idx`, and **Node B requests each chunk via `0x03 GET_PROVEN_ACCOUNT_CHUNK`**, verifies `verifyMerkleRangeProof` in its worker pool, pulls contract bytecodes via `0x05 (kind = 0)` and contract storage via `0x05 (kind = 2)` / `0x05 (kind = 1)`, and then requests `0x07 GET_BAL_RANGE` (`startBlock = 12062001, count = 16`) to heal to `#12,062,016`!

---

## 11. Integrator Guide (For Human Engineers & AI Coding Agents)

When implementing `ETHY/0` in any Ethereum client or library:

| Invariant | Requirement |
| --- | --- |
| **1. Canonical RLP Zero (`0x80`)** | Never encode integer `0` (`protocolVersion = 0`, `bucketIndex = 0`, `hasRight = 0`, `kind = 0`) as `Uint8Array([0])` (`0x00`). Always use unpadded bytes (`new Uint8Array(0)` → RLP `0x80`). |
| **2. Snappy Framing** | Like `eth` and `snap`, all `ETHY/0` message payloads after `HELLO` v5 are Snappy-compressed on the wire (`Peer._sendMessage` / `Peer._handleBody` in `@ethereumjs/devp2p` handles this automatically). |
| **3. Repeat `STATUS` & `v0.2` Compatibility** | Do **not** disconnect a peer that sends `ETHY/0 STATUS (0x00)` more than once, or a `v0.2` peer sending 9 elements (default `storageBucketsBitmask = 0`, `codeBucketsBitmask = 0`). Never set a `storageBucketsBitmask` or `codeBucketsBitmask` bit before 100% of that plane for bucket `b` is verified on disk. |
| **4. Verify Before Persist** | Never write an incoming `PROVEN_ACCOUNT_CHUNK` or `PROVEN_STORAGE_OR_CODE` payload to disk or `FlatStateManager` before cryptographic verification (`verifyMerkleRangeProof`, `computedRoot === storageRoot`, or `keccak256(code) === codeHash`) succeeds against the verified header chain. |
| **5. Preserve Proof Sidecars & Shared `code/`** | Keep `<nibble>.proof.idx` for account ranges and multi-chunk large contracts (`kind = 1`); small complete-trie contracts (`kind = 2`) need zero proof sidecar bytes. Persist all verified bytecodes in one cross-pivot `code/` store shared with `STATY/0`. |
| **6. Contiguous BAL Serving** | Persist verified `(headerRlp, balRlp)` bundles continuously past the 64-block RAM ring, never advertise a synthetic `balEarliestBlock` across a missing gap, and reject seeders with `balEarliestBlock > basePivotBlock + 1` unless another peer covers the bridge. |
| **7. Anti-Thundering-Herd Election** | Use the two-tier freshness horizon (`T_soft = 1,536`, `T_hard = 4,096`) and hash-lottery delay (§ 8.3) rather than a deterministic simultaneous cliff, and verify canonical header + liveness timeout before trusting `bucketsBitmask = 0` suppression. |

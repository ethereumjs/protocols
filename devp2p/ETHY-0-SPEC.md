# `ETHY/0` — Self-Proving Flat State & BAL Swarm Protocol (`devp2p`)

| Field | Value |
| --- | --- |
| **Protocol Identifier** | `ethy` |
| **Protocol Version** | `0` (`["ethy", 0]`) |
| **Transport Layer** | Ethereum DevP2P (`RLPx` v5 with Snappy framing) |
| **Message Space** | `0x00` – `0x0a` (`length = 11`) |
| **Discovery ENR Key** | `"ethy"` ([EIP-778](https://eips.ethereum.org/EIPS/eip-778) key `"ethy"`, value `rlp(0)` = `0x80`) |
| **Companion Protocols** | `eth/69–71` ([EIP-7642](https://eips.ethereum.org/EIPS/eip-7642), [EIP-7928](https://eips.ethereum.org/EIPS/eip-7928)), `snap/1`, Consensus Light Client (`@ethereumjs/beacon-light`) |
| **Status** | Draft Specification (`v0.1` — Amsterdam Era) |
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
8. [Worked Byte-Level Walkthrough](#8-worked-byte-level-walkthrough)
9. [Integrator Guide (For Human Engineers & AI Coding Agents)](#9-integrator-guide-for-human-engineers--ai-coding-agents)

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
1. **Bytecode (`kind = 0`):** For every returned bytecode blob, the receiver computes `keccak256(code)` and verifies it matches a requested `codeHash` referenced by a verified account's `codeHash` field.
2. **Storage Slots (`kind = 1`):** Verified via `verifyMerkleRangeProof` against the account's verified 32-byte `storageRoot` (when `proofNodes` is non-empty or complete-range `origin = 0x00..00, proofNodes = []` all-elements trie root check).

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

### 5.3 On-Disk Proof Sidecar Layout (`accounts/<nibble>.proof.idx`)
An `ETHY/0` implementation that ingests verified slabs into `accounts/<nibble>.bin` (or `.seg`) stores the accompanying Merkle boundary proofs in `accounts/<nibble>.proof.idx` as length-prefixed RLP records:
- Magic Header (8 bytes): `0x4554485950524631` (ASCII `"ETHY"` + `"PRF1"`).
- Each chunk entry is `[ uint32BE(recordByteLength), RLP.encode([ slabByteOffset, slabByteLength, originHash, lastHash, hasRight, chunkStateRoot, chunkBlockNumber, proofNodes ]) ]`.

---

## 6. Message Specification (`0x00` – `0x0a`)

| Code | Name | Direction | Description |
| --- | --- | --- | --- |
| `0x00` | `STATUS` | Peer ↔ Peer | Initial handshake + live updates advertising snapshot root, 16-bit `bucketsBitmask`, and servable BAL range. |
| `0x01` | `GET_BUCKET_MANIFEST` | Client → Server | Request the ordered list of self-proving chunk descriptors for account bucket `0x0..0xf`. |
| `0x02` | `BUCKET_MANIFEST` | Server → Client | Returns the chunk manifest for the requested bucket. |
| `0x03` | `GET_PROVEN_ACCOUNT_CHUNK` | Client → Server | Request one self-proving `BinaryAccount` slab + Merkle boundary proof by `(bucketIndex, originHash)`. |
| `0x04` | `PROVEN_ACCOUNT_CHUNK` | Server → Client | Returns the 136B-packed `binaryAccountsSlab`, `proofNodes`, `chunkStateRoot`, and `chunkBlockNumber`. |
| `0x05` | `GET_PROVEN_STORAGE_OR_CODE` | Client → Server | Request a batch of contract code blobs (`kind = 0`) or a proven storage range (`kind = 1`). |
| `0x06` | `PROVEN_STORAGE_OR_CODE` | Server → Client | Returns verified contract code blobs or storage slot range + boundary proof. |
| `0x07` | `GET_BAL_RANGE` | Client → Server | Request up to 32 contiguous `(blockNumber, headerRlp, balRlp)` bundles starting at `startBlock`. |
| `0x08` | `BAL_RANGE` | Server → Client | Returns contiguous `(blockNumber, headerRlp, balRlp)` bundles for post-pivot BAL healing / catch-up. |
| `0x09` | `GET_ETHY_PEERS` | Peer → Peer | Request up to `maxPeers` recently active `ETHY/0` peers (`ETHY_PEX`). |
| `0x0a` | `ETHY_PEERS` | Peer → Peer | Returns peer descriptors `[peerId, ip, tcpPort, udpPort, basePivotBlock, bucketsBitmask, ageSec]`. |

---

### `0x00` `STATUS`

Sent immediately upon mounting the `ETHY/0` subprotocol, and **re-sent whenever `bucketsBitmask` or `balLatestBlock` changes** (so connected peers learn immediately when a new bucket becomes servable or the BAL tail advances).

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
  latestBlockHash: bytes32        // 32-byte block hash at balLatestBlock (or 32 zero bytes)
]
```

**Validation Rules:**
- If `protocolVersion !== 0`, `chainId` mismatches local chain ID, or `genesisHash` mismatches local genesis hash, the receiver MUST disconnect with `DISCONNECT_REASON.SUBPROTOCOL_ERROR` (or `USELESS_PEER`).
- Unlike `eth/69` `STATUS` (which may only be sent once), **`ETHY/0 STATUS` MAY be sent multiple times during a session** to push updated `bucketsBitmask` and `balLatestBlock` values without polling!

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

Transfers either a batch of content-addressed contract code blobs (`kind = 0`) or a proven storage slot range for a contract (`kind = 1`).

**Request (`0x05 GET_PROVEN_STORAGE_OR_CODE`):**
- When `kind = 0` (Bytecode Batch, up to 64 `codeHashes`):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 0 (Bytecode batch)
    codeHashes: [bytes32, ...],
    maxResponseBytes: uint64
  ]
  ```
- When `kind = 1` (Storage Slot Range for one `accountHash`):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 1 (Storage)
    accountHash: bytes32,
    storageRoot: bytes32,
    originSlotHash: bytes32,
    maxResponseBytes: uint64
  ]
  ```

**Response (`0x06 PROVEN_STORAGE_OR_CODE`):**
- When `kind = 0` (Bytecode Batch):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 0
    codes: [bytes, ...]           // Receiver verifies keccak256(code) === requestedCodeHash
  ]
  ```
- When `kind = 1` (Storage Slot Range):
  ```text
  [
    requestId: uint64,
    kind: uint8,                  // 1
    accountHash: bytes32,
    originSlotHash: bytes32,
    lastSlotHash: bytes32,
    hasRight: uint8,
    slots: [[slotHash: bytes32, slotValue: bytes], ...],
    proofNodes: [bytes, ...]      // Empty [] when complete storage trie is returned from origin 0x00..00
  ]
  ```

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
1. $P.\text{bucketsBitmask} \neq 0$ (it holds at least one complete, proven account bucket), AND
2. Either:
   - $P$'s `basePivotBlock` (`#C`) is within the `eth/71` / `ETHY/0` BAL retention window of the connected peer set (`balEarliestBlock <= #C + 1` and `#Head - #C <= MAX_ETHY_BAL_TAIL_BLOCKS`, default `14,400` blocks $\approx 48$ hours), OR
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
         Wait for usable        Wait up to 2,500ms   Anchor live #P_0
         ETHY/0 peer &          after first ETH      immediately on
         sync via ETHY/0        STATUS for usable    SNAP/1 (while
         exclusively            ETHY/0 peer          seeding ETHY/0!)
                                   │          │
                        Found! ◄───┘          └───► Timeout (0 ETHY peers)
                          │                                  │
                          ▼                                  ▼
                Lock transport = 'ethy'            Lock transport = 'snap'
                Sync proven chunks +               Sync via SNAP/1 + BALs
                BAL_RANGE (#C+1..#H)               & seed finished buckets
                                                   on ETHY/0 as they finish!
```

1. **Immediate Selection on Direct Connect (`--connect` / `peers.json` VIP):**
   - If an `ETHY/0` peer connects and sends a usable `ETHY/0 STATUS` before the `2,500ms` grace window expires, the node immediately locks `syncTransport = 'ethy'` (zero extra wait).
2. **Bounded `2,500ms` Grace Window in `auto` Mode:**
   - When running in default `auto` mode on a public network where only Geth/Reth `snap/1` peers are initially found, the node waits at most **`2,500ms`** after its first `ETH STATUS` for an `ETHY/0` peer (via `peers.json` or `ETHY_PEX`). If none connects within `2.5s`, it immediately starts `SNAP/1` sync with no noticeable delay.
3. **Bucket-Granular Seeding Regardless of Sync Mode:**
   - Even when a node syncs via `SNAP/1` (`syncTransport = 'snap'`), it writes `accounts/<nibble>.proof.idx` for every verified slab and flips bit `1 << b` in its outbound `ETHY/0 STATUS` `bucketsBitmask` as each bucket `0x0..0xf` completes — immediately becoming an `ETHY/0` seeder for subsequent nodes!

---

## 8. Worked Byte-Level Walkthrough

Suppose **Node A (Seeder)** has completed bucket `0x0` at `basePivotBlock = 12,062,000` (`0xb80d70`) with `bucketsBitmask = 0x0001` (bucket `0x0` ready) and BAL tail `12,062,000 .. 12,062,016`:

1. **Node A sends `0x00 STATUS`:**
   ```ts
   [
     new Uint8Array(0),                      // protocolVersion = 0 (canonical 0x80)
     bigIntToUnpaddedBytes(11155111n),       // chainId = Sepolia (0xaa36a7)
     genesisHash32,                          // 32 bytes
     bigIntToUnpaddedBytes(12062000n),       // basePivotBlock = 12,062,000
     basePivotStateRoot32,                   // 32 bytes
     intToUnpaddedBytes(0x0001),             // bucketsBitmask = 0x01 (bucket 0x0 ready)
     bigIntToUnpaddedBytes(12062000n),       // balEarliestBlock
     bigIntToUnpaddedBytes(12062016n),       // balLatestBlock
     latestBlockHash32,                      // 32 bytes
   ]
   ```
2. **Node B (Joiner) requests the manifest for bucket `0x0` (`0x01 GET_BUCKET_MANIFEST`):**
   ```ts
   [
     bigIntToUnpaddedBytes(1n),              // requestId = 1
     new Uint8Array(0),                      // bucketIndex = 0 (canonical 0x80)
   ]
   ```
3. **Node A replies with `0x02 BUCKET_MANIFEST`** listing the chunks in `0.proof.idx`, and **Node B requests each chunk via `0x03 GET_PROVEN_ACCOUNT_CHUNK`**, verifies `verifyMerkleRangeProof` in its worker pool, and then requests `0x07 GET_BAL_RANGE` (`startBlock = 12062001, count = 16`) to heal to `#12,062,016`!

---

## 9. Integrator Guide (For Human Engineers & AI Coding Agents)

When implementing `ETHY/0` in any Ethereum client or library:

| Invariant | Requirement |
| --- | --- |
| **1. Canonical RLP Zero (`0x80`)** | Never encode integer `0` (`protocolVersion = 0`, `bucketIndex = 0`, `hasRight = 0`, `kind = 0`) as `Uint8Array([0])` (`0x00`). Always use unpadded bytes (`new Uint8Array(0)` → RLP `0x80`). |
| **2. Snappy Framing** | Like `eth` and `snap`, all `ETHY/0` message payloads after `HELLO` v5 are Snappy-compressed on the wire (`Peer._sendMessage` / `Peer._handleBody` in `@ethereumjs/devp2p` handles this automatically). |
| **3. Repeat `STATUS` Allowed** | Do **not** disconnect a peer that sends `ETHY/0 STATUS (0x00)` more than once! Subsequent `STATUS` messages are live capability updates announcing newly completed buckets in `bucketsBitmask` or a new `balLatestBlock`. |
| **4. Verify Before Persist** | Never write an incoming `PROVEN_ACCOUNT_CHUNK` to disk or `FlatStateManager` before `verifyMerkleRangeProof` succeeds against a `chunkStateRoot` that has been linked to the verified header chain. |
| **5. Preserve Proof Sidecars** | When syncing via `SNAP/1`, pass the raw `proof: Uint8Array[]` nodes from `AccountRange` through the verifier worker so `SnapPivotStore` can append them to `<nibble>.proof.idx` for `ETHY/0` seeding. |

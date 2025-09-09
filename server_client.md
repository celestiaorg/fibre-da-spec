# FibreDA — Full Specification v1 (client & server)

---

## 0) Glossary

* **FSP**: Fibre Service Provider — a validator‑operated server storing its assigned rows.
* **Default FSP** (DFSP): a preferred validator/FSP endpoint the client uses for unary interactions that do not require quorum:
  * Escrow balance queries / proofs 
  * SubmitPayForFibre (relay of MsgPayForFibre)
* **Row**: a fixed **index** in the codeword; total rows per blob are **N = K + parity = 4K = 16384**. Each row is a fixed‑length chunk; **row size is variable** per blob but must be a **multiple of 64 bytes**.
* **Commitment**: `SHA256(rowRoot || rlcRoot)`.
* **RLC / rlc\_orig**: GF(2¹²⁸) vector used for rs-ema1d encoding correctness verification; length **N** (16 bytes each ⇒ **\~256 KiB** total).
* **PaymentPromise (PP)**: off‑chain, escrow‑funded **payment authorization** signed by the escrow owner (per `x/fibre`). Carries `{owner, namespace(v2), blob_size, commitment, row_version, creation_height, signature}`.
* **Validator signatures**: ed25519 signatures by validators over the **commitment** (used in `MsgPayForFibre`).
* **Attestation / Receipt** (off‑chain): optional server‑signed receipt (domain‑separated), including **expiry** (TTL) for UX; not used on‑chain.
* **Assignment**: deterministic **permutation‑based** mapping from `(commitment, valset@height)` to **non‑overlapping** row indices per validator.

---

## 1) Goals & scope of v1

**Goals**

1. Encode blob into **K = 4096** originals, parity **3K**, **N = 16384**, shard via Assignment, collect ≥ 2/3 validator signatures by **count and voting power**.
2. Servers accept data **only** if `PaymentPromise` validates; they **sign** the attestation preimage above and store rows.
3. Provide **Bulk** upload (default) and **Streaming** upload for early rejection & backpressure.
4. **Library‑first** client; optional **light‑node** later.

**Improvements (after v1)**
* Shipping SQL or epoch‑bucket variants; v1 is **Badger‑only** (evaluate based on performance later).
* Allow **streaming** where per‑FSP payloads are large (§6) to allow fast return from server.

---

## 2) Parameters & reasoning

> **Encoding model**: **K = 2¹² = 4096** original rows, **encoding factor 1:3** ⇒ parity **3K = 12288**, so **N = 16384** rows per blob. **Row size** is chosen per blob (multiple of 64 B) and capped to keep **max blob size = 128 MiB**.

| Parameter           |            Default / Formula | Derived / Notes                                                                                      |
| ------------------- | ---------------------------: | ---------------------------------------------------------------------------------------------------- |
| `original_rows (K)` |                     **4096** | Fixed **2¹²** originals.                                                                             |
| `encoding_factor`   |                      **1:3** | Parity is **3×** originals ⇒ **parity\_rows = 12288**.                                               |
| `total_rows (N)`    |                    **16384** | `N = 4K`.                                                                                            |
| `row_size`          |  **auto**; **64 B × n**, n∈ℕ | **Multiple of 64 B**. Chosen per blob: `row_size = ceil(original_length / K)` then round up to 64 B. |
| `row_size_min`      |                     **64 B** | Minimum legal row size.                                                                              |
| `row_size_max`      |                   **32 KiB** | To cap `max_blob_size` at **128 MiB**: `K * row_size_max = 4096 * 32 KiB`.                           |
| `max_blob_size`     |                  **128 MiB** | `K * row_size_max`.                                                                                  |
| `min_capacity`      |                  **256 KiB** | `K * row_size_min = 4096 * 64 B`. Small blobs are zero-padded to row bounds.                         |
| `val_set_size`      |                      **100** | Planning default; runtime uses actual (from `valset_height`).                                        |
| `rows_per_val`      | `≈ ceil(N / V)` (even split) | With V=100 ⇒ **163 or 164** rows/validator (first `r=84` get 164; others 163).                       |
| `rlc_orig_size`     |                  **256 KiB** | `16 * N = 16 * 16384` bytes of GF128 coeffs.                                                         |
| `retention_ttl`                 |                 24 h | Single horizon; **promotion disabled**. **Refresh** can extend (§5.3).                  |         |                               |
| **Throughput**                  |           \~20 MiB/s | Configurable.                                                                           |         |                               |

**Server retention & limits**

* `retention_ttl = 24h`.
* `server_rows_per_message_limit ≈ ceil(N / val_set_size) + 1` (guard rail).
* **Throughput**: `val_throughput ≈ 20 MiB/s` (configurable).

    * Per-FSP **request size ≈ rows\_assigned × row\_size**.
      Examples (V=100, \~163–164 rows/FSP):

        * `blob_size = 8 MiB`  → **\~1.3 MiB** per request ⇒ \~14–16 RPS at 20 MiB/s.
        * `blob_size = 32 MiB` → **\~5.2 MiB** per request ⇒ \~3–4 RPS at 20 MiB/s.

**Client concurrency**: `send_workers=20`, `read_workers=20`.

---

## 3) Design decisions (summary)

* **Payment‑gated upload.** Server calls `QueryValidatePaymentPromise(PP)`. If valid & sufficient funds (or provably insufficient—see §5.4), proceed.
* **Unified attestation/signature.** FSP returns a single **validator signature** over:

  ```
  preimage = "FIBRE/v1" || commitment || chain_id || be64(posted_block_height)
  with posted_block_height := PP.creation_height (v1.2)
  ```

  Client aggregates ≥2/3 and submits `MsgPayForFibre{promise, validator_signatures}`.
* **Refresh (no re‑upload).** New PP for the **same commitment**, collect signatures again, submit `MsgPayForFibre`.
* **Assignment.** “Permute validators” is **OPTIONAL** to improve randomness (unclear if required).
* **Streaming rationale.** Server can **reject early** after validating **commitment** (dedup/existence) and **PaymentPromise** **before** reading large row payloads (§6.2).

---

## 4) Client architecture

### 4.1 Construction & config

```go
// Light-node backed (headers/valsets available)
NewClientWithLightNode(cfg LightNodeConfig, vtMode ValTrackerMode, opts ...Option) (*Client, error)

// Library-first (preferred v1)
NewClientWithLibrary(cfg LibraryConfig, vtMode ValTrackerMode, opts ...Option) (*Client, error)
```

**Key config fields**

* `ChainID string` // used in validator signature preimage
* `EscrowOwner string` // bech32
* `AutoFund int` // optional; fund escrow on insufficient balance with policy. Client auto funds escrow via `MsgDepositToEscrow` if needed with this amount

**Options**
* `WithSendWorkers(int)`, `WithReadWorkers(int)` // concurrency settings

### 4.2 Balance bootstrap and reconciliation 

On initialization the client **must**:

1. Query via DFSP **EscrowAccount( EscrowOwner )** and cache `available_balance@height₀`.
   2. If DFSP is down, query any FSP.
2. If Escrow does not exist, client **must** create it via `MsgCreateEscrow` (with optional initial deposit) and wait for inclusion + state proof (see §5.5).
3. If AutoFund is enabled, client may create with `initial_deposit = AutoFund` to avoid immediate top-up.

On each **Put()** or **Refresh()**:
* **Verify server feedback** during upload, DFSP (or the rejecting FSP) must return `InsufficientBalanceProof` (see §5.4). Upon such proof:

    * Update local cached balance to `available_balance@proof_height`.
    * Track **pending promises** returned by server (those not settled yet).
    * If **AutoFund** is enabled, **fund (deposit)** the escrow to cover shortfall, verify tx inclusion + state proof (see §5.5), then retry upload.

### 4.3 Public API (client)

```go
type Namespace struct {
  Version uint8  // MUST be 2
  Bytes   [29]byte
}

type PutResult struct {
  Commitment [32]byte
  ValidatorSignatures [][]byte // ed25519 over preimage (see §7)
  CreationHeight uint64        // == PP.creation_height
  TTL *time.Time               // optional (server-provided info)
}

type PFFConfirmation struct {
  TxHash  string   // tx hash of MsgPayForFibre
  Height  uint64   // inclusion height
}

type FundingTxResult struct {
  TxHash       string
  InclusionHeight uint64
  InclusionProof []byte   // chain inclusion proof (e.g., tx-proof)
  NewAvailableBalance string // coin string
  BalanceProofHeight uint64
  BalanceStateProof  []byte  // app-specific proof
}

type Client interface {
  // Construct PP internally, upload, aggregate sigs, and submit MsgPayForFibre.
  Put(ctx context.Context, data []byte, ns Namespace) (PutResult, PFFConfirmation, error)

  // Same as Put but returns signatures without submitting on-chain.
  PutWithoutPFF(ctx context.Context, data []byte, ns Namespace) (PutResult, error)

  // Provide creation height if known; if 0, client will fetch from FSP metadata.
  Get(ctx context.Context, commitment [32]byte, creationHeight uint64, originalLength uint64) ([]byte, error)

  // Refresh retention for the same commitment (no row upload).
  Refresh(ctx context.Context, commitment [32]byte, ns Namespace, creationHeight uint64) (PutResult, error)

  // ---- Funding API (manual) ----
  CreateEscrow(ctx context.Context, initialDeposit string) (FundingTxResult, error)
  DepositToEscrow(ctx context.Context, amount string) (FundingTxResult, error)
  RequestWithdrawal(ctx context.Context, amount string) (FundingTxResult, error)
  ClaimWithdrawal(ctx context.Context, requestedAt uint64) (FundingTxResult, error)
}
```

---

## 5) Client flows

### 5.1 Put()

1. **Bounds:** `0 < len(data) ≤ 128 MiB`.
2. **Valset & height:** `vals, height := vt.CurrentSet(ctx)`. Set `creation_height = height`.
3. **Encode:** choose `row_size` (×64 B), compute `commitment`, `rlc_orig`, and produce `rows[]`, `proofs[]` (K=4096, N=16384).
4. **Construct & sign PaymentPromise (internally):** `{owner, ns(v2), blob_size=originalLength, commitment, row_version=1, creation_height}` + owner signature.
5. **Assignment:** `assign := Assignment(commitment, valset@creation_height)`.
6. **Upload (parallel per FSP, ≤ send\_workers):** send `UploadRowsRequest{chain_id, posted_block_height=creation_height, commitment, rlc_orig, rows subset+proofs, promise}`.
7. **FSP validates** PP (escrow exists, sig valid, sufficient balance or returns **InsufficientBalanceProof**), verifies RLC/proofs & assignment, persists, and returns **validator\_signature** over the preimage (see §7).
8. **Aggregate** signatures; verify ≥2/3 by count and power (valset\@creation\_height).
9. **Pay:**

    * **Put**: client submits `MsgPayForFibre{promise, validator_signatures}` via DFSP; return `PutResult + PFFConfirmation`.
      * If DFSP fails (timeout, refusal), fallback to other FSPs in parallel until success or all fail.
    * **PutWithoutPFF**: return `PutResult` (caller submits later).

### 5.2 Get()

1. If `creation_height==0`, fetch it from any FSP’s metadata for this commitment.
2. Recompute `Assignment(commitment, valset@creation_height)`.
3. Fetch assigned rows from each FSP in parallel (`read_workers`).
4. Verify Merkle proofs; optional: enforce indices match assignment slice.
5. Decode once enough rows for `originalLength` are gathered; recompute & verify **RLC**.

### 5.3 Refresh (no re‑upload)

1. Make a **new PP** for the **same commitment** with a **new creation\_height**; `posted_block_height := new creation_height`.
2. Ask assigned FSPs for a **new validator signature** (no rows).
3. Submit **`MsgPayForFibre{PP, new_signatures}`** to extend retention TTL.
4. FSPs may extend TTL immediately upon verifying the PP, or upon observing the chain PayForFibre.

### 5.4 Insufficient funds — proofs & reconciliation

* If an FSP detects **insufficient available balance**, it **must** return **proof** instead of silently failing.
* The client, upon receiving the proof, **updates** its cached balance/pending view and, if **AutoFund**, **deposits** to escrow before retrying.

**Insufficient balance proof (server → client)**

* **Semantics:** “With chain state at `proof_height`, `available_balance` is X; considering these **pending promises** (not yet processed), your effective availability is insufficient for the requested PP.”
* **Contents (v1, baseline; more efficient proving TBD based pn payment spec):**

    * `owner`, `chain_id`, `proof_height`.
    * `available_balance` **+** `escrow_state_proof` (ICS23 or app‑specific state proof).
    * `pending_promises[]`: the `PaymentPromise`s **from this owner** that the FSP signed and which are **not settled** (as observed by the FSP).
      *Note:* absence proofs for `ProcessedPromise` entries may be costly; v1.2 treats this as **advisory** with best‑effort evidence. Efficient indexing/proving is **TBD**.

### 5.5 Funding API (manual) — return proofs

Each funding call **must** return:

* `TxHash`, `InclusionHeight`, `InclusionProof`
* `NewAvailableBalance`, `BalanceProofHeight`, `BalanceStateProof`

Operations:

* `CreateEscrow(initialDeposit)` → creates escrow + optional deposit.
* `DepositToEscrow(amount)` → increases balance.
* `RequestWithdrawal(amount)` → reserves amount; reduces `available_balance` immediately.
* `ProcessWithdrawal(requestedAt)` → completes withdrawal after delay.

---

## 6) Wire APIs (protobuf)

### 6.1 Bulk API

```proto
syntax = "proto3";
package fibre.v1;

message GF128 { bytes coeffs = 1; }                 // len == 16
message Commitment { bytes value = 1; }             // len == 32
message RowWithProof { uint32 index = 1; bytes row = 2; bytes proof = 3; }
message RlcOrig { repeated GF128 coeffs = 1; }      // len == N = 16384

// Copy reference x/fibre.PaymentPromise
message PaymentPromise {
  string owner = 1;        // bech32
  bytes namespace = 2;     // MUST be version 2; 29 bytes raw
  uint64 blob_size = 3;    // original length (pre-padding)
  bytes commitment = 4;    // 32 bytes
  uint32 row_version = 5;  // e.g., 1
  int64 creation_height = 6;
  bytes signature = 7;     // owner signature over PP sign-bytes
}

// Proof delivered when rejecting due to insufficient balance
message InsufficientBalanceProof {
  string owner = 1;
  string chain_id = 2;
  uint64 proof_height = 3;

  string available_balance = 4;   // coin string
  bytes  escrow_state_proof = 5;  // ICS23 or app-specific proof bytes

  repeated PaymentPromise pending_promises = 6; // best-effort list (TBD: more efficient proving/indexing)
}

message UploadRowsRequest {
  string chain_id = 1;                    // for validator signature preimage
  uint64 posted_block_height = 2;         // MUST equal promise.creation_height in v1.2

  Commitment commitment = 3;
  RlcOrig rlc_orig = 4;
  repeated RowWithProof rows = 5;         // this FSP's assigned subset only
  PaymentPromise promise = 6;             // MUST be included
  optional uint64 original_length = 10;   // optional sanity for padding checks
}

// Success or failure (insufficient funds). Servers SHOULD use gRPC error codes;
// this message is provided for explicit success/failure channels in non-gRPC deployments.
message UploadRowsResponse {
  // Success path:
  bytes validator_signature = 1;    // ed25519 over sha256("FIBRE/v1"||commitment||chain_id||be64(posted_block_height))
  optional uint64 ttl_epoch_minute = 2;
  optional uint32 backoff_ms = 10;

  // Failure path (insufficient balance):
  bool insufficient_balance = 20;
  InsufficientBalanceProof insufficient_balance_proof = 21;
}

message GetRowsRequest {
  Commitment commitment = 1;
}

message GetRowsResponse {
  repeated RowWithProof rows = 1;    // server returns all rows it holds for the commitment
  optional uint64 ttl_epoch_minute = 2;
  optional uint32 backoff_ms = 10;
}

service Fibre {
  rpc UploadRows(UploadRowsRequest) returns (UploadRowsResponse);
  rpc GetRows(GetRowsRequest) returns (GetRowsResponse);
}
```

### 6.2 Streaming API (early rejection rationale)

**Why streaming?** After the **`Init`** frame, the server can:

* **Validate** `commitment` (e.g., dedup/existing).
* **Validate** `PaymentPromise` (escrow, signature, already processed, balance).
* **Reject early** (returning InsufficientBalanceProof if applicable) **before** reading heavy `Chunk` payloads.

```proto
syntax = "proto3";
package fibre.v1;

message Init {
  string chain_id = 1;
  uint64 posted_block_height = 2;         // MUST equal promise.creation_height in v1.2
  bytes commitment = 3;                    // 32 bytes
  uint64 total_chunks = 4;
  uint64 total_bytes = 5;
  PaymentPromise promise = 6;
}

message Chunk  { uint32 index = 1; bytes data = 2; } // ≤ row_size per chunk
message Finish { bytes sha256 = 1; }

message UploadReq { oneof kind { Init init = 1; Chunk chunk = 2; Finish end = 3; } }

message UploadResp {
  // Success:
  bytes validator_signature = 1;
  optional uint64 ttl_epoch_minute = 2;

  // Failure:
  bool insufficient_balance = 20;
  InsufficientBalanceProof insufficient_balance_proof = 21;
}

service FibreStreaming { rpc Upload(stream UploadReq) returns (UploadResp); }
```

### 6.3 Refresh API (optional)

```proto
syntax = "proto3";
package fibre.v1;

message RefreshRequest {
  string chain_id = 1;
  uint64 posted_block_height = 2;       // == new PP.creation_height (v1.2)

  Commitment commitment = 3;
  PaymentPromise promise = 4;           // new promise for same commitment
}

message RefreshResponse {
  bytes validator_signature = 1;        // over the same preimage scheme
  optional uint64 ttl_epoch_minute = 2;
}

service FibreRefresh {
  rpc Refresh(RefreshRequest) returns (RefreshResponse);
}
```

---

## 7) Signatures & verification 

**Preimage for validator signatures (and server attestation):**

```
digest = sha256(
  "FIBRE/v1" ||
  commitment (32B) ||
  chain_id (utf-8) ||
  be64(posted_block_height)   // v1: posted_block_height := PP.creation_height
)
sig = ed25519_sign(validator_consensus_key_at(PP.creation_height), digest)
```

**On‑chain verification (`MsgPayForFibre`)**

* Valset used for power tally: **`PP.creation_height`**.
* Each signature is verified against the above preimage with `posted_block_height = PP.creation_height` and the tx’s `chain_id`.
* ≥ 2/3 voting power required.


---

## 8) Assignment (non‑overlapping; permutation‑based)

Inputs: `seed = commitment`, `n = 16384`, validators `k = |valset@PP.creation_height|`.

1. **Permute shares**: sort `i` by `SHA256("share" || seed || u32be(i))`.
2. **Index validators** lexicographically by `PubKey` → indices `j=0..k-1`.
3. **(OPTIONAL) Permute validators**: sort `j` by `SHA256("validator" || seed || u32be(j))`.
4. `base = n // k`, `r = n % k`; first `r` in chosen validator order get `base+1`, others `base`.
5. Walk permuted shares handing contiguous blocks to validators in that order.

---

## 9) Server architecture

**Components**

* Store: Badger (v1)
* gRPC: Bulk + (+ optional Streaming) (+ optional Refresh)
* PaymentPromise validator: `QueryValidatePaymentPromise(PP)`
* ValSet service: `valset@PP.creation_height`
* Rate limiter & fairness

**Badger keys**

* `d/<commitment>` → assigned rows blob (+ metadata: `row_size`, `original_length`, `creation_height`, `namespace`, `row_version`)
* `b/<YYYYMMDDHHmm>/<commitment>` → retention bucket

**Upload pipeline**

1. Validate PP: `valid && sufficient_balance && !already_processed`.

    * If **insufficient**, return **InsufficientBalanceProof**.
2. Build RLC from `rlc_orig`; verify row proofs against `commitment`.
3. Recompute Assignment; enforce indices match this validator’s slice.
4. Persist rows with `expiry = now + retention_ttl`.
5. Sign and return `validator_signature` (preimage in §7) + TTL hint if desired.
6. Include `backoff_ms` under pressure.

**Get pathway**

* Return all stored rows for this validator’s slice; include TTL if desired.

---

## 10) Observability

**Client metrics:** encode latency, chosen `row_size`, per‑FSP upload latency, signatures collected, quorum time, PayForFibre submit/inclusion, balance cache age, AutoFund events, reconciliation updates, insufficient‑proofs processed.

**Server metrics:** RPS, bytes/s, assignment mismatches, proof/RLC failures, write latency, GC, validator\_sig count, `backoff_ms`, PP validation latency, insufficient‑proofs emitted.

---

## 11) Testing plan

**Unit:** assignment determinism (with/without validator permutation), RLC, row\_size rounding, **signature preimage correctness**, PP sign‑bytes, insufficient‑proof encoding.

**Integration:** end‑to‑end Put/Get/Refresh, PP validation happy/failure, balance reconciliation + AutoFund, streaming early‑reject, failure injection (missing rows, wrong proofs), quorum aggregation and on‑chain `MsgPayForFibre`.

**Load:** ≥60 MiB/s writes with realistic `row_size` (8–32 KiB); verify backpressure behavior; bounded DB via GC.

---

## 12) Suggested defaults

```toml
[row]
original_count = 4096
encoding_factor = "1:3"
total_count = 16384
min_size = 64
max_size = 32768

[client]
send_workers = 20
read_workers = 20
mode = "library"
valtracker = "lazy"
auto_pay = true
auto_fund = false     # opt-in

[server]
retention_ttl = "24h"
rows_per_message_limit = 165
throughput_cap_bytes_per_sec = 10485760
```


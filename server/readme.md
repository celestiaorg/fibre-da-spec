# FibreDA — Full Specification v1.2 (client & server)

> This revision updates parameters so that **original rows (K)** are **2¹² = 4096** (not total). All derived values and examples are adjusted accordingly, while preserving the prior v1.1 design changes (Pre-Payment, no promotion, permutation-based non-overlapping assignment).

---

## 0) Glossary

* **FSP**: Fibre Service Provider — a validator-operated server storing its assigned rows.
* **Row**: a fixed **index** in the codeword; total rows per blob are **N = K + parity = 4K = 16384**. Each row is a fixed-length chunk for a given blob; **row size is variable** per blob but must be a **multiple of 64 bytes**.
* **Commitment**: `SHA256(rowRoot || rlcRoot)`.
* **RLC**/**rlc\_orig**: GF(2¹²⁸) vector used for rsema1d **Context-Based Verification**.
* **Pre-Payment (PrePay)**: **on-chain pre-payment transaction** that the client submits **first**; servers verify **inclusion** (by tx hash + inclusion height) and only then accept data.
* **Attestation**: validator signature binding acceptance (with prepay reference) to the commitment and retention horizon.
* **Assignment** (formerly “ShardMap”): deterministic **permutation-based** mapping from (commitment, **valset\@height**, validator) → **non-overlapping** set of row indices.

---

## 1) Goals & non-goals

**Goals**

1. Encode a blob into **K = 4096 original rows** with **encoding factor 1:3** (3K parity; **N = 16384** total rows), **shard** (non-overlapping) across FSPs via permutation assignment, and collect ≥ 2/3 **attestations** by count and voting power.
2. **Require Pre-Payment** on-chain **before upload**. Client supplies `(prepay_tx_hash, prepay_block_height)` and a `valset_height`. Servers **verify inclusion** and accept only if paid.
3. Provide **bulk** upload path for v1 (simple), with **streaming** available if per-FSP payloads are large or proxies enforce small limits (§6).
4. Keep v1 client **library-first**; support a **light-node** mode later.

**Non-goals (v1)**

* Shipping a SQL or custom epoch-bucket store; v1 is **Badger-only**. We will test SQL-light and epoch-bucket variants later and choose by measured performance.

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

**Server retention & limits**

* `retention_ttl = 24h` (single horizon; **no promotion**).
* `server_rows_per_message_limit ≈ ceil(N / val_set_size) + 1` (guard rail).
* **Throughput**: `val_throughput ≈ 20 MiB/s` (configurable).

    * Per-FSP **request size ≈ rows\_assigned × row\_size**.
      Examples (V=100, \~163–164 rows/FSP):

        * `blob_size = 8 MiB`  → **\~1.3 MiB** per request ⇒ \~14–16 RPS at 20 MiB/s.
        * `blob_size = 32 MiB` → **\~5.2 MiB** per request ⇒ \~3–4 RPS at 20 MiB/s.

**Client concurrency**: `send_workers=20`, `read_workers=20`.

---

## 3) Design decisions (summary)

* **Bulk vs Streaming**: **Bulk** remains v1 default; **Streaming** is recommended when `rows_per_val × row_size` is large or middleboxes enforce small frames. (§6)
* **Library-first** in v1 to simplify testing/ops; **Light-node** mode later. (§4)
* **ValTracker** fetches **valset at a specific height** to anchor assignment. (§4.2)
* **Concurrency**: run **rsema1d** and ValTracker **in parallel**. (§4.3)
* **Server verification**: pipeline **Pre-Payment inclusion check → row proof → RLC context → assignment check → persist → sign**. (§8.3)
* **No promotion**: storage uses a single **retention TTL**; **Refresh** resubmits **Pre-Payment** to extend. (§5.3)
* **Backpressure**: retain conservative **20 MiB/s** cap; actual RPS depends on `row_size`. (§8.5)

---

## 4) Client architecture

### 4.1 Construction & config

```go
// Light-node backed (headers/valsets available; best for Active mode):
NewClientWithLightNode(cfg LightNodeConfig, vtMode ValTrackerMode, opts ...Option) (*Client, error)

// Library-first (preferred v1; no local sync/storage; ValTracker defaults to Lazy):
NewClientWithLibrary(cfg LibraryConfig, vtMode ValTrackerMode, opts ...Option) (*Client, error)
```

**Options**:
`WithSendWorkers(int)`, `WithReadWorkers(int)`, `WithAssignment(Assignment)`,
`WithPrepaySubmitter(PrepaySubmitter)`, `WithDialer(Dialer)`.

### 4.2 ValTracker (Active vs Lazy)

* **Inputs**: `HeaderService` / `ValSetService` to fetch **validator set at a given height**; **Provider Records Server** for FSP endpoints.
* **Active Mode**: subscribe to chain heads; cache valsets; refresh endpoints/pools on changes.
* **Lazy Mode**: resolve on demand (`Put/Get/Refresh`).

```go
type ValInfo struct { PubKey []byte; VotingPow uint64; Address string }

type ValTracker interface {
  ActiveSetAt(ctx context.Context, height uint64) ([]ValInfo, error)
  CurrentSet(ctx context.Context) ([]ValInfo, height uint64, error)
}
```

### 4.3 Public API

```go
type PrepayRef struct {
  TxHash [32]byte
  BlockHeight uint64 // inclusion height
}

type Client interface {
  Put(ctx context.Context, data []byte, prepay PrepayRef) (commitment [32]byte, err error)
  Get(ctx context.Context, commitment [32]byte, originalLength uint64) ([]byte, error)
  Refresh(ctx context.Context, commitment [32]byte, prepay PrepayRef) (err error) // Pre-Payment resubmission only
}
```

### 4.5 Assignment (non-overlapping; permutation-based)

**Inputs**:
`seed = commitment`, total shares `n = N = 16384`, validators `k = |valset@height|`.

**Algorithm**

1. **Permute shares**: sort share indices `i` by `SHA256("share" || seed || u32be(i))`.
2. **Index validators** by public key (e.g., lexicographic `PubKey` bytes), giving indices `j = 0..k-1`.
3. **Permute validators**: sort validator indices `j` by `SHA256("validator" || seed || u32be(j))`.
4. `base = n // k`, `r = n % k`.
   First `r` validators in the permuted order get `base+1` shares; the rest get `base`.
5. Walk the permuted share list in order, handing out contiguous blocks to validators in the permuted order.

**Properties**: **Non-overlapping**, **even** split, reproducible from `(commitment, valset@height)`.

---

## 5) Client flows

### 5.1 Put()

0. Bounds: `0 < len(data) ≤ 128 MiB`.
1. **Pre-Payment**: submit and await inclusion; obtain `(prepay_tx_hash, prepay_block_height)`.
2. **Valset**: `vals, vals_height := vt.CurrentSet(ctx)`.
3. **Encode** (parallel with 2): rsema1d chooses `row_size` (64 B multiple), computes `commitment`, `rlc_orig`, and produces `rows[]`, `proofs[]`. (**K=4096, N=16384**).
4. **Plan**: `assign := Assignment(commitment, vals)` as per §4.5.
5. **Upload bulk** (parallel per FSP; ≤ `send_workers`):
   `UploadRowsRequest{commitment, rlc_orig, rows subset+proofs, valset_height, prepay_tx_hash, prepay_block_height, original_length?}`.
6. Collect **attestations**; verify ed25519 signatures over the receipt preimage (§7); aggregate by **count** and **voting power**. Wait until 2/3 quorum on both voting power and count.
7. If quorum not reached (e.g., too few attestations), return error along with prepay reference for possible retry.
8. Calculate `ttl = prepay_block_height.timestamp + retention_ttl`.
9. Return `commitment, ttl` or error.

### 5.2 Get()

1. Obtain valset at the original `valsetHeight`.
2. **Do we need to verify assignment?** Recompute **Assignment** (who holds which rows).
3. Fetch from each assigned FSP in parallel (`read_workers`); 
4. Verify each row’s **Merkle proof**; (optional per 2): ensure row indices match this FSP’s assignment slice.
5. Reconstruct with rsema1d once enough rows are gathered to reach `originalLength`.
6. Reconstruct **RLC**, verify encoding.

### 5.3 Refresh() — **Pre-Payment resubmission**

Optional; not required for v1.
* Input: `commitment, PrepayRef` (no data).
* Action: submit **new Pre-Payment** to extend retention TTL.
* Output: `ttl` (new horizon) or error.

### 5.4 Row size selection (informative)

```text
Given len(data) and K=4096:
row_size_raw = ceil(len(data) / K)
row_size     = round_up_to_multiple(row_size_raw, 64B)
row_size     = min(row_size, 32 KiB)             // cap to keep max_blob_size at 128 MiB
Pad data to K * row_size and encode to N=16384 rows.
```

### 5.5 Assignment details (numbers)

For `V = |valset|`, `n = N = 16384`:

```
base = n // V
r    = n % V
rows_per_val ∈ { base, base+1 }

Example: V=100 -> base=163, r=84
-> 84 validators get 164 rows, the remaining 16 get 163 rows.
```

---

## 6) Wire APIs (protobuf)

### 6.1 Bulk API (v1 default)

```proto
syntax = "proto3"; package fibre.v1;

message GF128 { bytes coeffs = 1; }          // len == 16
message Commitment { bytes value = 1; }      // len == 32
message RowWithProof { uint32 index = 1; bytes row = 2; bytes proof = 3; } // row len == row_size (per blob)
message RlcOrig { repeated GF128 coeffs = 1; }

message PrepayRef {
  bytes tx_hash = 1;        // len == 32
  uint64 block_height = 2;  // inclusion height
}

message StorageAttestation {
  string chain_id = 1;
  bytes signature = 2;               // ed25519 over domain-tagged digest (see §7)
}

message UploadRowsRequest {
  Commitment commitment = 1;
  RlcOrig rlc_orig = 2;
  repeated RowWithProof rows = 3;          // this FSP's assigned subset only
  uint64 valset_height = 4;                // height used to fetch validator set
  PrepayRef prepay = 5;                    // MUST be included
  optional uint64 original_length = 10;    // can be used to verify padding
}

message UploadRowsResponse {
  StorageAttestation attestation = 1;
  optional uint32 backoff_ms = 10;         // server-advised cooldown under pressure
}

message GetRowsRequest {
  Commitment commitment = 1;
}

message GetRowsResponse {
  repeated RowWithProof rows = 1;          // server returns all rows it holds for the commitment
  optinal ttl = 2;                         // retention horizon (epoch minute)
  optional uint32 backoff_ms = 10;         // server-advised cooldown under pressure
}

service Fibre {
  rpc UploadRows(UploadRowsRequest) returns (UploadRowsResponse);
  rpc GetRows(GetRowsRequest) returns (GetRowsResponse);
}
```

### 6.2 Streaming API (optional; for large per-FSP payloads)

```proto
syntax = "proto3"; package fibre.v1;

message Init {
  bytes commitment = 1;
  uint64 total_chunks = 2;
  uint64 total_bytes = 3;
  uint64 valset_height = 4;
  PrepayRef prepay = 5;
}
message Chunk  { uint32 index = 1; bytes data = 2; } // ≤ row_size per chunk
message Finish { bytes sha256 = 1; }

message UploadReq { oneof kind { Init init = 1; Chunk chunk = 2; Finish end = 3; } }

message UploadResp {
  StorageAttestation attestation = 1;
}

service FibreStreaming { rpc Upload(stream UploadReq) returns (UploadResp); }
```

---

## 7) Attestations (sign/verify)

**Preimage (v2)**

```
sign_bytes = sha256(
  "FIBRE/v1" ||
  commitment ||
  chain_id ||
  be64(prepay_block_height) ||
)
```

* Binds to network (`chain_id`)  and **expiry** based on pre-payment height (single TTL; no promotion).

**Server**

* After verifying **Pre-Payment inclusion** and all row checks, set `expiry = now + retention_ttl`, compute `expiry_epoch_minute`, **sign**, and return.

**Client**

* Rebuild preimage; verify **ed25519** against validator consensus key from **valset\@height**; ensure `expiry_epoch_minute` is in the future.

---

## 8) Server architecture

### 8.1 Components

* **Store**: **Badger** (v1). Alternatives deferred (§8.6).
* **Fibre gRPC**: Bulk + (Optional: Streaming).
* **Pre-Payment Verifier**: checks inclusion of `prepay_tx_hash` at `prepay_block_height` (Celestia).
* **ValSet Service**: fetches **validator set at `valset_height`**.
* **Rate limiter**: throughput cap and fairness.

### 8.2 Badger layout 

Keys:

* `d/<commitment>` → this FSP’s **assigned rows** blob (big value; includes metadata such as `row_size`).
* `b/<YYYYMMDDHHmm>/<commitment>` → minute bucket index for **retention**.

**Store API**

```go
type Store interface {
  Put(ctx context.Context, key [32]byte, value []byte, expiry time.Time) error
  Get(ctx context.Context, key [32]byte) ([]byte, bool, error)
  RunGC(ctx context.Context)
  Close() error
}
```

**GC**: sweep the current minute bucket; delete corresponding `d/*`; run Badger vlog GC opportunistically.

### 8.3 Upload verification pipeline (mandatory)

1. **Pre-Payment inclusion**: verify `(prepay_tx_hash, prepay_block_height)` on-chain. Verify that `prepay_block_height.timestamp + retention_ttl > now`.
2. Build **rsema1d verification context** from `rlc_orig`.
3. **Assignment check**: recompute **Assignment(commitment, valset\@valset\_height)**; ensure incoming row indices are exactly this validator’s slice (no extras/dupes).
4. For each row: verify **Merkle proof** against `commitment` and **Context-Based Verification** (RLC).
5. Persist rows with `expiry = prepay_block_height.timestamp + retention_ttl` (write + sync).
6. **Sign**  and return **attestation**.

### 8.4 Get pathway

* Accept `GetRowsRequest{commitment}`.
* Recompute Assignment and confirm this server has a slice for `commitment`; return **all stored rows** for it.

### 8.5 API limits & fairness

* Enforce `server_rows_per_message_limit` and maximum serialized request size.
* **Rate-limit** ingress to **\~20 MiB/s** (configurable). With \~**163–164 × row\_size** bytes per request at V≈100, this yields \~**14–16 RPS** for 8 MiB blobs.
* Include `backoff_ms` in responses to guide clients under pressure.
* Optional: blacklist abusive clients by IP or signer.

### 8.6 Storage roadmap

* Start with Badger. Later: prototype **SQL-light** and **epoch-bucket** stores and pick by measured throughput and GC behavior.

---

## 9) Security considerations

* **Receipt domain** (`"FIBRE/v1"`) prevents cross-protocol replay.
* **Chain binding**: `chain_id`, `prepay_block_height`, bind acceptance to a concrete chain state.
* **Row verification** includes **Context-Based Verification** (RLC), preventing malleability via valid-looking but undecodable rows.
* **Non-overlapping assignment** removes ambiguity about which FSP should store which rows.

---

## 10) Observability

**Client metrics**: encode latency, **row\_size** chosen, upload latency per FSP, attestation verify failures, quorum time, prepay submit & inclusion latency/height.

**Server metrics**: RPS, bytes/s, assignment mismatches, proof verify failures, RLC failures, write time, GC sweep durations, attestation count, `backoff_ms` emissions, Badger vlog GC stats, prepay inclusion verification latency.

---

## 12) Testing plan

**Unit**: Assignment determinism given `(commitment, valset@height)`; RLC context verification; attestation digest vectors; **row\_size rounding** (64 B multiple) with **K=4096**.

**Integration**: end-to-end Put/Get/Refresh with N mock FSPs; prepay inclusion verification (happy & failure paths); assignment mismatch rejections; failure injection (missing rows, wrong proofs).

**Load**: sustain ≥ 60 MiB/s writes across concurrent Puts with realistic `row_size` (e.g., 8–32 KiB), validate rate-limit behavior; verify GC keeps DB bounded.


**Compatibility**: bulk ↔ streaming parity; client library mode vs light-node mode; Active vs Lazy ValTracker.

---

## 13) Reference helpers (non-normative)

### 13.1 Attestation helpers (Go)

```go
const domain = "FIBRE/v1"

func Digest(
  commitment [32]byte, chainID string, valsetHeight uint64,
  prepayTxHash [32]byte, prepayBlockHeight uint64, expiryEpochMinute uint64,
) [32]byte {
  // sha256(domain || commitment || chainID ||  be64(prepayBlockHeight))
}

func VerifyEd25519(pub ed25519.PublicKey, sig []byte, d [32]byte) bool { /* ... */ }
```

### 13.2 Assignment (permutation idea; pseudocode)

```go
// Inputs: seed=commitment, n=total_rows=16384, valset (pubkeys), height
shares := [0..n-1]
sort(shares, by: sha256("share" || seed || u32be(i)))

vals := indexValidatorsByPubKey(valset) // deterministic base indexing
perm := [0..len(vals)-1]
sort(perm, by: sha256("validator" || seed || u32be(j)))

base := n / len(vals)
r := n % len(vals)

// Build mapping: validator perm[t] gets slice [offset, offset + size)
offset := 0
for t := 0; t < len(vals); t++ {
  size := base
  if t < r { size = base + 1 }
  assignedShares := shares[offset : offset+size]
  map[perm[t]] = assignedShares
  offset += size
}
```

### 13.3 Row size rounding

* `row_size` is chosen per blob; **must be a multiple of 64 B**.
* `row_size = round_up_64B( ceil(original_length / 4096) )`, then **cap at 32 KiB**.

---

## 14) Lifecycle diagrams (ASCII)

### 14.1 Upload (with Pre-Payment)

```
Client                 FSP (many)                 Chain
  |  prepay submit   |                             |
  |----------------->|                             |
  |  wait inclusion  |                             |
  |<-- height+hash --|                             |
  |  encode+valset   |                             |
  |----------------->|                             |
  | UploadRows       |-- verify prepay incl. -->   |
  | (bulk/stream)    |-- verify (proof+RLC) -->    |
  |                  |-- check assignment -->      |
  |                  |-- persist -> sign attn -->  |
  |<----- attns -----|                             |
  | aggregate 2/3    |                             |
```

### 14.2 Refresh (no re-upload)

```
Client                  Chain                 FSP
  |--- Prepay --------> |                     |
  |                     |                     |-- (update expiry) -->|
  |<-- inclusion ------ |                     |
  |                     |                     |
```

---

## 15) Bulk vs Streaming trade-off (explicit)

* **Bulk** is simpler and remains default.
* If `rows_per_val × row_size` approaches multi-MiB per request (e.g., V≈100 & `row_size=32 KiB` → \~5.2 MiB), consider **Streaming** (`Init → Chunk → Finish`) for progressive verification/backpressure.
* Both APIs **coexist**; server may signal preferred mode via error or `backoff_ms`.

---

## 16) Configuration (suggested defaults)

```toml
[row]
# Originals and totals
original_count = 4096                 # 2^12 originals (K)
encoding_factor = "1:3"               # parity = 3 × originals
total_count = 16384                   # derived: original_count * 4

# Row size is chosen per blob; must be a multiple of 64 bytes.
min_size = 64                         # bytes
max_size = 32768                      # 32 KiB (keeps max_blob_size at 128 MiB)

[client]
send_workers = 20
read_workers = 20
mode = "library"                      # or "light-node"
valtracker = "lazy"                   # or "active"

[server]
retention_ttl = "24h"                 # single TTL; no promotion
rows_per_message_limit = 165          # e.g., ceil(16384 / 100) + 1
throughput_cap_bytes_per_sec = 10485760  # 10 MiB/s

[payment]
confirmations_required = 0            # adjust if you want extra safety
```

# Escrow Client (separate component)

## Purpose & scope

A thin library that:

* Manages **escrow state** for an owner (balance bootstrap, proofs, reconciliation).
* **Builds & signs PaymentPromise (PP)** sign-bytes.
* Provides a **Funding API** (create/deposit/withdraw/process) with **tx inclusion proof + balance state proof** returns.
* **Submits PayForFibre** (PFF) using a single **Default FSP (DFSP)** as a relay (or falls back to direct chain).

> This client is independent of data upload/rows/assignment. FibreDA code calls into it.

## Trust & data sources

* Primary: **Default FSP (DFSP)** as a gateway for: `QueryEscrowAccount(owner)` (with state proof) and `SubmitPayForFibre`.
* Fallback: direct chain RPC for the same queries/transactions if DFSP is down or untrusted.

## Responsibilities

1. **Bootstrap balance** at init:

    * Query `EscrowAccount(owner)` via DFSP → cache `{available_balance, proof(height)}`.
    * If DFSP unavailable/invalid proof → fetch from chain directly.
2. **Promise construction**:

    * Build PP: `{owner, namespace(v2), blob_size, commitment, row_version, creation_height}`.
    * Sign PP sign-bytes (per `x/fibre`).
3. **Balance reconciliation**:

    * On upload rejection with `InsufficientBalanceProof`, **update cache** to the proved `available_balance@proof_height`.
    * Track reported **pending\_promises** (advisory until efficient proving is added).
4. **AutoFund (optional)**:

    * When proof says “insufficient,” submit **Deposit**; wait for **tx inclusion proof**; re-query balance proof via DFSP; retry.
5. **Submit PFF**:

    * Prefer `SubmitPayForFibre` via DFSP (single hop); verify tx result (optionally with tx proof). Fallback to direct chain if DFSP fails.

## Public API (library surface)

```go
type FundingTxResult struct {
  TxHash            string
  InclusionHeight   uint64
  InclusionProof    []byte
  NewAvailableBalance string
  BalanceProofHeight  uint64
  BalanceStateProof   []byte
}

type EscrowClient interface {
  // Bootstrap & cached getters
  Init(ctx context.Context) error                         // fetch balance@h0 via DFSP (+proof)
  CachedAvailableBalance() (bal string, height uint64).  // For debug/inspection

  // PaymentPromise
  BuildAndSignPP(ns Namespace, blobSize uint64, commitment [32]byte,
                 rowVersion uint32, creationHeight uint64) (PaymentPromise, error)

  // Submit PayForFibre (relay-first, fallback direct)
  SubmitPayForFibre(ctx context.Context, pp PaymentPromise, sigs [][]byte) (txHash string, height uint64, txProof []byte, err error)

  // Funding API (manual)
  CreateEscrow(ctx context.Context, initialDeposit string) (FundingTxResult, error)
  DepositToEscrow(ctx context.Context, amount string) (FundingTxResult, error)
  RequestWithdrawal(ctx context.Context, amount string) (FundingTxResult, error)
  ProcessWithdrawal(ctx context.Context, requestedAt uint64) (FundingTxResult, error)

  // Reconcile on server feedback
  ApplyInsufficientBalanceProof(proof InsufficientBalanceProof) error
}
```

## DFSP selection & failover

* Allow user to configure a list of FSP endpoints (gRPC URLs).
* If DFSP fails to respond or returns an invalid proof, try validator from the ValTracker list.

## Errors & idempotency

* Treat **Create/Deposit/Withdrawal** as idempotent by `(txHash, owner)`.
* Cache last good `{balance, proof(height)}`; only advance on strictly higher proof heights (after chunks submission and val signing over fibre blob).

## Security notes

* Never accept balance numbers without a **state proof** and **height**.
* Keep PP signing keys separate from validator comms keys.
* PFF relay via DFSP is safe: DFSP can’t forge quorum signatures; client retains fallback to direct chain.

---

# Escrow Server (separate component)

## Purpose & scope

A minimal **gateway** service (co-located with an FSP) that:

* Answers **escrow balance queries** with **chain state proofs**.
* Relays **`MsgPayForFibre`** to the chain and returns tx results (optionally with **tx inclusion proof**).

> This service has no row storage logic; it’s orthogonal to FibreDA/FSP.

## Trust model

* Clients **verify** proofs and tx results; server is a **convenience gateway**, not a trust root.
* Multi-tenant: server must not leak other users’ pending objects beyond what is required/proved.

## Required endpoints (gRPC; names are illustrative)

```proto
syntax = "proto3";
package escrow.v1;

// Tx relay for PayForFibre and funding txs
service EscrowTx {
  rpc SubmitPayForFibre(SubmitPFFRequest) returns (SubmitPFFResponse);
  // Optional helpers:
  rpc SubmitCreateEscrow(SubmitCreateEscrowRequest) returns (FundingTxResponse);
  rpc SubmitDepositToEscrow(SubmitDepositRequest)    returns (FundingTxResponse);
  rpc SubmitRequestWithdrawal(SubmitRequestWithdrawalRequest) returns (FundingTxResponse);
  rpc SubmitProcessWithdrawal(SubmitProcessWithdrawalRequest) returns (FundingTxResponse);
}

// Messages (abridged for brevity)
message SubmitPFFRequest {
  string chain_id = 1;
  PaymentPromise promise = 2;
  repeated bytes validator_signatures = 3;
}
message SubmitPFFResponse {
  string tx_hash = 1;
  uint64 inclusion_height = 2;
  bytes  tx_inclusion_proof = 3;  // optional
}

message FundingTxResponse {
  string tx_hash = 1;
  uint64 inclusion_height = 2;
  bytes  tx_inclusion_proof = 3;
  string new_available_balance = 4;
  uint64 balance_proof_height = 5;
  bytes  balance_state_proof = 6;
}
```

## InsufficientBalanceProof 

* Returned by **FibreDA servers** during upload gating **and** by Escrow Server on explicit request.
* Contents in v1 (baseline; efficient proving TBD):

    * `owner`, `chain_id`, `proof_height`
    * `available_balance`, `escrow_state_proof`
    * `pending_promises[]` (best-effort list of PP that appear unpaid/unsigned on chain)

```proto
message InsufficientBalanceProof {
  string owner = 1;
  string chain_id = 2;
  uint64 proof_height = 3;
  string available_balance = 4;
  bytes  escrow_state_proof = 5;
  repeated PaymentPromise pending_promises = 6;
}
```

## Server behavior

* **QueryEscrowAccount**: fetch app state at a **specific height**; return balance + **state proof**.
* **SubmitPayForFibre**: build and broadcast tx, then:

    * Return `{tx_hash, inclusion_height}`; include **tx proof** if client asks.
    * Optional: also return **new balance proof** post-inclusion (one more query).
* **Rate limiting**: per owner/IP; stricter on tx relay than queries.
* **Privacy**: redact unrelated pending data; scope pending list to the **owner**.

## Observability

* Metrics: query latency, proof construction time, tx relay success/fail, rate-limit hits.
* Logs: request/response IDs, proof heights, tx hashes (no secrets).

## Failure modes & fallbacks

* If **proof construction** fails: return an explicit error with the last chain height observed.
* If **relay** fails: include mempool/abci error; client should **retry** or **fallback direct** to chain.
* If **chain lag**: expose current indexer height; client decides to wait or fallback.

---

## How these pieces connect (without mixing with FibreDA)

* **FibreDA client** uses **Escrow Client** for: PP signing, balance bootstrap, AutoFund, and PFF submission.
* **Escrow Client** talks to **Escrow Server (DFSP)** first, and the **chain** as fallback.
* **FibreDA servers** (not part of escrow) still **gate uploads** with `QueryValidatePaymentPromise` and may emit `InsufficientBalanceProof`—the **Escrow Client** knows how to consume that proof and update its cache.

If you want, I can splice these sections straight into your spec (right before “FibreDA — Full Specification v1.2”) or ship a tiny **patch/diff** that inserts them and cross-links only where necessary.

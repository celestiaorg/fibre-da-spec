# `x/fibre` Prepay Specification

## Abstract

The `x/fibre` prepay mechanism enables users to first pay for data in full to be published by the Celestia validator set. This specification outlines a payment system where users first submit a payment for their data commitment, then validators sign over the data chunks they receive, and finally aggregated signatures are submitted to confirm the blob.

## Contents

1. [State](#state)
1. [Messages](#messages)
1. [Events](#events)
1. [Queries](#queries)
1. [Parameters](#parameters)
1. [Client](#client)

## State

The fibre prepay module maintains state for tracking payments and module parameters.

### Params

```proto
message Params {
  option (gogoproto.goproto_stringer) = false;
  uint32 gas_per_blob_byte = 1
      [ (gogoproto.moretags) = "yaml:\"gas_per_blob_byte\"" ];
}
```

#### `GasPerBlobByte`

`GasPerBlobByte` is the amount of gas consumed per byte of blob data when a `MsgPayForFibre` is processed. This determines the gas cost for prepaying for fibre blob inclusion.

### Payment Tracking

The module stores payment records to enable validators to quickly verify that payments have been made for specific commitments. This is essential for validators joining the network via state sync, allowing them to query payment history without indexing events.

```proto
message PaymentRecord {
  // payment_key is a composite key: hash(signer + commitment)
  bytes payment_key = 1;
  // signer is the original payer's address
  string signer = 2;
  // commitment is the blob commitment that was paid for
  bytes commitment = 3;
  // namespace is the namespace the payment covers
  bytes namespace = 4;
  // blob_size is the size of the blob being paid for
  uint32 blob_size = 5;
  // block_height is the height at which payment was made
  int64 block_height = 6;
  // payment_count tracks how many times this commitment has been paid for
  uint32 payment_count = 7;
}
```

#### Payment Indexing Strategy

Payments are indexed to optimize for both fast lookups and efficient pruning:

**Primary Index**: `payment_key = hash(signer + commitment)`
- Enables O(1) lookup for the most common query: "Has signer X paid for commitment Y?"
- Key format: `payments/{hash(signer + commitment)}` → `PaymentRecord`

**Secondary Indexes**:
- **By Commitment**: `commitments/{commitment}/{signer}` → `payment_key` (for multiple payments per commitment)
- **By Signer**: `payers/{signer}/{commitment}` → `payment_key` (for user payment history)
- **By Block Height**: `pruning/{block_height}/{payment_key}` → `null` (for time-based pruning)

This indexing design allows:
1. Fast verification during `MsgConfirmFibreBlob` processing
2. Support for multiple payments per commitment from different signers
3. Efficient time-based cleanup of old payment records
4. Quick signer payment history queries

#### Pruning Mechanism

Payment records are automatically pruned after 24 hours.

**Pruning**:
- Payment records older than 24 hours are removed
- Pruning occurs during `EndBlock` processing each block
- Uses the `pruning/{block_height}/{payment_key}` index to efficiently identify records to delete
- Only processes records from `current_height - 144` to maintain constant time complexity

**Pruning Process** (executed in `EndBlock`):
1. Calculate pruning height: `pruning_height = current_height - 144`
2. Iterate over `pruning/{pruning_height}/*` keys
3. For each payment_key found:
   - Delete primary record: `payments/{payment_key}`
   - Delete secondary indexes: `commitments/*` and `payers/*` entries
   - Delete pruning index entry: `pruning/{pruning_height}/{payment_key}`
4. Process completes in O(n) where n = payments made 24 hours ago

## Messages

### MsgPayForFibre (PFF)

`MsgPayForFibre` (shortened to PFF) pays for a single blob to be included in the Fibre service. Unlike `MsgPayForBlobs` which can pay for multiple blobs, PFF is designed to pay for exactly one blob.

#### Message Verification and Processing

When a `MsgPayForFibre` is submitted, the following verification and processing steps occur:

**Stateless Validation (`ValidateBasic`)**:
1. **Signer Validation**: Verify the signer address is a valid bech32 Celestia address
2. **Namespace Validation**: Namespace must be valid and namespace version must be 1
3. **Blob Size Validation**: Blob size must be greater than 0
4. **Commitment Validation**: Commitment must be valid (32 bytes, non-zero)
5. **Share Version Validation**: Share version must be supported (0 or 1)

**Stateful Processing (`PayForFibre`)**:
1. **Gas Consumption**: Calculate and consume gas based on `gas_per_blob_byte * blob_size + fixed_cost`
   - Gas calculation uses the same sparse share calculation as PayForBlobs
   - Fixed cost accounts for signature verification and account access operations
2. **Fee Deduction**: Standard transaction fees are deducted from the signer's account balance
3. **Payment Recording**: Store payment record in state with composite key indexing
4. **Event Emission**: Emit `EventPayForFibre` with payment details

**Gas Calculation Formula**:
```
total_gas = (sparse_shares_needed(blob_size) * share_size * gas_per_blob_byte) + fixed_cost
```

Where `sparse_shares_needed` calculates the number of shares required for the blob data, following the same logic as standard blob transactions but applied to the single blob in PFF.

```proto
// MsgPayForFibre pays for the inclusion of a single blob in the Fibre service.
message MsgPayForFibre {
  // signer is the bech32 encoded signer address
  string signer = 1;
  // namespace is the namespace that the blob is associated with. A
  // namespace is a byte slice of length 29 where the first byte is the
  // namespaceVersion and the subsequent 28 bytes are the namespaceId.
  bytes namespace = 2;
  // blob_size is the size of the blob in bytes.
  uint32 blob_size = 3;
  // commitment is the hash of the row root and the RLC (random linear combination) root.
  bytes commitment = 4;
  // share_version is the version of the share format that the blob
  // associated with this message should use. Must match the share_version
  // used to generate the commitment.
  uint32 share_version = 8;
}
```

#### Generating the Commitment

The commitment for PFF differs from the share commitment used in `MsgPayForBlobs`. For Fibre, the commitment is the hash of:
- The row root of the data
- The RLC (random linear combination) root

### MsgConfirmFibreBlob

`MsgConfirmFibreBlob` contains aggregated signatures from 2/3+ validators confirming they have received, verified, and will serve the blob data. This message wraps the original `MsgPayForFibre` to avoid field duplication.

```proto
// MsgConfirmFibreBlob confirms that validators have received and will serve the blob.
message MsgConfirmFibreBlob {
  // signer is the bech32 encoded address submitting the confirmation
  string signer = 1;
  // original_pff is the original MsgPayForFibre that was paid for
  MsgPayForFibre original_pff = 2;
  // validator_signatures contains the aggregated signatures from validators
  repeated ValidatorSignature validator_signatures = 3;
}

// ValidatorSignature represents a validator's signature over the blob commitment
message ValidatorSignature {
  // validator_address is the validator's address
  string validator_address = 1;
  // signature is the validator's signature over the sign bytes
  bytes signature = 2;
}
```

#### Validation

**Stateless Validation (`ValidateBasic`)**:
- Signer address must be valid
- Original PFF must pass its own `ValidateBasic`
- Must contain at least one validator signature

**Stateful Processing (`ConfirmFibreBlob`)**:
1. **Payment Verification**: Verifies payment exists using `payment_key = hash(original_pff.signer + original_pff.commitment)`
2. **Validator Signature Verification**: Verifies signatures represent at least 2/3 of current validator voting power. Note: Only signatures from the current active validator set are accepted, creating time pressure for users to submit confirmations before validator set changes.
3. **Commitment Inclusion**: Includes only the commitment in the data square (not blob data)
4. **Event Emission**: Emits `EventConfirmFibreBlob`

**Key Differences from PayForBlobs**:
- PayForBlobs includes actual blob data when processed
- MsgConfirmFibreBlob only includes the commitment after validator consensus
- Requires prior payment verification

## Transaction Flow

The Fibre prepay mechanism follows this flow:

1. **Payment Phase**: User submits `MsgPayForFibre` (PFF) transaction containing the blob commitment and payment. This transaction pays the full gas cost upfront.

2. **Data Distribution Phase**: User distributes unique chunks of data to validators using an algorithm outside the scope of this specification.

3. **Validator Signing Phase**: Validators receive data chunks, verify the encoding, and sign over the commitment data. The signature covers:
   - Commitment (row root + RLC root hash)
   - Namespace
   - Original submitting account address
   - Namespace version
   - Share version

4. **Confirmation Phase**: Once 2/3+ validator signatures are collected, a `MsgConfirmFibreBlob` transaction is submitted containing the aggregated signatures and the original PFF message. The confirmation verifies payment was made, then includes only the commitment in the data square after validating signatures.

## Events

### Fibre Events

#### `EventPayForFibre`

| Attribute Key | Attribute Value                    |
|---------------|------------------------------------|
| signer        | {bech32 encoded signer address}    |
| namespace     | {namespace the blob is published to} |
| blob_size     | {size of blob in bytes}            |
| commitment    | {blob commitment hash}             |

#### `EventConfirmFibreBlob`

| Attribute Key    | Attribute Value                      |
|------------------|--------------------------------------|
| signer           | {bech32 encoded submitter address}   |
| original_signer  | {bech32 encoded original PFF signer} |
| namespace        | {namespace the blob is published to} |
| validator_count  | {number of validator signatures}     |

## Queries

### PaymentRecord

Queries a payment record by signer and commitment.

**Request**:
```proto
message QueryPaymentRecordRequest {
  string signer = 1;
  bytes commitment = 2;
}
```

**Response**:
```proto
message QueryPaymentRecordResponse {
  PaymentRecord payment_record = 1;
  bool found = 2;
}
```

### PaymentsByCommitment

Queries all payments made for a specific commitment.

**Request**:
```proto
message QueryPaymentsByCommitmentRequest {
  bytes commitment = 1;
  cosmos.base.query.v1beta1.PageRequest pagination = 2;
}
```

**Response**:
```proto
message QueryPaymentsByCommitmentResponse {
  repeated PaymentRecord payment_records = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}
```

### PaymentsBySigner

Queries all payments made by a specific signer.

**Request**:
```proto
message QueryPaymentsBySignerRequest {
  string signer = 1;
  cosmos.base.query.v1beta1.PageRequest pagination = 2;
}
```

**Response**:
```proto
message QueryPaymentsBySignerResponse {
  repeated PaymentRecord payment_records = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}
```

## Parameters

| Key            | Type   | Default |
|----------------|--------|---------|
| GasPerBlobByte | uint32 | 8       |

## Client

### CLI

#### Transactions

```shell
# Submit a payment for fibre blob
celestia-appd tx fibre pay-for-fibre <hex encoded namespace> <blob_size> <hex encoded commitment> [flags]

# Confirm fibre blob with validator signatures
celestia-appd tx fibre confirm-fibre-blob <original_pff_json> <validator_signatures_json> [flags]
```

#### Queries

```shell
# Query payment record by signer and commitment
celestia-appd query fibre payment-record <signer> <hex encoded commitment>

# Query all payments for a commitment
celestia-appd query fibre payments-by-commitment <hex encoded commitment>

# Query all payments by a signer
celestia-appd query fibre payments-by-signer <signer>

# Query module parameters
celestia-appd query fibre params
```

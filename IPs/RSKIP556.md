---
rskip: 556
title: Blob Carrying Transactions (EIP-4418)
created: 10-APR-2026
author: SM (@smishraiov)
purpose: Sca,Usa
layer: Core
complexity: 3
status: Draft
description: Blob transactions for ephemeral data storage
---

|RSKIP          |556           |
| :------------ |:-------------|
|**Title**      |Blob Carrying Transactions (EIP-4418) |
|**Created**    |APR-10-2026 |
|**Author**     |SM |
|**Purpose**    |Sca,Usa |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |


## Abstract

This is a proposal to implement blob-carrying transactions in Rootstock ([EIP-4844](https://eips.ethereum.org/EIPS/eip-4844), Type 3 in EIP-2718 envelope format). A blob represents 128KB of ephemeral data that can be deleted by nodes after some time. This can be used by L2 rollups as a cheaper alternative to transaction calldata for Data Availability. Blob data cannot be accessed by EVM execution. All blob-related cryptography - including a new precompiled contract -  will consume the same native `c-kgz-4844` [library](https://github.com/ethereum/c-kzg-4844) used by all clients in Ethereum. When producing blocks containing any blob transactions, miners validate and include blob data, commitments and proofs in “sidecars”. Unlike Ethereum, sidecars are not propagated separately. Instead, they are inlined inside block bodies. Thus, no changes are needed to the P2P network. To minimize impact on propagation times and merged mining, at most one blob can be included in a transaction. Furthermore, a block cannot include more than two blobs. These restrictions can be eased in the future. If a block contains any blob transactions, then miners must validate blobs before mining a new block on top of it. When a new blob transaction is detected in the P2P network, a node must validate blob data before forwarding the transaction to peers. Blob data and proofs can be pruned. Therefore, regular full nodes will not be required to validate blobs when synchronizing with the network. However, commitments are stored permanently and nodes must verify the commitment root (a new field) in the block header. Fees from blobs will be added to the block’s cumulative fees and distributed as per REMASC logic. Blob gas (another new field in block header) will not count towards block gas limit.

## Motivation

Since their introduction through EIP-4844 the adoption of blobs has exploded in Ethereum. This was a core part of Ethereum’s rollup-centric approach to scaling. L2 rollups switched to using blobs instead of transaction calldata for Data Availability. On Ethereum, ephemeral storage via blobs is orders of magnitude cheaper than calldata. Each blob can store up to 128KB of data. Therefore, for node operators, the transition from calldata to blobs greatly reduced long term data storage needs (since blobs can be pruned after 18 days). After the recent Fusaka upgrade, Ethereum now has a target of 14 blobs per block, with a maximum of 21 blobs per block. While current usage is typically below this target, it is nevertheless a striking illustration of the massive amount of cheap and temporary storage available for rollups.

Despite prior attempts,  no rollups are currently active on Rootstock. Network congestion has not been an issue on Rootstock and blockspace remains plentiful. Thus, this proposal may not seem urgent from the perspective of scaling Rootstock. However, cheap and ephemeral data storage can help the development of application-specific rollups such as those for payments solutions. There have been earlier proposals to introduce ephemeral storage mechanisms on Rootstock. [RSKIP-281](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP281.md) is the most recent example. The fact that RSKIP-281 is from 2021 is a  reminder that ephemeral storage, while not a pressing issue, is not a novel one either.

Just like EIP-4844 and RSKIP-281, any proposal of this kind for ephemeral storage of transaction data is bound to be fairly complex and involve critical changes to consensus. Therefore, the current lack of urgency is beneficial, as it gives the community more time for deliberation.

## Specification

**Prerequisites:** This proposal requires [RSKIP-543](https://github.com/rsksmart/RSKIPs/pull/543) which introduced EIP-2718 envelope format for typed transactions. It also requires [RSKIP-546](https://github.com/rsksmart/RSKIPs/pull/546) which uses RSKIP-543 to introduce Ethereum’s Type-1 and Type-2 transactions in Rootstock.

### Parameters

| Constant | Value |
| --- | --- |
| `BLOB_TX_TYPE` | `Bytes1(0x03)` |
| `BYTES_PER_FIELD_ELEMENT` | `32` |
| `FIELD_ELEMENTS_PER_BLOB` | `4096` |
| `BLS_MODULUS` | `52435875175126190479447740508185965837690552500527637822603658699938581184513` |
| `VERSIONED_HASH_VERSION_KZG`  | `Bytes1(0x01)` |
| `POINT_EVALUATION_PRECOMPILE_ADDRESS` |  `Bytes20(0x0A)` |
| `POINT_EVALUATION_PRECOMPILE_GAS` | `50000`  |
| `MAX_BLOB_GAS_PER_BLOCK` | `262144` |
| `GAS_PER_BLOB` | `2**17` |
| `HASH_OPCODE_BYTE` | `Bytes1(0x49)` |
| `HASH_OPCODE_GAS` | `3` |
| `RSK_MIN_SIDECAR_REQ_BLOCKS` | `2**16` |
| `RSK_BLOB_GAS_PRICE_DIVISOR` | `8`  |
| `RSK_MAX_BLOBS_PER_TX` | `1` |

In Ethereum, blobs refer to temporary transaction data stored by consensus clients for approximately 18 days, after which the data can be pruned. The actual parameterization for minimum storage duration is in terms of number of blocks, not days. This duration has not changed since the initial deployment of EIP-4844 nearly two years ago (via Dencun hard fork in March 2024). Accordingly, we propose to retain blob data using a parameter `RSK_MIN_SIDECAR_REQ_BLOCKS` which is set at 2**16 Rootstock blocks. At an average block time of 25 seconds, this corresponds to about 19 days. In order to limit the impact of blob transactions on the network, a transaction cannot carry more than `RSK_MAX_BLOBS_PER_TX` blobs. There is no such limit on Ethereum. As in Ethereum, the parameter `MAX_BLOB_GAS_PER_BLOCK`  caps the number of blobs that can be included in a block (and we set this to two intially). These limits can later be modified in the future. 

### Cryptographic Helpers

Blobs are ephemeral. Once pruned, they cannot be recovered from historical block data. Some immutable cryptographic reference must however be preserved to uniquely connect transactions with the blobs they once carried. For rollups, there is also the core issue of data availability - which requires some consensus-based validation of what was sent to the base layer. EIP-4844 satisfied the above needs through the use of cryptographic commitments. The detailed specification is in [KZG polynomial commitments](https://ethereum.github.io/consensus-specs/specs/deneb/polynomial-commitments/). This RSKIP does not introduce any changes to EIP-4844’s cryptographic components. The existent cryptographic tooling (e.g. parameters from Ethereum’s one time, [trusted setup ceremony)](https://github.com/ethereum/kzg-ceremony) from Ethereum will be reused directly.

Users, such as rollup operators, first encode data into blobs. Each blob represents a set of `FIELD_ELEMENTS_PER_BLOB`(i.e. 4096) *field elements*, each of which is of size `BYTES_PER_FIELD_ELEMENT` (i.e. 32 bytes). Each blob must therefore be exactly 128KB, otherwise the existent cryptographic tooling will not work. 

A simple way of encoding data into blobs is to combine 31 bytes of user data with a leading or trailing zero byte  (`0x00`) to obtain a “32-byte chunk”. This chunk is interpreted as a field element. The size (32 bytes) comes from the parameter `BYTES_PER_FIELD_ELEMENT`.  Users continue creating such elements till they have a full blob with `FIELD_ELEMENTS_PER_BLOB`  chunks. If they do not have enough data to encode, they must still fill the blob to 128KB,  e.g. by padding with zero bytes. If they have more data to encode, they must create additional blobs.

Users then compute a KZG commitment for each blob. By doing this, users are committing to a very specific polynomial representation of their data. By contrast, RSKIP-281 uses (Keccak, or user-defined) hashes as a mechanism for users to commit to ephemeral data. Users then generate KZG proofs for these commitments. These proofs form the basis for efficient verification of blob data. The idea is that blobs, commitments, and proofs are submitted together in a transaction. Only commitments are stored permanently as part of historical data on the L1 - blobs and proofs are deleted after some time.

Users who create and post blobs (such as rollup node operators) may choose to store the blobs and proofs (off chain) for as long as they deem necessary, in case there are disputes. Since commitments are stored permanently, users can verify (off-chain) that a paritcular blob was at some point published to and validated by the network.

This flow (commitment and verification) is implemented using the API of a native cryptography library, [c-kzg-4844](https://github.com/ethereum/c-kzg-4844). In Ethereum, all major execution and consensus clients use this library. It has bindings for the usual high level languages C#, Go, [Java](https://github.com/ethereum/c-kzg-4844/blob/main/bindings/java/README.md), Rust, Python etc. This means, the implementers of this RSKIP do not have to worry about cryptographic details such as the `BLS_MODULUS` listed in the parameters. All of the cryptography toolkit developed for EIP-4844 can be reused as is. The readme in the c-kzg-4844 repository lists the main API (cryptographic functions) to compute proofs and verify them efficiently

- `blob_to_kzg_commitment`
- `compute_kzg_proof`
- `compute_blob_kzg_proof`
- `verify_kzg_proof`
- `verify_blob_kzg_proof`
- `verify_blob_kzg_proof_batch`

Baring one, all of these methods use blobs as an input. One method, `verify_kzg_proof` does not require access to the underlying blob. This can be used to verify that the polynomial that a user has committed to evaluates to a specific value at a specific point. This method is used to implement a pre-compiled contract (described later), which can be used to settle “on-chain” disputes regarding claims made about data posted using blobs. This is what the parameters `POINT_EVALUATION_PRECOMPILE_ADDRESS` and `POINT_EVALUATION_PRECOMPILE_GAS`  refer to.

The c-kzg library includes references to newer blob methods from [EIP-7594](https://eips.ethereum.org/EIPS/eip-7594). These are related to the latest version of blob transactions (after the Fusaka upgrade), which is more complex and is intended for peer data availability sampling (Peer DAS). Peer DAS is Ethereum’s current scaling approach through sharding of blobs - so validators only need to store pieces of data (called “cells”) instead of entire blobs. We do not use peer DAS as it is not well suited for Rootstock’s merged mining based consensus. We stick with the original EIP-4844 and require users to precompute proofs for KZG commitments over an entire blob (not to multiple cells as in Peer DAS). As noted earlier, commitments will be stored permanently, while blobs and proofs are ephemeral.

### Type aliases

| Type | Base type | Additional checks |
| --- | --- | --- |
| `Blob` | `ByteVector[BYTES_PER_FIELD_ELEMENT * FIELD_ELEMENTS_PER_BLOB]` |  |
| `VersionedHash` | `Bytes32` |  |
| `KZGCommitment` | `Bytes48` | Perform IETF BLS signature “KeyValidate” check but do allow the identity point |
| `KZGProof` | `Bytes48` | Same as for `KZGCommitment` |
|  |  |  |

The additional checks in the Type aliases table above refer to usage of BLS12-381 in the scheme. BLS12-381 is a *pairing-friendly elliptic curve construction*. When used for BLS signatures, the identity point (point at infinity) is rejected as invalid. However, when used for KZG commitments (as in the current case), the identity point is valid. This is allowed since a blob can (theoretically) commit to the zero polynomial.

A blob is identified by its “versioned hash” which is the `sha256` hash digest of the blob’s KZG commitment with the first byte replaced with the parameter `VERSIONED_HASH_VERSION_KZG`. The following illustrative helper function is from EIP-4844. As suggested by the name, versioning is intended for upgrades and forward compatibility (e.g. other cryptographic commitment schemes). As described below, blob transaction payloads use (32 byte) versioned hashes rather than the original (48 byte) KZG commitments.

```python
def kzg_to_versioned_hash(commitment: KZGCommitment) -> VersionedHash:
    return VERSIONED_HASH_VERSION_KZG + sha256(commitment)[1:]
```

### Blob transaction

Define a new type of EIP-2718 (RSKIP-543) transaction, called “blob transaction”, where the `TransactionType` is `BLOB_TX_TYPE` and the `TransactionPayload` is the RLP serialization of the following `TransactionPayloadBody`:

`[chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, to, value, data, access_list, max_fee_per_blob_gas, blob_versioned_hashes, y_parity, r, s]`

The fields `chain_id`, `nonce`, `max_priority_fee_per_gas`, `max_fee_per_gas`, `gas_limit`, `value`, `data`, and `access_list` follow the same semantics as RSKIP-546.

The field `to`  must not be `nil` and must always represent a 20-byte address. This means that blob transactions cannot have the form of a create transaction.

The field `max_fee_per_blob_gas` is a `uint256` and the field `blob_versioned_hashes` represents a list of hash outputs from `kzg_to_versioned_hash` , one per each blob attached (see note about helper function above).

The [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) `ReceiptPayload` for this transaction is `rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`. Blob gas fields (blob gas used, blob gas price) are not dependent on execution outcome and need not be repeated in receipts.

### Signature

The signature values `y_parity`, `r`, and `s` are calculated by constructing a secp256k1 signature over the following digest:

`keccak256(BLOB_TX_TYPE
 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, 
gas_limit, to, value, data, access_list, max_fee_per_blob_gas, 
blob_versioned_hashes]))`.

See the “Networking” section for two separate representations of blob transactions. An initial version includes blobs, commitments and proofs. This is used for transaction broadcast (e.g. using `eth_sendrawtransaction`) and propagation through the p2p network. A different version is used for long term retrieval which does not contain blob data, commitments or proofs. Note that any mechanism for ephemeral transaction data must also have such dual representation: a network representation for P2P propagation, and a canonical version which does not contain ephemeral data for permanent storage. 

### Header extension

The block header encoding is extended with two new fields:

- `blob_gas_used` is the total amount of blob gas consumed by the transactions within the block.
Those familiar with EIP-4844 will observe there is no “excess blob gas” in header. This is because, unlike Ethereum, there is no target level of blobs here.
- `blob_commitments_root` is the root of a trie that contains all the blob commitments included in a block.

Commitments have an index (within each block). To obtain the blob commitment root, the commitments can be stored in order as leaf nodes in Rootstock’s Trie data structure without any RLP encoding. If there are no blobs in a block, then the `blob_commitments_root` will be set as  `EMPTY_HASH` for the hash of an [empty trie](https://github.com/rsksmart/rskj/blob/6a4c9a24813e01f938e3074afa3e81163d8b4916/rskj-core/src/main/java/co/rsk/trie/Trie.java#L75), which is the Keccak hash of a RLP encoded  empty byte array.  In Ethereum, blob commitment roots are stored in beacon block headers. In Rootstock, there is no separate consensus chain (or consensus block), these new fields will be added to every Rootstock block once this consensus change becomes effective.

### **Gas accounting**

As in EIP-4844, gas accounting for blobs is controlled by `GAS_PER_BLOB` . The value 2**17 is just the the number of bytes in a blob (128KB). The maximum number of blobs in a block is initially limited to two (2) by setting `MAX_BLOB_GAS_PER_BLOCK` at 2**18.

Blob gas is priced differently than execution. In particular, the minimum gas price for blobs is obtained by dividing the current `min_gasPrice` by `RSK_BLOB_GAS_PRICE_DIVISOR` . For example if the current `min_gasPrice` is 0.026 `gwei`, then the minimum gas price for blobs is 0.00325 `gwei`. This rescaling makes the cost of using blobs about 128 times cheaper than using transaction calldata. The factor 128 is the product of the Divisor (8) and the cost of calldata, which is 16 gas/byte (for non-zero bytes).

```python
def calc_blob_fee(header: Header, tx: Transaction) -> int:
    return get_total_blob_gas(tx) * min_gasPrice/RSK_BLOB_GAS_PRICE_DIVISOR

def get_total_blob_gas(tx: Transaction) -> int:
    return GAS_PER_BLOB * len(tx.blob_versioned_hashes)
```

The block validity conditions are modified to include blob gas checks. The actual `blob_fee` as calculated via `calc_blob_fee` is deducted from the sender’s balance before transaction execution. It is not refunded in case of transaction failure.

### Opcode to get versioned hashes

A new instruction `BLOBHASH` is added (with opcode `HASH_OPCODE_BYTE`). It reads `index` from the top of the stack as big-endian `uint256`, and replaces it on the stack with `tx.blob_versioned_hashes[index]`if `index < len(tx.blob_versioned_hashes)`, and otherwise with a zeroed `bytes32` value. The opcode has a gas cost of `HASH_OPCODE_GAS`. This is only meaningful inside the execution of a blob-carrying transaction. Without `BLOBHASH`, a contract (called by Blob-carrying transaction) would not know which commitment belongs to the transaction. With this information, a contract can later use the **point-evaluation precompile** (see below) to verify a KZG proof against that commitment.

### Point evaluation precompile

Add a precompile at `POINT_EVALUATION_PRECOMPILE_ADDRESS` that verifies a KZG proof which claims that a blob (represented by a commitment) evaluates to a given value at a given point.

The precompile costs `POINT_EVALUATION_PRECOMPILE_GAS` and executes the following logic:

```python
def point_evaluation_precompile(input: Bytes) -> Bytes:
    """
    Verify p(z) = y given commitment that corresponds to the polynomial p(x) and a KZG proof.
    Also verify that the provided commitment matches the provided versioned_hash.
    """
    # The data is encoded as follows: versioned_hash | z | y | commitment | proof | with z and y being padded 32 byte big endian values    assert len(input) == 192
    versioned_hash = input[:32]
    z = input[32:64]
    y = input[64:96]
    commitment = input[96:144]
    proof = input[144:192]

    # Verify commitment matches versioned_hash    assert kzg_to_versioned_hash(commitment) == versioned_hash

    # Verify KZG proof with z and y in big endian format    assert verify_kzg_proof(commitment, z, y, proof)

    # Return FIELD_ELEMENTS_PER_BLOB and BLS_MODULUS as padded 32 byte big endian values    return Bytes(U256(FIELD_ELEMENTS_PER_BLOB).to_be_bytes32() + U256(BLS_MODULUS).to_be_bytes32())
```

The precompile MUST reject non-canonical field elements (i.e. provided field elements MUST be strictly less than `BLS_MODULUS`). The points evaluation precompile is for smart contracts, mainly rollups, to help settle disputes. Note that, with the Fusaka upgrade, Ethereum introduced an additional precompile for multi-point evaluation for batched verification. This proposal does not include the multi-point precompile - it can be introduced in the future if needed.

### Networking: Blob Sidecars

For transaction gossip responses, the EIP-2718 `TransactionPayload` of the blob transaction is wrapped to become:`rlp([tx_payload_body, blobs, commitments, proofs])` . 

Each of these elements are defined as follows:

- `tx_payload_body` - is the `TransactionPayloadBody` of standard EIP-2718 blob transaction as described above.
- `blobs` - list of `Blob` items
- `commitments` - list of `KZGCommitment` of the corresponding `blobs`
- `proofs` - list of `KZGProof` of the corresponding `blobs` and `commitments`

During  initial broadcast and gossip, the *network* representation of a blob transaction, is `BLOB_TX_TYPE || rlp([tx_payload_body, blobs, commitments, proofs])` . Note the difference with the canonical representation which is `BLOB_TX_TYPE || rlp(tx_payload_body)` . Only the canonical version is stored permanently in blockchain database.

When a new Type-3 transaction shows up in the network, a node MUST validate `tx_payload_body` and verify the wrapped data against it before forwarding it to peers. To do so, it must ensure that:

- There are an equal number of `tx_payload_body.blob_versioned_hashes`, `blobs`, `commitments`, and `proofs`.
- That the total number of blobs does not exceed `RSK_MAX_BLOBS_PER_TX` . Rootstock nodes also validate transaction length in `TxPendingValidator` . This check should to be modified for blob transactions since they will surely violate the 128KB limit (a blob is 128KB by itself). For example , the `getSize()`  method should only check the length of the transaction payload (the canonical version) and not the wrapped version (network representation). The account balance check logic must also be modified to ensure the sender can pay fees for both execution gas as well as blob gas (which is deterministic per blob).
- The KZG `commitments` hash to the versioned hashes, i.e. `kzg_to_versioned_hash(commitments[i]) == tx_payload_body.blob_versioned_hashes[i]`
- The KZG `commitments` match the corresponding `blobs` and `proofs`. (Note: this can be optimized using `verify_blob_kzg_proof_batch`, with a proof for a
random evaluation at a point derived from the commitment and blob data for each blob)

Once blob transactions are included in a block, the `transactions` list only includes the canonical version - i.e. the standard EIP-2718 blob transaction `TransactionPayload`  without the wrapper. The other fields from the wrapped version (`commitments`, `proofs`, and `blobs`) are added to a new field (a list)  `Blobsidecar` in the block body. In other words, the blob data are included “inline” within blocks. This is an important distinction from Ethereum, where blobs (with corresponding commitments and proofs) are propagated separately, as “sidecars”. This is why we call the new field in the block body  `Blobsidecar` - even though, technically, they are inline.

With this structure, a block can be visualized as follows

```python
Block
├── Header
│    ├── parentHash
│    ├── stateRoot
│    ├── transactionsRoot
│    ├── receiptsRoot
│    ├── blobCommitmentsRoot
│    ├── blobGasUsed
│    └── ...
└── Body
	├── Transactions
	├── Uncles
	└── BlobSidecar
			└── per-tx blob bundle
			   ├── txIndex
			   ├── commitment
		     ├── proof       #can be pruned
		     └── blobbytes   #can be pruned
```

In Rootstock, nodes automatically broadcast all transactions that pass initial validation to their peers. However, blocks are broadcast only to a subset (a square root of the number) of peers. The others are just sent blockhashes, and they can request full block bodies as needed. This proposal does not change any of the semantics for the P2P propagation of transactions and blocks. Blob transactions and blocks containing them are relatively larger, which is why the initial values for the parameters are set conservatively to limit the number of blobs in a single transaction to 1 (using `RSK_MAX_BLOBS_PER_TX`) and that in a block to 2 (using `MAX_BLOB_GAS_PER_BLOCK`). 

### Block Validation

A miner who includes blob transactions in a Rootstock block must populate the `Blobsidecar` with corresponding blobs, commitments, and proofs. The block header must contain the correct `blob_gas_used`  and `blob_commitment_root` . A miner must also validate the commitments of any blobs included in a parent block, using the blobs and proofs from the `Blobsidecar` of the parent block. Any block that fails the check is invalid and cannot be used as a parent. These checks are are added to core consensus rules for block production.

Once a block is produced, all clients validate the blob fields in the block header. Non-mining full nodes do not have to validate commitments. Blobs are ephemeral and depending on how far behind the tip of the chain a node is, blobs and proofs may not be available. This (non availability of blob) will certainly be the case for a node that is synchronizing with the network and is more than  `RSK_MIN_SIDECAR_REQ_BLOCKS` blocks (approximately 19 days) behind the chain tip, when blobs may have been pruned. Although some archival nodes (perhaps those operated by rollups, block indexers, or explorers) may retain blobs for longer. As mentioned in the Networking section, once a node is fully synchronized, it must validate blob transaction payload and wrapper data.

```python
def validate_block(block: Block) -> None:
    ...

    blob_gas_used = 0

    for tx in block.transactions:
        ...

        # modify the check for sufficient balance        
        max_total_fee = tx.gas * tx.max_fee_per_gas
        if get_tx_type(tx) == BLOB_TX_TYPE:
            max_total_fee += get_total_blob_gas(tx) * tx.max_fee_per_blob_gas
        assert signer(tx).balance >= max_total_fee

        ...

        # add validity logic specific to blob txs        
        if get_tx_type(tx) == BLOB_TX_TYPE:
        # there must be at least one blob            
        assert len(tx.blob_versioned_hashes) > 0

        # all versioned blob hashes must start with VERSIONED_HASH_VERSION_KZG
        for h in tx.blob_versioned_hashes:
                assert h[0] == VERSIONED_HASH_VERSION_KZG

        # ensure user was willing to pay minimum blob gas fee        
        assert tx.max_fee_per_blob_gas >= min_gasPrice/RSK_BLOB_GAS_PRICE_DIVISOR

        # keep track of total blob gas spent in the block            
        blob_gas_used += get_total_blob_gas(tx)

       # ensure the total blob gas spent is at most equal to the limit    
       assert blob_gas_used <= MAX_BLOB_GAS_PER_BLOCK

       # ensure blob_gas_used matches header    
       assert block.header.blob_gas_used == blob_gas_used
```

### Data Deletion and Permanent Storage

The KZG commitment scheme is deterministic. Identical blobs, i.e. blobs created from the same underlying data (and encoded in identical manner) will always result in the same commitment and versioned-hash. Proofs, however, can be different because they can be created for different singleton points, or over multiple points.

Nodes can delete block-level blob sidecars after `2**16`  (`RSK_MIN_SIDECAR_REQ_BLOCKS`) blocks (approximately 19 days assuming 25 seconds block time on Rootstock). Deletion is a node implementation issue and there is no consensus check for deletion or retention. The only requirement for consensus is that blob commitments must be stored permanently in the historical database - so that users can still verify (off chain) that a certain blob was in fact seen on and validated by the network.

To make pruning easier, the node implementations can store a copy of only the commitments (and reference indexes) together with other block data. The full contents of `Blobsidecar` can be stored separately and deleted after the retention period.

Versioned hashes can be computed from commitments, but not the other way around. Since versioned hashes appear in transactions, they cannot be removed. A Rootstock node must store both. Versioned-hashes are 32-bytes each while commitments are 48 bytes each. Thus, a Rootstock node will store a total of 80 bytes permanently for each blob ever included in a block.  However, 80 bytes is very small (a 1:1600 ratio) compared to the 128KB for the original blob, which is deleted. Recall that if apps use transaction calldata (instead of blobs) that data can never be removed.

### Incentives

For rollups, blobs offer a cheaper alternative to transaction calldata for Data Availability. For miners, blobs transactions are a new source of fees. For all node operators, blob transactions imply reduced long term storage requirements compared to calldata. The computational burden of verifying blob proofs is not expected to be significant.

Apart from blob fees, Ethereum does not offer any direct incentives to validators. Even the fees became effective only after the recent Fusaka upgrade. In the original (Dencun) version, blob fees were close to zero and even the meager amount collected was burnt. There are no explicit penalties for pruning blobs too early, or for “not serving” blobs to peers when requested. Therefore, other than new blob fees for miners (shared via REMASC), we do not introduce any other rewards or penalties related to blob transactions. This means, in particular, that there is no guarantee that blobs will be stored or served by every Rootstock node for 19 days. The only guarantee provided by the network is the initial validation of blob data when blob-carrying transactions are mined. This guarantees that the commitments stored permanently in blocks were verifiably linked to an underlying blob data that was seen on the network and validated by miners (though that blob may no longer be available on chain). Users (e.g. rollups) should make their own arrangements for storing blobs and, when necessary, use commitments to prove the historical inclusion and validation of blob data.

Rootstock is a proof of work chain where block production resembles a Poisson process. To avoid any significant negative impact on block propagation times (or the uncle rate), the number of blobs is initially restricted to no more than one per transaction and two per block. In Ethereum PoS, block times are more stable and block producers have designated slots - which allow for much higher limits (currently 21 blobs per block). Unlike Ethereum, our main motivation is not to scale Rootstock. This is why we do not have a “target” parameter for the number of blobs (unlike EIP-4844). Our primary goal is to introduce this popular mechanism for ephemeral data - but with caution. The per-transaction and per-block limits can be increased in the future - informed by actual adoption and usage experience. 

## Summary of changes to Rootstock consensus & Node

Given the complexity of the proposal, the following list is not exhaustive

- New Type-3 Transaction (EIP 2718 / RSKIP-543 transaction type)
- `eth_sendrawtransaction` should be extended to support submission of blob transactions
- Block header extension for blob gas fields
    - Blob fees are added to REMASC pool (REMASC itself is unaffected)
    - Blob gas does not count towards block gas limit.
- Blob Validation
    - Miners must validate all blobs that are included in a parent block before building a new block on top of it. This can be performed efficiently with batched proof verification.
    - All full nodes must validate every blob attached to a blob-carrying transaction when such a transaction appears in the P2P network. Any transaction that fails blob validation will be dropped and not propagated to peers.
    - If a block contains blob transactions, all full nodes must validate blob commitment root (commitments are stored permanently, only actual blobs and proofs can be pruned)
- Block sidecars must be stored in database
    - Only commitments and transaction reference indexes are stored permanently. Blobs and proofs can be deleted.
- Integrate c-kzg-4844 library to directly use Ethereum’s tooling for blob cryptography (KZG commitments, proof verification).
- New Point Evaluation precompile contract - based on the c-kzg-4844 library.
- New EVM op-code for `blobhash` to obtain versioned hashes in the context of a blob transaction.
- Nice to have: A new RPC method `eth_feeHistory` . This was introduced in Ethereum after EIP-1559 and later extended after EIP-4844. It responds with execution and blobgas fees from recent blocks. The following is an example response from Sepolia

```bash
curl -X POST [https://sepolia.infura.io/v3/YOUR_API-KEY/](https://sepolia.infura.io/v3/2e1de8421ba94b4a867b39a5f71c20c8) \
-H "Content-Type: application/json" \
-d '{
"jsonrpc":"2.0",
"method":"eth_feeHistory",
"params":["0x2","latest",[50]],
"id":1
}'
# response (info for 2 recent blocks)
{"jsonrpc":"2.0","id":1,"result":{"oldestBlock":"0xa1ebbd","reward":[["0x65287ca"],["0x67269f1"]],"baseFeePerGas":["0x26b92dca","0x264f76d1","0x2ae1d52c"],"gasUsedRatio":[0.4573437511432302,0.9773379],"baseFeePerBlobGas":["0x7328669","0x7474f4f","0x77195d9"],"blobGasUsedRatio":[0.7142857142857143,0.7619047619047619]}}
```

There are no baseFees in Rootstock (neither for execution nor for blobs). So the new RPC method `eth_feeHistory` can be modified to respond with the values using `gasPrice` for execution gas and populating `BaseFeePerBlobGas` with `min_gasPrice`  divided by `RSK_BLOB_GAS_PRICE_DIVISOR` .

### Activation

Todo: Block height at which activated is TBD

## Implementation

Todo: There is no implementation of this proposal at present.

Implementers should not program any of the core KZG cryptography code themselves. Instead, they should use the native library from

- [https://github.com/ethereum/c-kzg-4844](https://github.com/ethereum/c-kzg-4844)

All major Ethereum clients (execution and consensus clients) use this native library to create KZG commitments and proofs from blobs and for proof verification. The repository contains information for bindings for several higher level languages including [Java](https://github.com/ethereum/c-kzg-4844/blob/main/bindings/java/README.md). The point evaluation precompile contract implementations also rely on the same underlying library. Any deviation from this pattern can lead to severe security consequences. 

Implementers should investigate `precompute=0` [guidance](https://github.com/ethereum/c-kzg-4844/tree/main#precompute) for applications that do not use “cells” (a modern version of blob transactions).

Commitments have to be stored forever. These can perhaps be stored in their own directory like transaction receipts. Blob sidecars should also be stored in their own directory for easy deletion.

### Wrapping Blob Transactions for Broadcast

A blob is exactly 131,072 bytes, which is far too large for a simple 1-byte RLP prefix. Instead, it uses the "Large String" RLP rule (`0xb8` + length of length).

**For a single blob:**

- **Blob Size:** 131,072 bytes (Hex: `0x020000`).
- **Length of Length:** The value `0x020000` requires 3 bytes to represent.
- **Prefix:** For this large string, the prefix is therefore 183 (`0xb7`) + 3 = 186 (`0xba)`.
- Thus every blob inside the wrapped RLP starts with the 4-byte sequence: `0xba 02 00 00.`

The "Encoding Map" for the full broadcast package for a blob carrying transaction will look like

| **Item** | **RLP Type** | **Typical Prefix** | **Content** |
| --- | --- | --- | --- |
| **The Outer List** | **List** | `0xf9` or `0xfa` | Contains the 4 items below. |
| **`tx_payload_body`** | **List** | `0xf8` | The standard transaction fields. |
| **`blobs`** | **List of Strings** | `0xf9` | A list of those `0xba`-prefixed blobs. |
| **`commitments`** | **List of Strings** | `0xf8` | A list of 48-byte strings (Prefix: `0xb0`). |
| **`proofs`** | **List of Strings** | `0xf8` | A list of 48-byte strings (Prefix: `0xb0`). |

## Rationale

This proposal requires all mining nodes to download and verify blobs. Verification of blob commitments is quite fast. For instance  `verify_blob_kzg_proof` is expected to finish [under 3ms on most](https://github.com/ethereum/c-kzg-4844) systems. 

The additional computational burden of verifying blobs is not expected to be significant. However, that may not be the case with additional bandwidth requirements. This is why `RSK_MAX_BLOBS_PER_TX` is set initially to a max of 1 blob per transaction and `MAX_BLOB_GAS_PER_BLOCK` is set at a value corresponding to a maximum of 2 blobs per block.  The current proposal is intended to introduce ephemeral storage, it is not intended as a massive scaling solution for Rootstock. This low initial value can also help avoid introducing new problems related to network bandwidth - for both merge miners and regular node operators. With this bound, the network impact of blob transactions will not be too different from that of transactions with a lot of calldata, such as those used to deploy multiple contracts.

Those familiar with EIP-4844 will observe that certain parameters from that EIP are not included. This proposal does not call for dynamic pricing for blobs, so there is no “blob fee update fraction”. There is no “target” level of blobs. Instead of a BaseFeePerBlobGas, we introduce a minimum gasprice for blobs that is tied to the current `mingasPrice`. This can be modified in the future to allow updates by consensus upgrades or miner action.

EIP-4844 [explains that versioned hashes](https://eips.ethereum.org/EIPS/eip-4844#versioned-hashes--precompile-return-data) are used instead of commitments for forward compatibility.

As mentioned previously, this proposal sticks to the original EIP-4844 and does not include the newer version of peer data availability sampling (DAS). This is because of Rootstock’s merged mining consensus mechanism which relies on bitcoin mining pools. Peer DAS is intended for large scale decentralization of sharded blobs where each validator only stores a part of each blob using erasure coding schemes. This does not make sense for the relatively small set of potential mining pools in Bitcoin. Peer DAS also shifts the burden of proof generation (for blob commitments) from users to validators. There are nearly 1 million active validators in Ethereum. Some of those validators (validator “keys”) may be linked to just a single consensus client, but still is a very large set. Adopting something like that in Rootstock will imply additional computational burden. Sticking with the original EIP-4844 is a both practical and minimalist. The focus here is on enabling blob transactions, rather than on a rollup-based roadmap for scaling the Rootstock network.

## Brief comparison with RSKIP-281

Like this proposal, [RSKIP-281](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP281.md) requires a new transaction type to send “ephemeral call data” (ECD). The ECD can later be removed. Instead of the (much more complex) KZG scheme, RSKIP-281 relies on the use of hash functions to commit to ECD. Keccak is used by default. However, users can also implement custom hash functions (as solidity smart contracts).  For instance, users can implement a commitment function that makes it easy to Merkleize their data. For security (prevent manipulation), the Keccak hash is always checked. Unlike blob transactions, there are no restrictions on the size or encoding format for ECD. RSKIP-281 even allows user to set default instructions - to either store ECD permanently or remove it. The default behavior can be changed by the ECD owner though a transaction. For this, RSKIP-281 defines a new precompiled contract to handle the registration and deletion of ECD. This relies on EVM state storage to manage ephemeral data. The ECD can even be accessed by the EVM - which , if used even once, would make the future removal of that ECD impossible.

These additional choices offer users a lot of flexibility to optimize the use of ECD for their specific use cases. But these degrees of freedom also come with challenges. Since RSKIP-281 is unique to Rootstock, it will require a new transaction type that is unique to Rootstock as well. This can be a friction for wallets, and builders launching new rollups using popular “rollup-stacks” - nearly all of which offer blobs as the only alternative to transaction calldata. The potential friction is not just about the way rollups post transaction data, but may also affect dispute resolution mechanisms (designed for calldata or blobs). In Ethereum, even providers of “alternative” Data Availability solutions have adopted the blob architecture. Apart from potential integration challenges, there are also technical challenges with RSKIP-281. For one, Blockchain synchronization becomes much more complex. This is due to RSKIP-281’s use of the EVM to manage ECD lifetimes through on-chain transactions. As described in RSKIP-281, “removal” of ECD also becomes a challenge. In part because it offers users the “option” to keep or remove ECD by default.  This is in contrast to EIP-4844 (i.e. this RSKIP), where removal is simple and does not require any on-chain interaction.  Nodes are free to prune blobs (and proofs) after a certain amount of time. In Ethereum, full nodes that are NOT validators, can prune blobs aggressively or not download them at all.

## Backward compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

Unlike some hard forks, SPV light-clients will also need to be updated.

## Test cases

Todo.

## Security Considerations

Blocks containing blob transactions can take a bit longer to propagate. This is why the proposal has an initial cap of at most two blobs per block. To get an idea of the impact, it is useful to think about the estimated number of blobs on Rootstock. Consider a new L2 rollup on Rootstock with 50 thousand (L2) transactions per day. This is about 10 times the average number of daily transactions on Rootstock. This rollup will post around 60 blobs every day on Rootstock. This works out to around 1 blob every 60 Rootstock blocks (using 25 seconds as time between Rootstock blocks). The estimate is derived using a “blobs to L2 transactions ratio” [from Base](https://dune.com/seattle_brew/base-blobs) (a L2 Ethereum rollup from Coinbase). The impact of blobs from such a rollup is likely to be negligible. However, if the transaction volume from hypothetical rollups(s) hits 500 thousand transactions per day, then a blob will be posted every 6 Rootstock blocks and the impact will be significant. We should keep in mind that if rollups use calldata (or RSKIP-281’s ephemeral calldata) instead of blobs, we will face similar consequences.

The security of the underlying cryptographic toolkit, such as the native library [c-kzg-4844](https://github.com/ethereum/c-kzg-4844) and Ethereum’s “[trusted setup ceremony](https://github.com/ethereum/kzg-ceremony)” for EIP-4844 is taken as given. The repo for the native library contains links to [audits](https://github.com/ethereum/c-kzg-4844?tab=readme-ov-file#audits). There are additional security notations for [blst](https://github.com/supranational/blst?tab=readme-ov-file#status), the underlying BLS12-318 signature library used by c-kzg-4844.

## References

\[1] [EIP-4844: Shard Blob Transactions](https://eips.ethereum.org/EIPS/eip-4844)

\[2] [Specification for KZG Commitments](https://ethereum.github.io/consensus-specs/specs/deneb/polynomial-commitments/) 

\[3] [RSKIP-281: Ephemeral Calldata](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP281.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
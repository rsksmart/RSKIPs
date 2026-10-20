---
rskip: 543
title: Typed Transaction Envelope
description: Introduce Ethereum style transaction versioning
status: Draft
purpose: Sca, Usa
author: PDG (@patogallaiovlabs), SM (@smishraiov)
layer: Core
complexity: 2
created: 2026-01-05
---

## Abstract

This is a proposal to implement [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) style *Typed Transactions* in Rootstock. This will  improve Rootstock’s compatibility with the Ethereum ecosystem and make it easier to introduce other changes such as Account Abstraction and Data Availability mechanisms for rollups. Transaction Receipts in Rootstock already include a [“type field”](https://github.com/rsksmart/rskj/pull/1984) - though this is only for JSON RPC. The main departure of our proposal from EIP-2718 is the use of a Rootstock specific namespace to ensure that there are no conflicts in the type encoding between Rootstock and Ethereum.

## Motivation

The usefulness of typed transactions is evident from their adoption in Ethereum (EIP-2718). Ethereum now has several types of transactions.  Currently, in addition to legacy transactions, the other types include: type 1 for access lists (EIP-2930), type 2 for gas fee market (EIP-1559), type 3 for blob-carrying transactions intended for data availability for rollups (EIP-4844) and type 4 for a particular type of Account Abstraction (EIP-7702).

The usefulness of a transaction versioning system in Rootstock is evident from past proposals such as  [RSKIP145](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP145.md) and [RSKIP213](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP213.md). To improve compatibility with tooling and frameworks in the Ethereum ecosystem, we propose a versioning system that closely follows EIP-2718. A bit of evidence in support for such compatibility is the fact that RSKJ already has a [field for transaction type](https://github.com/rsksmart/rskj/pull/1984) (currently using just `0x0`  for legacy transactions) in Transaction Result and Transaction Receipt objects.

## Specification

Currently, the raw RLP representation of all transactions in Rootstock starts with an initial byte that is at least the Hex value `0xc0`. In RLP, any value lower than this corresponds to a “scalar” or “string”, while larger values are interpreted as “lists”.  Take an explicit example of a recent transaction from rootstock

```bash
# example of RLP encoded raw transaction using Foundry's cast tool
cast tx --rpc-url https://public-node.rsk.co "0x1e7697247535e1ed62963cf80b86ced33497c677a6a370fbacbff44ddb8b59b0"   --raw

# response
0xf8a809840393870082a832943a15461d8ae0f0fb5fa2629e9da7d66a794a6e3780b844a9059cbb0000000000000000000000002eee847bf1b899ef47222f933eb98085d0a2113b0000000000000000000000000000000000000000000000e5b8a7032e131100005fa0e39df2a7a3ddc5098c6809b038c2f05bc9809fb71cd3364c52487c3e1d0c0b28a071671984c82d34a7a6dc73bd166df08e12238d9f382fde5d584893807a3ff303
```

This starts with the first byte `f8` . In RLP, a first bye in the range `0xf8` – `0xff` points to a long list. This would be an example of a legacy transaction in Ethereum, which uses the format `rlp([nonce, gasPrice, gasLimit, to, value, data, v, r, s])`

EIP-2718 introduced a new format by concatenating a transaction type prefix with the `TransactionPayload`. The payload is treated as a bytearray (not necessarily RLP) whose semantics are to be specified in each proposal for a new transaction type. The overall encoding for transactions follows the format `tx-type || TransactionPayload`.  For example, this [recent transaction](https://etherscan.io/getRawTx?tx=0x8b3d615ed5851c470bc026e5e395cff95c2cc2a2861c5cf0eadfc64801edd8b2) in Ethereum starts with the first byte `0x02`  which denotes a Type-2 (EIP-1559) transaction. This is followed by the second byte `f8` which is similar to the RLP encoding just like our Rootstock example above. Thus far, the first four transaction types introduced in Ethereum have continued to use RLP encoding for the payload. However, EIP-2718 explicitly states that `TransactionPayload` does not have to use RLP encoding (see the [Rationale](https://eips.ethereum.org/EIPS/eip-2718#rationale) section in the EIP).

EIP-2718 restricts transaction type numbers to unsigned 8 bit numbers between `0`  and `0x7f`. We propose to follow this strictly on Rootstock as well. However, to avoid conflicts between new transaction types specifically introduced in Rootstock from those in Ethereum, we need a differentiator, or a namespace separator.

As specified (in EIP-2718's  Backwards Compatibility section), clients can differentiate between the legacy transactions and typed transactions by looking at the first byte. If it starts with a value in the range `[0, 0x7f]` then it is a new transaction type, if it starts with a value in the range `[0xc0, 0xfe]` then it is a legacy transaction type. `0xff` is not realistic for an RLP encoded transaction, so it is reserved for future use as an extension sentinel value. A transaction typically cannot exceed 128KB. Therefore, in terms of RLP list sizes, anything higher than `0xfa` is impractical (as `0xfa` is enough to store 16MB). However, to be consistent with EIP-2718, for the present, we only reserve `0xff` for future sentinel use. The range `[0xc0, 0xfe]` includes both long and short lists in RLP format. In Rootstock, REMASC transactions do not need gas or signatures and can be represented using short lists in RLP format. This is aligned with EIP-2718's range for legacy transactions (`[0xc0, 0xfe]`), therefore we can treat remasc transactions as legacy without additional special handling.


Note that the range for type values includes `0`. At present, there is no formal specification in Ethereum for a "type-0" transaction. In other words, there is no "formal" Type-0 transaction. Ethereum clients include a type field in their JSON RPC response for transaction receipts. This is just for convenience as it allows the use of the same JSON schema for receipts of different types. It should be clear that, in consensus, there is no Type-0 transaction. A transaction encoded in the EIP-2718 format with a leading `0x00` should NOT be interpreted as a legacy transaction - as noted later, the signature rules are different for EIP-2718 transactions. Implementers may wish to reject such transactions either explicitly (or they can reject them by treating them the same way as any undefined type). 

### Transactions

Following EIP-2718, a transaction is one of:

* `LegacyTransaction`: the existing Rootstock format, `rlp([nonce, gasPrice, gasLimit, to, value, data, v, r, s])`, unchanged by this proposal.
* `TransactionType || TransactionPayload`: a typed transaction. `TransactionType` is a single byte in the range `[0x01, 0x7f]`, and the format of `TransactionPayload` is defined by the RSKIP that introduces the type.
* `0x02 || rsk-tx-type || TransactionPayload`: a Rootstock-specific transaction, described in [Rootstock-specific transactions](#rootstock-specific-transactions).

The transaction hash is `keccak256` of the full encoding, including the type prefix of a typed transaction. Hashes of legacy transactions are unchanged.

A typed transaction such as `0x04 || rlp([...])` is not itself valid RLP: it is a type byte followed by an RLP-encoded payload. In block bodies and network messages, which are RLP lists of transactions, a legacy transaction is included as is (it is already an RLP list), while a typed transaction is RLP-encoded as a string, i.e. `rlp(TransactionType || TransactionPayload)`. Decoders tell them apart by whether an element is an RLP list or an RLP string.

The transactions root in the block header is the root hash of `Trie(rlp(Index) => Transaction)`, where `Transaction` is the full encoding above. For a typed transaction this is `TransactionType || TransactionPayload` itself, not its RLP string encoding used in block bodies.

### Rootstock-specific transactions

To introduce  transactions specific to Rootstock, we create a special case from the existing format by extending Ethereum’s Type-2 (EIP-1559) format. In the future, it is possible that the Rootstock community may consider introducing Ethereum’s Type 1 transactions (EIP-2930) which allows for Access Lists. This is why we do not interfere with this Type-1 format. Instead, we extend Ethereum’s Type-2 transaction format as we believe this type is highly unlikely to be implemented on Rootstock, since it (EIP-1559) requires the burning of a “base fee” component of transaction fees.

We retain the base EIP-2718 structure of `tx-type || TransactionPayload` — except when `tx-type == 02` . If the initial byte of the encoded transaction is `0x02`, then we create an additional check to see if the second byte is in the range `[0x00, 0x7f]`. This is essentially repeating the logic used to distinguish EIP-2718 typed transactions from legacy ones. If it is, then we interpret it as a Rootstock-specific transaction type. We can do this because Ethereum’s Type 2 transactions use the RLP format for the payload (which is a list), so the first byte of the payload is in the range `[0xc0, 0xff]`. If the second byte is in that range, we interpret this as a (currently unimplemented) dynamic fee (EIP-1559) transaction format from Ethereum. A second byte in the range `[0x80, 0xbf]` is invalid: it is neither a Rootstock-specific type nor the start of an RLP list. This leaves the path open for future implementation of this type on Rootstock - should there be a need for it.

Thus, the raw encoding of a Rootstock-specific transaction will be `0x02 || rsk-tx-type || TransactionPayload`  where `rsk-tx-type` is in the range `[0x00, 0x7f]`. This allows for the introduction of Rootstock-specific transaction types with no risk of conflicts with Ethereum’s type numbers.

For example, suppose we were to implement a transaction type that is identical to Ethereum’s Type-4 transactions for Account Abstraction (EIP-7702). Then we can do so by directly using the identical EIP-2718 format i.e. `0x04 || TransactionPayload`  as used in Ethereum. However, if we were to implement a different system for account abstraction (e.g. RSKIP167’s Install Code), then we could encode the transaction as `0x02 || rsk-AA-type || TransactionPayload` where the value for `rsk-AA-type`  (some value in `0-7f` ) will be specified in a RSKIP where such a transaction type is proposed.

### Classification

The following table summarizes how an encoded transaction is classified by its first byte and, when the first byte is `0x02`, its second byte. All ranges are inclusive. A defined type is valid only once the RSKIP that defines it is activated; every other type in the range `[0x01, 0x7f]` is undefined.

| First byte | Second byte | Classification |
|---|---|---|
| `0x00` | any | Invalid: there is no Type-0 transaction |
| `0x01`, `[0x03, 0x7f]` | any | Typed transaction whose `TransactionType` is the first byte |
| `0x02` | `[0x00, 0x7f]` | Rootstock-specific transaction whose `rsk-tx-type` is the second byte; reserved, and invalid until an RSKIP defines that `rsk-tx-type` |
| `0x02` | `[0x80, 0xbf]` | Invalid |
| `0x02` | `[0xc0, 0xff]` | Typed transaction with `TransactionType` `0x02` (EIP-1559 format) |
| `[0x80, 0xbf]` | any | Invalid: neither a typed transaction nor an RLP list |
| `[0xc0, 0xfe]` | any | Legacy transaction |
| `0xff` | any | Invalid: reserved for future use |

### Signatures

Just as now, transaction signatures (when required, e.g. not REMASC) should be computed over the entire encoded raw transaction including type prefixes (if any). There is no change in signatures for legacy transactions since they do not have a type prefix (as mentioned earlier, there is NO type zero transaction).

### Transaction Receipts

EIP-2718 specifies the following for receipts. The receipt root in the block header MUST be the root hash of `Trie(rlp(Index) => Receipt)` where:

* `Index` is the index in the block of the transaction this receipt is for
* `Receipt` is either `TransactionType || ReceiptPayload` or `LegacyReceipt`
* `TransactionType` is a positive unsigned 8-bit number between `0` and `0x7f` that represents the type of the transaction
    * The above restriction may have to be modified to allow for Rootstock-specific transaction types
* `LegacyReceipt` is just the format used in Rootstock, i.e. `rlp(postTxStateRLP, cumulativeGasRLP, bloomRLP, logInfoListRLP, gasUsedRLP, statusRLP)`. 
* The `TransactionType` of the receipt MUST match the `TransactionType` of the transaction with a matching `Index`. 
* `ReceiptPayload` is byte array whose interpretation is dependent on the `TransactionType` and defined in a RSKIP where such a transaction type is proposed. For example, currently there are four defined typed transaction formats in Ethereum  - and all of them use the following format for the receipt payload: `rlp(statusRLP, cumulativeGasRLP, bloomRLP, logInfoListRLP)`. If Rootstock implements any of these types, then it MUST use the same receipt payload format. On the other hand, if Rootstock implements a new transaction type, it will also need to specify the format of the receipt payload.

RSKJ includes a type field in receipts (currently hardcoded to `0x0`).  The value is stored in `TransactionResult` and `TransactionReceipt` classes as a string. Note that this field was introduced in [pull request 1984](https://github.com/rsksmart/rskj/pull/1984) primarily for JSON RPC compatibility with Ethereum. In RPC responses for receipts for legacy transactions, Ethereum clients (like geth) include a type field with value `0x0` - even though legacy transactions technically do not have a EIP-2718 type. This allows the reuse of the same JSON schema for all (currently defined) transaction types.

### Activation

Block height at which activated is TBD.

## Rationale

The main difference from EIP-2718 is the extension of the Type-2 transaction format to allow for transaction types (numbers) that are unique to Rootstock. The rationale is that, a Type-2 transaction with its dynamic fee market and burning of base fees is unlikely to be implemented in Rootstock - at least not in the way specified in EIP-1559. However, this does not preclude the implementation of a Type-2 transaction on Rootstock. Type-2 transactions are the most commonly used type on Ethereum, and they will very likely be implemented in Rootstock as well. However, internally they will have to be handled a bit differently in Rootstock.

## Backward compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated.

## Implementation

The implementation is in the RSKJ repository on the `account-abstraction` integration branch, under the consensus rule `RSKIP543`; it is not yet merged into `master`. Typed-envelope support was added in [pull request 3462](https://github.com/rsksmart/rskj/pull/3462).

## Test cases

Todo.

## References

\[1] [EIP-2718: Typed Transaction Envelope](https://eips.ethereum.org/EIPS/eip-2718)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

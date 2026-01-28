---
rskip: 543
title: Implement EIP-2718 Style Typed Transactions in Rootstock
description: Introduce Ethereum style transaction versioning
status: Draft
purpose: Sca, Usa
author: PDG (@patogallaiovlabs), SM (@smishraiov)
layer: Core
complexity: 2
created: 2026-01-05
---

| RSKIP | 543 |
| :------------ |:-------------|
| **Title** | Implement EIP-2718 Typed Transactions in Rootstock |
| **Created** | 05-JAN-2026 |
| **Author** | Patricio Gallardo, Shreemoy Mishra |
| **Purpose** | Sca, Usa |
| **Layer** | Core |
| **Complexity** | 2 |
| **Status** | Draft |

## Abstract

This is a proposal to implement [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) style *Typed Transactions* in Rootstock. This will  improve Rootstock’s compatibility with the Ethereum ecosystem and make it easier to introduce other changes such as Account Abstraction and Data Availability mechanisms for rollups. Transaction Receipts in Rootstock already include a [“type field”](https://github.com/rsksmart/rskj/pull/1984) (consistent with EIP-2718). The main departure of our proposal from EIP-2718 is the use of a Rootstock specific namespace to ensure there are no conflicts in the type encoding between Rootstock and Ethereum.

## Motivation

The usefulness of typed transactions is evident from their adoption in Ethereum (EIP-2718). Ethereum now has several types of transactions.  Currently, in addition to legacy transactions, the other types include: type 1 for access lists (EIP-2930), type 2 for gas fee market (EIP-1559), type 3 for blob-carrying transactions intended for data availability for rollups (EIP-4844) and type 4 for a particular type of Account Abstraction (EIP-7702).

The usefulness of a transaction versioning system in Rootstock is evident from past proposals such as  [RSKIP145](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP145.md) and [RSKIP213](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP213.md). To improve compatibility with tooling and frameworks in the Ethereum ecosystem, we propose a versioning system that closely follows EIP-2718. A bit of evidence in support for such compatibility is the fact that RSKJ already has a [field for transaction type](https://github.com/rsksmart/rskj/pull/1984) (currently using just `0x0`  for legacy transactions) in Transaction Result and Transaction Receipt objects.

## Specification

Currently, the raw RLP representation of all transactions in Rootstock starts with an initial byte that exceeds the Hex value `0xc0`. In RLP, any value lower than this corresponds to a “scalar” or “string”, while larger values are interpreted as “lists”.  Take an explicit example of a recent transaction from rootstock

```bash
# example of RLP encoded raw transaction
cast tx --rpc-url https://public-node.rsk.co "0x1e7697247535e1ed62963cf80b86ced33497c677a6a370fbacbff44ddb8b59b0"   --raw

# response
0xf8a809840393870082a832943a15461d8ae0f0fb5fa2629e9da7d66a794a6e3780b844a9059cbb0000000000000000000000002eee847bf1b899ef47222f933eb98085d0a2113b0000000000000000000000000000000000000000000000e5b8a7032e131100005fa0e39df2a7a3ddc5098c6809b038c2f05bc9809fb71cd3364c52487c3e1d0c0b28a071671984c82d34a7a6dc73bd166df08e12238d9f382fde5d584893807a3ff303
```

This starts with the first byte `f8` . In RLP, a first bye in the range `0xf8` – `0xff` points to a long list. This would be an example of a legacy transaction in Ethereum, which uses the format `rlp([nonce, gasPrice, gasLimit, to, value, data, v, r, s])`

EIP-2718 introduced a new format by concatenating a transaction type prefix with the `TransactionPayload`. The payload is treated as a bytearray (not necessarily RLP) whose semantics are to be specified in each proposal for a new transaction type. The overall encoding for transactions follows the format `tx-type || TransactionPayload`.  For example, this [recent transaction](https://etherscan.io/getRawTx?tx=0x8b3d615ed5851c470bc026e5e395cff95c2cc2a2861c5cf0eadfc64801edd8b2) in Ethereum starts with the first byte `0x02`  which denotes a Type-2 (EIP-1559) transaction. This is followed by the second byte `f8` which is similar to the RLP encoding just like our Rootstock example above. Thus far, the first four transaction types introduced in Ethereum have continued to use RLP encoding for the payload. However, EIP-2718 explicitly states that `TransactionPayload` does not have to use RLP encoding (see the [Rationale](https://eips.ethereum.org/EIPS/eip-2718#rationale) section in the EIP).

EIP-2718 restricts transaction type numbers to unsigned 8 bit numbers between `0`  and `0x7f`. We propose to follow this strictly on Rootstock as well. However, to avoid conflicts between new transaction types specifically introduced in Rootstock from those in Ethereum, we need a differentiator, or a namespace separator.

## Rootstock-specific transactions

To introduce  transactions specific to Rootstock, we create a special case from the existing format by extending Ethereum’s Type-2 (EIP-1559) format. In the future, it is possible that the Rootstock community may consider introducing Ethereum’s Type 1 transactions (EIP-2930) which allows for Access Lists. This is why we do not interfere with this Type-1 format. Instead, we extend Ethereum’s Type-2 transaction format as we believe this type is highly unlikely to be implemented on Rootstock, since it (EIP-1559) requires the burning of a “base fee” component of transaction fees.

We retain the base EIP-2718 structure of `tx-type || TransactionPayload` — except when `tx-type == 02` . If the initial byte of the encoded transaction is `0x02`, then we create an additional check to see if the second byte is less than `7f`. This is essentially repeating the logic used to distinguish EIP-2718 typed transactions from legacy ones. If the second byte is indeed less than or equal to `7f`, then we interpret it as a Rootstock-specific transaction type. We can do this (use `7f`) because Ethereum’s Type 2 transactions use the RLP format for the payload (which is a list), so the first byte of the payload cannot be less than `7f`. If the second byte is larger than `7f` we interpret this as a (currently unimplemented) dynamic fee (EIP-1559) transaction format from Ethereum. This leaves the path open for future implementation of this type on Rootstock - should there be a need for it.

Thus, the raw encoding of a Rootstock-specific transaction will be `0x02 || rsk-tx-type || TransactionPayload`  where `rsk-tx-type` is between `0`  and `0x7f`. This allows for the introduction of Rootstock-specific transaction types with no risk of conflicts with Ethereum’s type numbers.

For example, suppose we were to implement a transaction type that is identical to Ethereum’s Type-4 transactions for Account Abstraction (EIP-7702). Then we can do so by directly using the identical EIP-2718 format i.e. `0x04 || TransactionPayload`  as used in Ethereum. However, if we were to implement a different system for account abstraction (e.g. RSKIP167’s Install Code), then we could encode the transaction as `0x02 || rsk-AA-type || TransactionPayload` where the value for `rsk-AA-type`  (some value in `0-7f` ) will be specified in a RSKIP where such a transaction type is proposed.

### Signatures

Just as now, transaction signatures (when required, e.g. not Remasc) should be computed over the entire encoded raw transaction including type prefixes.

### Transaction Receipts

EIP-2718 specifies the following for receipts. The receipt root in the block header MUST be the root hash of `Trie(rlp(Index) => Receipt)` where:

* `Index` is the index in the block of the transaction this receipt is for
* `Receipt` is either `TransactionType || ReceiptPayload` or `LegacyReceipt`
* `TransactionType` is a positive unsigned 8-bit number between `0` and `0x7f` that represents the type of the transaction
    * The above restriction may have to be modified to allow for Rootstock-specific transaction types
* `ReceiptPayload` is byte array whose interpretation is dependent on the `TransactionType` and defined in a RSKIP where such a transaction type is proposed.
* `LegacyReceipt` is  `rlp([status, cumulativeGasUsed, logsBloom, logs])`

The `TransactionType` of the receipt MUST match the `TransactionType` of the transaction with a matching `Index`.


Currently, RSKJ already includes a type field in receipts (currently set to `0x0`).  The value is stored in `TransactionResult` and `TransactionReceipt` classes as a string.

### Activation

Block height at which activated is TBD

## Implementation

There is no implementation of this proposal at present

## Rationale

The main difference from EIP-2718 is the extension of the Type 2 transaction format to allow for transaction types (numbers) that are unique to Rootstock. The rationale for this choice is included in the specification.  

The path mentioned here is included merely as an illustrative example, and is not meant to be a stringent component of the specification.

## Backward compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated.

## Test cases

Todo.

## References

\[1] [EIP-2718: Typed Transaction Envelope](https://eips.ethereum.org/EIPS/eip-2718)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
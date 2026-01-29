---
rskip: 546
title: Implement Transactions and Receipts encoding following Ethereum's Type 1 and Type 2 Envelope formats
created: 27-JAN-2026
author: PDG (@patogallaiovlabs), SM (@smishraiov)
purpose: Usa
layer: Core 
complexity: 1
status: Draft
description: 
---

|RSKIP          | 546          |
| :------------ |:-------------|
|**Title**      | Implement Transactions and Receipts following Ethereum's Type 1 and Type 2 Envelope formats |
|**Created**    | 27-JAN-2026 |
|**Author**     | Patricio Gallardo, Shreemoy Mishra |
|**Purpose**    | Usa |
|**Layer**      | Core |
|**Complexity** | 1 |
|**Status**     | Draft |

## Abstract

This is a proposal to support Ethereum's Type 1 and Type 2 formatted transactions in Rootstock. Apart from the new (EIP-2718 based) encodings for transactions and receipts, this proposal does not introduce any other changes to Rootstock's consensus rules. In particular, neither "Access Lists" (EIP-2930) nor "Dynamic GasPrice" parameters (EIP-1559) will be implemented. Transactions encoded in these new formats will be executed in the same way as legacy transactions. Access Lists, if provided, will not be used in the EVM, but will result in additional gas fees  as specified below. For Type 2 transactions, the lower of the two EIP-1559 gas price fields will be used as the transaction’s effective gas price.

## Motivation

At present, Rootstock only supports legacy transactions. Since the adoption of EIP-2718 in Ethereum, there are now four transaction types based on that format. In Ethereum, wallets and applications have largely migrated away from legacy transactions. 

[RSKIP-543](https://github.com/rsksmart/RSKIPs/pull/543/) is a proposal to introduce EIP-2718 in Rootstock. The current RSKIP builds on that and proposes to support Type 1 and Type 2 transaction formats used in Ethereum. As noted in the abstract, these changes only introduce the encodings, while keeping the same semantics as legacy transactions. While these changes are superficial, they will be useful for Account Abstraction (EIP-7702, [RSKIP-545](https://github.com/rsksmart/RSKIPs/pull/545/)) in Rootstock. Following EIP-7702, the "Type 4" transaction proposed in RSKIP-545 contains the fields introduced successively in Type 1 and Type 2 transactions. Introducing these new formats will also be a convenience for developers integrating wallets or porting applications from other EVM chains to Rootstock. This can help improve Rootstock's compatibility with Ethereum's developer ecosystem.

## Specification

### Prerequisites

This RSKIP requires [RSKIP-543](https://github.com/rsksmart/RSKIPs/pull/543/) which introduces EIP-2718 typed transaction envelope format in Rootstock.


### Type 1 Transactions

The encoding of Type 1 transactions is intended to match EIP-2930, but only the transaction format is introduced. Access Lists are not supported in Rootstock and will be ignored.

The EIP-2718 `TransactionPayload` for this transaction is `rlp([chainId, nonce, gasPrice, gasLimit, to, value, data, accessList, signatureYParity, signatureR, signatureS])`.

The `signatureYParity`, `signatureR`, `signatureS` elements of this transaction represent a secp256k1 signature over `keccak256(0x01 || rlp([chainId, nonce, gasPrice, gasLimit, to, value, data, accessList]))`.


After validating the signature, and ignoring the `accessList` field, the transaction can be processed in the same way as a legacy transaction. See the "Transaction Normalization and Additional Gas" section below.

The EIP-2718 `ReceiptPayload` for this transaction is `rlp([status, cumulativeGasUsed, logsBloom, logs])`. Thus, the full receipt will be `01 || ReceiptPayload`.

### Type 2 Transactions

The EIP-2718 TransactionPayload for this transaction type is `rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, amount, data, access_list, signature_y_parity, signature_r, signature_s])`.

The `signature_y_parity`, `signature_r`, `signature_s` elements of this transaction represent a secp256k1 signature over `keccak256(0x02 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, amount, data, access_list]))`.

In Ethereum, under EIP-1559 rules, a transaction’s effective gasprice is computed based on a BaseFee (which is burnt). Block producers only get the “priority fee” component. The priority fee for a transaction is computed as below (which is copied from EIP-1559).

```
# From https://eips.ethereum.org/EIPS/eip-1559
# priority fee is capped because the base fee is filled first

priority_fee_per_gas = min(transaction.max_priority_fee_per_gas, transaction.max_fee_per_gas - block.base_fee_per_gas)

#Note: in Rootstock, there is no base_fee_per_gas
```

Unlike Ethereum, Rootstock does not support the dynamic fee market of EIP-1559. The “Base Fee” is effectively zero. Therefore, in Rootstock,  one of `maxPriorityFeePerGas` or `maxFeePerGas` fields, whichever is lower, will become the effective gas price for this transaction. This is what the `GASPRICE` opcode (`0x3a`) will return in the context of this transaction.

The EIP-2718 `ReceiptPayload` for this transaction is `rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`. Thus, the full receipt will be `02 || ReceiptPayload`.


### Transaction Normalization and Additional Gas

Recall the legacy transaction format: `rlp([chainId, nonce, gasPrice, gasLimit, to, value, data, signatureYParity, signatureR, signatureS])`. The new fields in Type 1 and Type 2 transactions are required to validate the signature. After signature validation the new transaction types can be *normalized* to the legacy format by ignoring the `accessList` field and setting the `gasPrice = min(maxPriorityFeePerGas, maxFeePerGas)`. After which they can be executed in the same way as legacy transactions.

In Ethereum, the `accessList` field is expected to be of type `[[{20 bytes}, [{32 bytes}...]]...]`, where `...` means “zero or more of the thing to the left”. However, this detail is not relevant for Rootstock since Access Lists are not supported.

Depending on the type (1 or 2), the `accessList` will be the 8th or 9th element of the RLP encoded transaction payload. If the list is empty, it will be encoded as `0xc0`. In any case, even though we retain the raw bytes for verifying the signature we do not need to fully deserialize the `accessList` field. If `accessList` is not empty, we only need the length in bytes of the data without deserializing the list elements. This is useful for calculating the additional gas cost for the access list.

**Additional Gas for Access Lists**: Even though Access Lists will not be used, the transaction sender will be charged an additional amount of gas to cover bandwidth costs and to reduce spam. In Ethereum the additional gas cost for access list items is computed using a formula that works out to around 120 gas/byte for "addresses" and 60 gas/byte for "storage keys". In Rootstock, we propose charging `80 gas/byte` for the accessList field. This additional gas can be collected at the same time as for calldata which is based on the transaction's `data` field.


## Rationale

The additional gas cost of access list is set somewhat arbitrarily at 80 gas/byte. This is in the range of 60-120 gas/byte used in Ethereum. Since access lists are not supported in Rootstock, we expect this field to be empty. However, to deter abuse or spam, the additional gas cost has to be meaningful.

## Backwards compatibility

This change requires a network upgrade. All full nodes must be upgraded.

## References

[1] https://eips.ethereum.org/EIPS/eip-1559

[2] https://eips.ethereum.org/EIPS/eip-2718

[3] https://eips.ethereum.org/EIPS/eip-2930

[4] https://eips.ethereum.org/EIPS/eip-7702

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

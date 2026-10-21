---
rskip: 546
title: Implement EVM's Type 1 and Type 2 Envelope formats
description: Support Ethereum's Type 1 (EIP-2930) and Type 2 (EIP-1559) transaction encodings
status: Draft
purpose: Usa
author: PDG (@patogallaiovlabs), SM (@smishraiov)
layer: Core
complexity: 1
created: 2026-01-27
---

## Abstract

This is a proposal to support Ethereum's Type 1 and Type 2 formatted transactions in Rootstock. Apart from the new (EIP-2718 based) encodings for transactions and receipts, this proposal does not introduce any other changes to Rootstock's consensus rules. In particular, it does not implement the execution semantics of access lists (EIP-2930) or the dynamic fee market (EIP-1559). Transactions encoded in these new formats will be executed in the same way as legacy transactions. An access list, if provided, has no effect on execution under this RSKIP, but results in additional gas fees as specified below. For Type 2 transactions, `maxPriorityFeePerGas` is used as the transaction’s effective gas price.

## Motivation

At present, Rootstock only supports legacy transactions. Since the adoption of EIP-2718 in Ethereum, there are now four transaction types based on that format. In Ethereum, wallets and applications have largely migrated away from legacy transactions. 

[RSKIP-543](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP543.md) is a proposal to introduce EIP-2718 in Rootstock. The current RSKIP builds on that and proposes to support Type 1 and Type 2 transaction formats used in Ethereum. As noted in the abstract, these changes only introduce the encodings, while keeping the same semantics as legacy transactions. While these changes are superficial, they will be useful to enable Account Abstraction (EIP-7702, [RSKIP-545](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP545.md)) in Rootstock. Following EIP-7702, the "Type 4" transaction proposed in RSKIP-545 contains the fields introduced successively in Type 1 and Type 2 transactions. Introducing these new formats will also be a convenience for developers integrating wallets or porting applications from other EVM chains to Rootstock. This can help improve Rootstock's compatibility with Ethereum's developer ecosystem.

## Specification

### Prerequisites

This RSKIP requires [RSKIP-543](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP543.md) which introduces EIP-2718 typed transaction envelope format in Rootstock. Under RSKIP-543's classification rules, a Type 2 transaction is one whose first byte is `0x02` and whose second byte is in the range `[0xc0, 0xff]` (the start of an RLP list). Note that `0x02` followed by a byte in `[0x00, 0x7f]` is the Rootstock-specific namespace and is not a Type 2 transaction.


### Type 1 Transactions

The encoding of Type 1 transactions is intended to match EIP-2930, but only the transaction format is introduced. This RSKIP introduces the access list only as part of the encoding: it is validated and charged for as described below, but has no effect on execution. Access-list semantics may be specified in a future RSKIP.

The EIP-2718 `TransactionPayload` for this transaction is `rlp([chainId, nonce, gasPrice, gasLimit, to, value, data, accessList, signatureYParity, signatureR, signatureS])`.

The `signatureYParity`, `signatureR`, `signatureS` elements of this transaction represent a secp256k1 signature over `keccak256(0x01 || rlp([chainId, nonce, gasPrice, gasLimit, to, value, data, accessList]))`.


After the checks in [Validity](#validity), and ignoring the `accessList` field, the transaction can be processed in the same way as a legacy transaction. See the "Transaction Normalization and Additional Gas" section below.

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

Unlike Ethereum, Rootstock does not have the dynamic fee market of EIP-1559. The “Base Fee” is effectively zero. As in EIP-1559, nodes MUST reject a Type 2 transaction with `maxPriorityFeePerGas > maxFeePerGas`. With a zero base fee, EIP-1559's `effective_gas_price` is then exactly `maxPriorityFeePerGas`, and this is the gas price the transaction pays; `maxFeePerGas` only serves as the cap required by that rule. This is what the `GASPRICE` opcode (`0x3a`) returns in the context of this transaction, and, as for legacy transactions, it MUST be at least the block's minimum gas price.

The sender's balance is checked against the normalized transaction, as for a legacy transaction: it must cover `value + gasLimit * effective gas price`. EIP-1559's additional requirement that the balance cover `gasLimit * maxFeePerGas` does not apply.

The EIP-2718 `ReceiptPayload` for this transaction is `rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`. Thus, the full receipt will be `02 || ReceiptPayload`.

Both receipt payloads use the typed receipt format of RSKIP-543, which differs from Rootstock's legacy receipt `rlp([postTxState, cumulativeGas, bloom, logs, gasUsed, status])`: it omits `postTxState` and `gasUsed`, as described in RSKIP-543. The `status` field is encoded as in the legacy receipt (`0x01` for success, the empty string for failure). Receipts of legacy transactions are unchanged.


### Inconsistencies in Field Names

Note that the names of several fields differ across the different transaction types. For example, `to` and `value` fields from legacy transactions (and type 1) become `destination` and `amount` respectively in Type 2 transactions. There are also inconsistencies in the signature fields: legacy transactions carry `v`, `r`, `s`, while Type 1 uses `signatureYParity`, `signatureR`, `signatureS` and Type 2 uses `signature_y_parity`, `signature_r`, `signature_s`. Unlike the legacy `v`, the y-parity of a typed transaction is only `0` or `1` and does not encode the chain ID, which is a separate field. Minor differences show up in the names of receipt fields as well, such as `cumulativeGasUsed` vs `cumulative_transaction_gas_used`. These inconsistencies are a result of following the notation and text from the original EIPs. Implementers should use field names consistently.

### Validity

A Type 1 or Type 2 transaction is valid only if all of the following hold. Nodes MUST reject a transaction that fails any of them, as specified in RSKIP-543 for invalid transactions.

* **Shape.** The `TransactionPayload` is an RLP list with exactly 11 elements (Type 1) or 12 elements (Type 2). Every element is an RLP string, except the access list, which is an RLP list.
* **Canonical encoding.** The payload is canonically RLP-encoded: re-encoding the decoded list yields the same bytes.
* **Integers.** `chainId`, `nonce`, `gasPrice`, `maxPriorityFeePerGas`, `maxFeePerGas`, `gasLimit` and `value` are at most 32 bytes long and have no leading zero bytes; zero is encoded as the empty string.
* **Fee caps (Type 2).** `maxPriorityFeePerGas <= maxFeePerGas`.
* **Destination.** `to` is either empty (contract creation) or exactly 20 bytes.
* **Chain ID.** `chainId` is nonzero and equal to the chain ID of the network, as required by RSKIP-543.
* **Signature.** The y-parity is `0` or `1` (with `0` encoded as the empty string). `r` and `s` are at most 32 bytes long with no leading zero bytes, `r` and `s` are in the range `[1, secp256k1n - 1]`, and `s < secp256k1n / 2` (integer division), the same low-`s` rule Rootstock applies to legacy transactions.

### Transaction Normalization and Additional Gas

Recall the legacy transaction format: `rlp([nonce, gasPrice, gasLimit, to, value, data, v, r, s])`, where `v` carries the signature's y-parity and, as in EIP-155, optionally the chain ID. The new fields in Type 1 and Type 2 transactions are required to validate the signature. After signature validation the new transaction types can be *normalized* to the legacy format by ignoring the `accessList` field and, for Type 2, setting `gasPrice = maxPriorityFeePerGas` (the effective gas price defined above). After which they can be executed in the same way as legacy transactions.

The `accessList` MUST have the same shape as in Ethereum, `[[{20 bytes}, [{32 bytes}...]]...]`, where `...` means “zero or more of the thing to the left”: an RLP list of zero or more entries, each an RLP list of exactly two elements, a 20-byte address string and an RLP list of zero or more 32-byte storage-key strings. A transaction whose access list does not have this shape is invalid.

Depending on the type (1 or 2), the `accessList` is the 8th or 9th element of the RLP encoded transaction payload. An empty access list is encoded as `0xc0`. Beyond checking its shape, nodes do not interpret the access list; the additional gas below depends only on its encoded length.

**Additional Gas for Access Lists**: Even though the access list has no effect on execution under this RSKIP, the transaction sender will be charged an additional amount of gas to cover bandwidth costs and to reduce spam. In Ethereum the additional gas cost for access list items is computed using a formula that works out to around 120 gas/byte for "addresses" and 60 gas/byte for "storage keys". In Rootstock, the additional gas is `0` for an empty access list (`0xc0`), and otherwise `80 * len(accessList)`, where `len(accessList)` is the length in bytes of the access list's complete RLP encoding, including its list header. This additional gas MUST be added to the transaction's intrinsic gas, alongside the cost of its `data`. A transaction whose `gasLimit` is below its intrinsic gas is invalid, as for any other transaction.

### Activation

This RSKIP is proposed for the Cardamom network upgrade ([RSKIP-694](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP694.md)). Activation block heights are TBD.

This RSKIP MUST NOT activate before RSKIP-543. Type 1 and Type 2 transactions are valid only when both RSKIP-543 and this RSKIP are active; before that, nodes MUST reject them.

## Rationale

The additional gas cost of access list is set somewhat arbitrarily at 80 gas/byte. This is in the range of 60-120 gas/byte used in Ethereum. Since access lists have no effect on execution under this RSKIP, we expect this field to be empty in practice, and an empty list is free so that ordinary transactions pay nothing extra. However, to deter abuse or spam, the additional gas cost has to be meaningful.

## Backwards compatibility

This change is a hard fork and requires a network upgrade. All full nodes must be upgraded.

## Implementation

The implementation is in the RSKJ repository under the consensus rule `RSKIP546`, in [pull request 3558](https://github.com/rsksmart/rskj/pull/3558), which is not yet merged.

## Test cases

The vectors below are taken from the RSKJ encoder test vectors. They use chain ID `33` (`0x21`) and placeholder signature values (`r = 0x11…11`, `s = 0x22…22`, y-parity `1`), so they pin the encodings and signing hashes but not signature recovery. All three share these fields: `nonce = 1`, `gasLimit = 21000` (`0x5208`), `to = 0x1234567890123456789012345678901234567890`, `value = 10^18` (`0x0de0b6b3a7640000`), and empty `data`.

### Type 1

`gasPrice = 10^9` (`0x3b9aca00`), empty access list.

Signing preimage, `0x01 || rlp([chainId, nonce, gasPrice, gasLimit, to, value, data, accessList])`:

```
01ea2101843b9aca00825208941234567890123456789012345678901234567890880de0b6b3a764000080c0
```

Signing hash: `0x05fac67ac7f82d280d881e905c6ea5738e580128de006a56f05baefd5126c23a`

Signed envelope:

```
01f86d2101843b9aca00825208941234567890123456789012345678901234567890880de0b6b3a764000080c001a01111111111111111111111111111111111111111111111111111111111111111a02222222222222222222222222222222222222222222222222222222222222222
```

Normalized legacy form: `nonce = 1`, `gasPrice = 10^9`, `gasLimit = 21000`, `to`, `value` and `data` as above. The empty access list adds no gas.

### Type 2

`maxPriorityFeePerGas = 10^9` (`0x3b9aca00`), `maxFeePerGas = 2 * 10^9` (`0x77359400`), empty access list.

Signing preimage, `0x02 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, amount, data, access_list])`:

```
02ef2101843b9aca008477359400825208941234567890123456789012345678901234567890880de0b6b3a764000080c0
```

Signing hash: `0xda29f69ce83037cdfc5a06235bd6510b9dc72727026220b6105254aa7e74c9f4`

Signed envelope:

```
02f8722101843b9aca008477359400825208941234567890123456789012345678901234567890880de0b6b3a764000080c001a01111111111111111111111111111111111111111111111111111111111111111a02222222222222222222222222222222222222222222222222222222222222222
```

Normalized legacy form: `nonce = 1`, `gasPrice = maxPriorityFeePerGas = 10^9`, `gasLimit = 21000`, `to`, `value` and `data` as above. `maxFeePerGas` is not charged. The empty access list adds no gas.

### Type 1 with a non-empty access list

As the Type 1 vector, with the access list `[[0x00…00 (20 bytes), [0x00…00 (32 bytes)]]]`. Its RLP encoding is 58 bytes long (`f838…`), so the additional gas is `80 * 58 = 4640`.

Signing preimage:

```
01f8632101843b9aca00825208941234567890123456789012345678901234567890880de0b6b3a764000080f838f7940000000000000000000000000000000000000000e1a00000000000000000000000000000000000000000000000000000000000000000
```

Signing hash: `0x59ee291b7f17e48e400006ae35fc2b3114a3fd4a42490f2c60fd047a2eac826c`

Signed envelope:

```
01f8a62101843b9aca00825208941234567890123456789012345678901234567890880de0b6b3a764000080f838f7940000000000000000000000000000000000000000e1a0000000000000000000000000000000000000000000000000000000000000000001a01111111111111111111111111111111111111111111111111111111111111111a02222222222222222222222222222222222222222222222222222222222222222
```

### Access list above the gas limit

As the Type 1 vector, with `gasLimit = 21000` and the access list `[[0x1234567890123456789012345678901234567890, [0x00…00, 0x00…01, 0x00…02]]]` (one address, three 32-byte storage keys). Its RLP encoding is 126 bytes long:

```
f87cf87a941234567890123456789012345678901234567890f863a00000000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000001a00000000000000000000000000000000000000000000000000000000000000002
```

The additional gas is `80 * 126 = 10080`, so the intrinsic gas is `21000 + 10080 = 31080`. This exceeds the gas limit, and the transaction is invalid.

### Chain ID

| Case | Expected |
| --- | --- |
| `chainId = 200`, encoded as `81c8` (a single-byte string, since `0xc8 > 0x7f`) | Valid encoding. For example, the Type 2 envelope `02f87381c801843b9aca008477359400…` from the RSKJ encoder test vectors; valid only on a network whose chain ID is `200` |
| `chainId = 286`, encoded as `82011e` | Invalid. It MUST NOT be truncated to a single byte, which would give `30`, the chain ID of Rootstock Mainnet |
| `chainId = 0`, encoded as `80` | Invalid |

### Canonical encoding

| Case | Expected |
| --- | --- |
| y-parity `0`, encoded as `80`, as in the Type 2 envelope `02f8722101843b9aca008477359400825208941234567890123456789012345678901234567890880de0b6b3a764000080c080a0…` | Valid encoding |
| y-parity `0` encoded as `00` | Invalid (leading zero byte) |
| y-parity `2`, `27` (`1b`) or `255` (`ff`) | Invalid |
| y-parity encoded as the two bytes `0100` | Invalid |
| `nonce = 5` encoded in long form as `8105` instead of `05` | Invalid (not canonical) |
| Empty access list encoded in long form as `f800` instead of `c0` | Invalid (not canonical) |

### Access-list shape

Each of the following access lists makes the transaction invalid:

| Access list | Reason |
| --- | --- |
| `80` | An RLP string, not a list |
| `81c0` | An RLP string whose content is `c0`, not a list |
| `[[<address encoded as a 20-element RLP list>, []]]` | The address must be a 20-byte RLP string |
| `[[<20-byte address>, [<32-element RLP list>]]]` | Each storage key must be a 32-byte RLP string |
| `c401` | Truncated: the list header announces 4 bytes but only 1 follows |

## References

[1] https://eips.ethereum.org/EIPS/eip-1559

[2] https://eips.ethereum.org/EIPS/eip-2718

[3] https://eips.ethereum.org/EIPS/eip-2930

[4] https://eips.ethereum.org/EIPS/eip-7702

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

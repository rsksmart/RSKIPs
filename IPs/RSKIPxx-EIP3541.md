---
rskip: TBD
title: Reject new contract code starting with the `0xEF` byte
description: Implements EIP-3541
status: Draft
purpose: Usa
author: PDG (@patogallaiovlabs), SM (@smishraiov)
layer: Core
complexity: 1
created: 2026-01-05
---

| RSKIP | TBD |
| --- | --- |
| **Title** | Reject new contract code starting with the `0xEF` byte|
| **Created** | 05-JAN-2026 |
| **Author** | Patricio Gallardo, Shreemoy Mishra |
| **Purpose** | Usa |
| **Layer** | Core |
| **Complexity** | 1 |
| **Status** | Draft |

## Abstract

Implement [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541) to disallow deployment of new contract starting with the `0xEF` byte. Code already existing in the account trie starting with `0xEF` byte is not affected semantically by this change.

## Motivation

Our motivation deviates from the original intent of EIP-3541. In the original EIP-3541, `0xEF` was meant to represent the first byte of a “magic sequence” for later use in the development of the EVM Object Format (EOF). The authors of [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) (for Account Abstraction) repurposed the use of this sequence format to denote EOA’s that choose to *inject code* into their account space. This is why EIP-3541 was a pre-requisite for EIP-7702. Thus, our primary motivation in this proposal is to pre-emptively satisfy this pre-requisite for any future proposal to implement EIP-7702 on Rootstock. This RSKIP is intended to follow EIP-3541 very closely, and large sections of the text are replicated from that proposal.

## Specification

Once this RKSIP is adopted, new contract creation (via create transaction, or via `CREATE` or `CREATE2` instructions) results in an exceptional abort if the deployed *code*’s first byte is `0xEF`.

The *initcode* is the code executed in the context of the *create* transaction, `CREATE`,  or `CREATE2` instructions. The *initcode* returns *code* (via the `RETURN` instruction), which is inserted into the account. 

The opcode `0xEF` is currently an undefined instruction in RSKJ’s. It pops no stack items and pushes no stack items, and it causes an exceptional abort when executed. This means *initcode* or already deployed *code* starting with this instruction will continue to abort execution.

The exceptional abort due to *code* starting with `0xEF` behaves exactly the same as any other exceptional abort that can occur during *initcode* execution, i.e. in case of abort all gas provided to a `CREATE*` or create transaction is consumed.

### Activation

Todo: Block height at which activated is TBD

## Implementation

Todo: There is no implementation of this proposal at present

## Rationale

Contracts using unassigned opcodes are generally understood to be at risk of changing semantics. Hence using the unassigned `0xEF` should have lesser effects, than choosing an assigned opcode, such as `0xFD` (`REVERT`), `0xFE` (`INVALID`), or `0xFF` (`SELFDESTRUCT`). Arguably while such contracts may not be very useful, they are still using valid opcodes.

## Backward compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated.

## **Test Cases**

Each test case below may be executed in 3 different contexts:

- create transaction (no account code)
- `CREATE`, with account code: `0x6000356000523660006000f0151560165760006000fd5b` (Yul code: `mstore(0, calldataload(0)) if iszero(create(0, 0, calldatasize())) { revert(0, 0) }`),
- `CREATE2`, with account code: `0x60003560005260003660006000f5151560185760006000fd5b` (Yul code: `mstore(0, calldataload(0)) if iszero(create2(0, 0, calldatasize(), 0)) { revert(0, 0) }`)

| Case | Calldata | Expected result |
| --- | --- | --- |
| deploy one byte `ef` | `0x60ef60005360016000f3` | new contract not deployed, transaction fails |
| deploy two bytes `ef00` | `0x60ef60005360026000f3` | new contract not deployed, transaction fails |
| deploy three bytes `ef0000` | `0x60ef60005360036000f3` | new contract not deployed, transaction fails |
| deploy 32 bytes `ef00...00` | `0x60ef60005360206000f3` | new contract not deployed, transaction fails |
| deploy one byte `fe` | `0x60fe60005360016000f3` | new contract deployed, transaction succeeds |

## References

\[1] [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541)

\[2] [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
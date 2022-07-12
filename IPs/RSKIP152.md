---
rskip: 152
title: CHAINID Opcode
description: 
status: Adopted
purpose: Sec
author: SMS (@sebastians)
layer: Core
complexity: 1
created: 2019-11-19
---
# CHAINID Opcode

|RSKIP          |152           |
| :------------ |:-------------|
|**Title**      |CHAINID Opcode |
|**Created**    |19-NOV-19 |
|**Author**     |SMS |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

The purpose of this RSKIP is to describe the new opcode: CHAINID. This opcode returns the RSK chain identification number.

## Motivation

This RSKIP proposes the addition of the CHAINID opcode to prevent replay attacks between different chains, specially for replay protection of ECDSA-signed messages in smart contracts.

## Specification

Adds a new opcode CHAINID at 0x46, which does not consume any input argument from the stack. It pushes the current chain ID onto the stack. Chain ID is a 256-bit value. The operation costs 2 gas to execute. The values for the RSK blockchain are the following, according to https://chainid.network/:
	
	- Mainnet: 30
	- Testnet: 31

While the numbers 32 and 33 are assigned to the Devnet and Regtest, respectively.

## Rationale

The following is a copy of the decisions made by the Ethereum developers in order to implement the opcode:

>The current approach proposed by EIP-712 is to specify the chain ID at compile time. Using this approach will result in problems after a hardfork, as well as human error that may lead to loss of funds or replay attacks on signed messages. By adding the proposed opcode it will be possible to access the current chain ID and validate signatures based on that.
>
>Currently, there is no specification for how chain ID is set for a particular network, relying on choices made manually by the client implementers and the chain community. There is a potential scenario where, during a "contentious split" over a divisive issue, a community using a particular value of chain ID will make a decision to split into two such chains. When this scenario occurs, it will be unsafe to maintain chain ID to the same value on both chains, as chain ID is used for replay protection for in-protocol transactions (per EIP-155), as well as for L2 and "meta-transaction" use cases (per EIP-712 as enabled by this proposal). There are two potential resolutions in this scenario under the current process: 1) one chain decides to modify their value of chain ID (while the other keeps it), or 2) both chains decide to modify their value of chain ID.
>
>In order to mitigate this situation, users of the proposed CHAINID opcode must ensure that their application can handle a potential update to the value of chain ID during their usage of their application in case this does occur, if required for the continued use of the application. A Trustless Oracle that logs the timestamp when a change is made to chain ID can be implemented either as an application-level feature inside the application contract system, or referenced as a globally standard contract. Failure to provide a mitigation for this scenario could lead to a sudden loss of legitimacy of previously signed off-chain messages, which could be an issue during settlement periods and other longer-term verification events for these types of messages. Not all applications of this opcode may need mitigations to handle this scenario, but developers should provide reasoning on a case-by-case basis.
>
>One example of a scenario where it would not make sense to leverage a global oracle is with the Plasma L2 paradigm. In the Plasma paradigm, an operator or group of operators submit blocks from the L2 network to the base chain (in this case Ethereum) summarizing transactions that have occurred on that chain. The submission of these blocks may not perfectly align with major events on the mainchain, such as a split causing an update of chain ID, which may cause a significant insecurity in the protocol if chain ID is utilized in signing messages. If the operators are not allowed to control the update of chain ID they will not be able to perfectly synchronize the update with their block submissions, and certain past transactions may be rejected because they do not align with the update. This is one example of the unintended consequences of trying to specify too much of the behavior of chain ID during a contentious split, and why having a simple opcode for access is most optimal, versus a more complicated precompile or contract.
>
>This proposed opcode would be the simplest possible way to implement this functionality, and allows developers the flexibility to implement their own global or local handling of chain ID changes, if required.

## References

[1] Original EIP-1344: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1344.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

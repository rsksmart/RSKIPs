---
rskip: 65
title: MINGASPRICE Opcode
description: 
status: Draft
purpose: Sec
author: JIO <jorlicki@iovlabs.org>
layer: Core
complexity: 1
created: 2018-05-11
---

# MINGASPRICE Opcode

|RSKIP          |64           |
| :------------ |:-------------|
|**Title**      |MINGASPRICE Opcode |
|**Created**    |11-MAY-18 |
|**Author**     |JIO |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |
## Abstract

In this RSKIP we propose an extra opcode. It will provide non-native smart contracts access to current block information about Minimum Gas Price. This information will allow smart contracts to mitigate some financial behavior attacks, based on statistics, by making them economically infeasible, i.e. the attack cannot make money by statistically manipulating the contract. We can also deal with network congestion from the smart contract.

## Motivation

Minimum Gas Price per block is consensed statistically in RSK by miners and was implemented following RSKIP9 to avoid bribery attacks on miners [1,2,3]. This value is included in RSK block header and modified by the miner rewarded by consensus mechanism on each new block, between certain percentual limits. If available from all smart contract this information will allow reflective measurements of the blockchain economy so the smart contracts can adapt to hostile scenarios of at least the following two kinds:

* Network congestion of the network. Transaction congestion can be detected by measuring a higher than usual Minimum Gas Price.
* Financial behavior attacks, based on statistics, by making the economically infeasible, i.e. the attack cannot make money by statistically manipulating the contract.

For the first type of scenario the contract may be designed to reduce its activity if the minimum gas price is too high. For example, if we are collecting 0.01 SBTC donations but the transaction will cost 0.02 SBTC then the contract will revert all transactions (REVERT Opcode, revert() or require() on Solidity) refunding most of the gas [4], until a most economically favorable network is detected.

For the second type of scenario the opcode will help with the self-control of smart contracts that change their dynamic behavior based on number of events per measure of time. In the last case, knowing the minimum gas price can help the smart contract estimate the cost of an attack to manipulate its behavior, thus limiting the attack, making it economically infeasible. This use case is a generalization of the case in which a smart token controling its supply be measuring the number of transactions per day is attack by a user trying to manipulate the number of transaction to inflate the token supply.

## Specification

A new opcode is added: MINGASPRICE

No arguments are pushed by caller. The minimum gas price value is pushed on the stack after execution of this opcode.

The gas cost of opcode MINGASPRICE is 2.
The opcode number is 0xAC = 172.

The gas is consumed immediately when the opcode is executed.


## Rationale

No need for for a high gas cost because is a read-only instruction, we are only reading best block header info.

## References

[1] https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP9.md

[2] Smart Contracts for Bribing Miners http://homepages.cs.ncl.ac.uk/patrick.mccorry/minerbribery.pdf

[3] Talk at BPASE '18 - Smart Contracts for Bribing Miners https://www.youtube.com/watch?v=BIKrqRE9yAY

[4] Solidity Learning: Revert(), Assert(), and Require() in Solidity, and the New REVERT Opcode in the EVM https://medium.com/blockchannel/the-use-of-revert-assert-and-require-in-solidity-and-the-new-revert-opcode-in-the-evm-1a3a7990e06e

### Copyright

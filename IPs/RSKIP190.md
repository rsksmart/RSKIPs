---
rskip: 190
title: Powpeg address change audit trail
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2020-11
---
# Powpeg address change audit trail


|RSKIP          | 190 |
| :------------ |:-------------|
|**Title**      |Powpeg address change audit trail|
|**Created**    |NOV-2020 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

This RSKIP specifies a change in the Powpeg federation migration process to generate an audit trail for the change in Powpeg addresses, embedded in Bitcoin transactions. An OP_RETURN output with the pushed message "RSKMIG" is added to the first Bitcoin transaction that the Bridge commands to sign when migrating the pegged funds to a new federation.

## Motivation

When the Powpeg federation composition changes, a new federation address is activated and the old one is disposed. Light clients and wallets must always query a trusted RSK node to obtain the latest federation address. Several methods can be used to reduce the extent of this trust. One is that the node provides certain amount of cumulative proof of work starting from the logged event that the Bridge generates when a new federation address is activated. However, a fixed amount of work does not guarantee that the federation address is the last. 
Another protection that can be useful for a wallet is to request from a full node a chain of federation migration transactions, starting with a known address embedded in the wallet source code. However, without the identification of at least one migration transaction, it's impossible for the wallet to distinguish between a peg-out without a change output and a migration transaction. This RSKIP proposes the use of an additional Bitcoin transaction output with an OP_RETURN opcode and a magic word to mark migrations. The wallet should only use the last address of the migration chain. 

## Specification

The first transaction of a federation migration process must have, as the last output, an OP_RETURN followed by the push of 6 bytes with ASCII content "RSKMIG".

## Rationale

There exists other alternatives to the proposed change, such as requiring that the old federation signs a different message containing only the new federation address. This results in a smaller chains in terms of space. However it requires changing the Powpeg node, the Bridge and PowHSM firmware, while this proposal only requires changing the Bridge code.

## Backwards Compatibility

This is a hard-forking change, but it doesn't affect smart contracts, dApps, federation nodes or PowHSMs.

## Test Cases

TBD

## Implementation

TBA

## Security Considerations

While a dumb wallet cannot validate if the last address of a given chain is actually the active address, the incentive to fool the wallet into using an old address is low, because to steal funds the attacker would need to be in control of the majority of private keys of such old federation address.
This RSKIP also gives an alternate method for Bitcoin wallets to obtain RSK federation address: scanning the Bitcoin blockchain looking for migration transactions. This ensures that the wallet has the last address without the need for scanning the RSK blockchain.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

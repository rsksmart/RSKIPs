---
rskip: 192
title: getTransactionIndex Precompile method
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2020-11
---
# getTransactionIndex Precompile method


|RSKIP          | 192 |
| :------------ |:-------------|
|**Title**      |getTransactionIndex Precompile method|
|**Created**    |NOV-2020 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

This RSKIP proposes to re-add the TXINDEX opcode functionality, which is planned to be removed in Iris network upgrade, as a pre-compiled contract. 

## Motivation

Wallets and dApps often need to collect events from a certain smart-contract. A useful technique to achieve this is to link events, which works as follows:

The contract stores the block number where state last changed (i.e. lastChangeBlockNumber) and this value is publicly accessible. Also each time the state changes an event is logged. Wallets query the contract in the best block for lastChangeBlockNumber, then they retrieve the state at that particular block to get the previous state change. To retrieve the event information, wallets have to scan all transactions in the block that may have called the contract. Alternatively, the event can be stored and retrieved from the contract storage with SSTORE, but incurring in a much higher gas cost than the LOG opcode. The retrieval process is repeated recursively until the last known change is reached. If the events are logged by the contract, the wallet has to make a choice: 

1. Be a full node and execute all transactions to reliably extract all events. 
2. Query a public infura-like node to get events. 

Both options are unsatisfactory for lightweight wallets and dApps. If events are retrieved from a third party node, the completeness of retrieved data cannot be verified, because even if given events can be verified with tree inclusion proofs, the wallet cannot detect missing events. 
The alternative method is that the contract is designed to store the events instead of logging them. The contract needs to maintain a list of all events occurring in a block, and clear it (preferably in constant gas cost) when the contract is called in a following block. This technique is expensive for the user, as SSTORE cost is over 20K. While the cost can be reduced to 5K by reusing past storage words, many words may be required to store the event information. 
A simple and clean solution is that the contract stores only two fields lastChangeBlockNumber and lastTransactionIndex. Both can be stored in the same 256-bit storage word. Also the contract must log both fields together with any other event information with the LOG opcode.
To efficiently extract the events, the wallet proceeds as follows: first retrieves both fields from storage, and then only requests a specific receipt (lastTransactionIndex) on a specific previous block (lastChangeBlockNumber) to get the prior state change event, and repeats with the transaction index and block number retrieved from the event.

RSK has currently an opcode TXINDEX for that purpose. TXINDEX pushes into the stack the index of the current transaction in the block (starting from zero), and therefore RSK contracts could potentially use the linked-events technique. However Solidity compiler does not support this opcode natively and there is no team or developer willing fork the Solidity compiler, add the opcode, and maintain the repository updated. Therefore the RSK community decided to deprecate this opcode. To keep the same functionality, in this RSKIP we propose to add to the BlockHeaders precompile a method to obtain the transaction index.

## Specification

The method with signature `GetTransactionIndex() returns (uint32)` is  is added to the BlockHeader contract that exists at address `0x0000000000000000000000000000000001000010`. The first transaction on a block has index zero.



## Backwards Compatibility

This is a hard-forking change, but it shouldn't affect existing smart contracts, because no contract should call a precompile contract with an nonexistent method selector.


## Test Cases

TBD

## Implementation

TBA

## Security Considerations

None identified.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

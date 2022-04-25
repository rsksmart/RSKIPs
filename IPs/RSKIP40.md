---
rskip: 40
title: Basic Bridge for two-way-peg to Bitcoin
description: 
status: Adopted
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-04-25
---

# Basic Bridge for two-way-peg to Bitcoin

|RSKIP          |40           |
| :------------ |:-------------|
|**Title**      |Basic Bridge for two-way-peg to Bitcoin|
|**Created**    |25-APR-2017 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted | 

# **Abstract**

This RSKIP specifies the methods the two-ya-peg Bridge exposes and the semantics of each method. 

# **Specification**

## updateCollections

Parameters: none

Iterates over the getRskTxsWaitingForConfirmations map of Sha3 hashes to transaction values and updates the RskTxsWaitingForSignatures map of Sha3 keys to Bitcoin transaction values including Bitcoin transactions for which there are enough confirmations in the RSK blockchain.

## receiveHeaders

Parameters: array of bitcoin blocks serialized in Bitcoin wire protocol format

Receives an array of serialized Bitcoin block headers and adds them to the internal BlockChain structure.

## registerBtcTransaction

Parameters: Serialized Bitcoin transaction, block number where the transaction is present, serialized partial merkle tree

Receives a serialized Bitcoin transaction, its height and a serialized partial merkle tree. Verifies the transaction hash is part of the merkle tree.

## releaseBtc

Parameters: none, the invoking RSK transaction is the input itself

Calculates a destination Bitcoin address from the address of the RSK transaction sender. Converts the amount in Weis of the RSK transaction to Satoshis for the Bitcoin transaction. If the amount is greater than the minimum non-dust value, the Bitcoin transaction with a single output is added to the map of RSK transaction waiting for confirmations using the Sha3 hash of the RSK transaction hash as key and the Bitcoin transaction as value.

## addSignature

Parameters: public key of federation member, array of signatures, RSK transaction hash

The RSK transaction hash is used as key to retrieve the Bitcoin transaction from the internal map.

Each federator must sign all the inputs belonging to the transaction that must release the BTC.

When more than N of M signatures have been collected for the unsigned transaction id, the contract will emit a BtcRelease event. 

## getStateForBtcReleaseClient

Returns transactions waiting to be signed and transactions waiting to be broadcasted.

## getStateForDebugging

Returns:

RSK Transactions waiting to be broadcasted

RSK Transactions waiting for confirmations

RSK Transactions waiting for signatures.

The Bitcoin blockchain best chain height

## getBtcBlockchainBestChainHeight

Returns the Bitcoin blockchain best chain height known by the Bridge contract.

## getBtcBlockchainBlockLocator

Returns hashes of Bitcoin blockchain blocks. Federators can use these hashes to find what is the latest block in the chain the Bridge has and synchronize.

## getBtcTxHashesAlreadyProcessed

Returns the list of Bitcoin transaction hashes the Bridge already knows about.

## getFederationAddress

Returns the current address of the federation.

## getMinimumLockTxValue

Returns a constant value, the minimum amount of BTC allowed to be locked, in satoshis.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
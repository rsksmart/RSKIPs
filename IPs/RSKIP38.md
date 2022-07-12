---
rskip: 38
title: Signature Compression
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-02-21
---

# Signature Compression

|RSKIP          |38           |
| :------------ |:-------------|
|**Title**      |Signature Compression|
|**Created**    |21-FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft | 


# **Abstract**

In this document we present a signature aggregation trick (SAT), a simple way to increase the scalability of a payment network 42%. 

**This RSKIP is superseded by the Lumino Transaction Compression Protocol [LTCP]**

## Discussion

Increasing the transactions in a per second with a function that is sub-linear with the amount of resources consumed by the network is hard. Most real improvements must pay a cost: changing the underlying security assumptions. This is the case of the Twin network, the Lumino network, and the Lightning network. However there are several tricks that can increase the tps while maintaining the same security assumptions. For example [CoinJoin] transaction cut-through and [MimbleWimble] can reduce the amount of data that is stored in the blockchain and the number of signatures that are verified. Some other minor techniques, like Schorr signature aggregation, provide small savings. SAT is a simple trick that can achieve a 40% increase in blockchain space and historic blockchain verification. 

The most common Bitcoin transaction uses only P2SH. On average, a P2SH Bitcoin transaction having 2 inputs and 2 outputs consumes 340 bytes, 128 of which are digital signatures (38%), and other 64 bytes are public keys (totaling 57%). Segwit nodes are able to remove the digital signatures and public keys from historic storage, but the cost is that the possibility to serve other users with full block data for verification is lost.

SAT proposes that users are able to provide shorter signature information by aggregating all signatures into a single signature of a hash of all the transaction data that was previously signed in the selected set of transactions. Two forms of aggregation is possible: if all inputs were signed by the same key, then a single scriptSig prefix and a single signature using the the same key can be used. In case they were signed using different keys, then the key that results from the addition of all private keys can be used to sign all hashed transaction data. 

In case of RSK or Ethereum, it is simpler. A RSK transaction is about 100 bytes in length, which 64 bytes corresponds to the signature. However RSK uses ECDSA key recovery to recover the public key from the signature, so if the signature is removed, the transaction cannot be associated with a source account. To apply the SAT trick, transactions must  contain the source address, which is 20 bytes in length. Therefore the saving is actually only 42 bytes over 100 bytes, or 42%.

To sync with a blockchain using this trick, N blocks ahead must be fetched until all transactions in a block can be verified.

The SAT trick has the obvious drawback that trades scalability for privacy, as the re-use of Bitcoin addresses reduces the privacy of users.

 

# **Specification**

A new optional field is added to transactions: "sender". Sender specifies the source address. When specified, after the signature recovery is performed, the resulting address is compared with “sender”. On a mismatch, the transaction invalidates the block.

A new payload of a normal transaction SAT. A SAT payload has:

- A list of transaction ids. Each transaction must have the "sender" field.

- A signature of a root hash of a Merkle tree built on the transaction id list.

- This information is piggybacked to a normal transaction.

This transaction should be included in a block before the maturity time of the transactions referenced. The effect of this transaction is that is removes the signatures of all referenced transactions.

The cost of this transaction has a special discount: 14000 gas for each transaction referenced. With this discount, the throughput of the network  is increased almost 3X. However the cost cannot be negative, since no miner would be willing to include it. Therefore the transactions with SAT payload should be gas consuming. E.g. A transaction consuming 300K gas can clear 20 signatures. This will create a market were SAT payload is sold/bought so that users can decrease the cost of their high gas consuming transactions. But also open the door to problems of overloaded blocks that consume much more resources than the gas limit. Also user can add SAT payload because of altruism. 

An alternative is that the SAT payload reduces the fees to those miners that originally included the referenced transactions (this is done before the maturity of the blocks that included them). Therefore a miner is incentivized to include the SAT payload if he was not itself the same miner that mined many of the previously cleaned transactions. The REMASC prevents some incentives from censoring SAT, since fees are smoothed. However this alternative has many problems that arise from competing incentives for miners.

[CoinJoin]: https://bitcointalk.org/index.php?topic=281848.0
[MimbleWimble]: https://scalingbitcoin.org/papers/mimblewimble.pdf

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
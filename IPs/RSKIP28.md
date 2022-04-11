---
rskip: 28
title: Ephemeral Data
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2016-12-29
---

# Ephemeral Data

|RSKIP          |28           |
| :------------ |:-------------|
|**Title**      |Ephemeral Data|
|**Created**    |29-DIC-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

[comment]: <> ([ MUST BE COMPLETED WITH THE DESIGN PRESENTED IN THE MIT ])

# **Abstract**

This RSKIP proposes that data can be transmitted to contracts in a way that its availability is delayed, and if not used for some time, the data is disposed.

# **Motivation**

Most smart contracts perform computations based on information coming from the outside world. This data must be transferred to the platform through the transaction layer. The cost of such transfers is generally significant, because the platform must price several factors into transfers. Every byte transferred:

* consumes bandwidth.

* delays block propagation (because it increases the inter-node transfer time).

* delays block propagation even more (because the data must be processed when verifying the block at each hop).

* must be stored in the blockchain forever.

* needs to be read from HDD/SSD when requested by a peer

* needs to be stored in RAM when processed.

Each of these items has a cost to the network. Some apply only to miners while other apply to all nodes.

In this post I propose a new technique to transfer data to smart contracts and to network nodes at very low costs, thus enabling new use cases.

# **Specification**

**Introducing Ephemeral Data**

The new technique is called Ephemeral Data or the less compact Segregated, Conditionally-Stored and Optionally Delayed Data.

I’ll introduce Ephemeral Data with an analogy: the Schrödinger’s cat paradox. In this thought experiment related to quantum mechanics, a cat is "locked in a steel chamber, wherein the cat’s life or death depended on the state of a radioactive atom, whether it had decayed and emitted radiation or not. According to Schrödinger, the Copenhagen interpretation implies that the cat remains both alive and dead until the state is observed.". The same idea can be applied to transaction data. The “Schrödinger’s” data can be sent to the blockchain in a new special transaction field that remains invisible to consensus unless a smart-contract refers to it. When the data is observed, it becomes part of the blockchain forever. If the data is not observed, the data disappears and all that remains on-chain forever is the data hash digest (and later we’ll see that even this hash may be disposed).

The cost of adding Ephemeral data to a transaction is related to the bandwidth consumption to transfer it, but the sender does not need to pay (immediately) for eternal blockchain storage.

**Ephemeral**

Ephemeral data is available to be referred during a certain pre-defined period (Ephemeral period). When this period is over, the data, as far as consensus is concerned,  is lost forever. The cost of Ephemeral data will depend on, among other factors, the Ephemeral period selected by the sender. The higher the period, the higher the cost. The size of the Ephemeral data also multiplies its cost.

**Segregated**

Ephemeral data is not directly hashed along with the other transaction data to be signed, only the hash digest of the Ephemeral data is. Therefore when Ephemeral data is disposed, only the hash digest remains. In some cases not even the hash digest needs to be stored, as the LTCP protocol can provide a signature of a previous transaction record excluding the Ephemeral hash digest. This can only occur of the Ephemeral period is over before the next transaction is included in a block.

**Conditionally-Stored**

Ephemeral only becomes part of the blockchain if it is loaded by a contract using the EDLOAD opcode. If it is not used before the ephemeral period deadline, no storage is necessary. Full nodes can be optimized to store Ephemeral data in RAM, when the ephemeral period is short. This way the full nodes saves a disk access for data storage and another disk access for data deletion.

**Optionally Delayed**

Ephemeral data can be optionally Delayed, by request of the transaction owner, meaning that a smart-contract can only access the data starting from the second block following the block of transaction inclusion. This lowers the cost of Ephemeral data propagation time, as the Ephemeral data can be transferred with less priority than the remaining (non-Ephemeral) parts of the block.  Even if there is a risk that a miner creating a block does not broadcast the Ephemeral data later, forcing the remaining miners to backtrack, this is the same risk accepted by miners that do verification-less mining for some seconds after receiving a block header.

**Ephemeral Data and scalability**

Ephemeral data turns out to be a powerful scalability tool that enables several new use cases. For example, it enables using the cryptocurrency network as an encrypted transport for peer to peer messages, such as in the Bitmessage protocol.  Ephemeral data automatically forces nodes to temporarily store the messages in the network. After the ephemeral time, if the receiver has not checked its inbox, the message is automatically destroyed. The transaction doesn’t need to have destination, nor value. The full encrypted message (including real message destination address) can be hidden in encrypted Ephemeral data. This transaction, compressed by LTCP, could consume only  a few bytes before being disposed. While this hypothetical messaging system may not be a long term scalable and cheap peer-to-peer messaging solution, it’s indeed interesting and novel, and can have a use case for decentralized anonymous broadcast.

**Ephemeral Data Internals**

Every transaction has two data sections: normal data and Ephemeral data. The signature, instead of covering a hash digest of the full transaction, covers a hash digest of the transaction concatenated with the Ephemeral hash digest. Although not implemented in RSK, it’s possible that a transaction holds several independent pieces of Ephemeral chunks, each identified by its own hash digest.

When a transaction is included in a block, Ephemeral data is separated from the rest of the transaction. Transactions are stored in the standard transaction tree, Ephemeral data is transferred in a separate list that may not be complete. As with LTCP, the blockchain synchronization protocol is modified to accept a block basing the decision on the existence of signals in following blocks. To facilitate this signaling, a hash digest representing a Merkle root of the list of referenced ephemeral datas can be included in the block header. 

A transaction that uses Ephemeral data can specify three new fields:

EDSize (uint) specifies the Ephemeral data size. This is required because the cost of the transaction depends on this size. If not specified, it’s assumed to be 32 bytes.

EDDelayed (boolean) indicates if this data is available immediately or it’s delayed 2 blocks. If not specified, it is assumed false.

EDBlockCount (uint): specifies how many blocks the data will be available to contracts. The ephemeral count must be lower than a certain fixed bound. If Ephemeral data is not used before the deadline, the Ephemeral data can be removed safely from the blockchain forever. However the actual removal from memory in full nodes should occur several thousands of blocks later, until it’s clear that the block will not be reverted by a chain reorg. If this field is not specified, it is assumed to represent 24 hours in blocks.

**Using Ephemeral Data in a Contract**

Any contract can use Ephemeral data by executing the opcode EDLOAD. The opcode first argument is the ID (hash-digest) of the Ephemeral data chunk, the second being the offset and size of the part to retrieve. To retrieve all, the offset should be set to zero and the size should be set to the value returned by EDSize. Contracts can obtain the ID of a Ephemeral chunk in a transaction by executing the opcode EDID. This opcode pushes the ID of the Ephemeral data of the current transaction into the EVM stack. If the transaction has no Ephemeral data, it pushes the all-zero value. EDLOAD fails if a contract tries to load Ephemeral data marked as Delayed before the second block after inclusion. It can, however, store the ID value for future use, or receive it later in another transaction payload. The EDSize receives an ID and pushes on the EVM stack the size of such object. If the object is not present, then the resulting value is all zeros.

**Ephemeral Data and Block Propagation**

Ephemeral IDs can be used during block validation, but the Ephemeral data itself cannot interfere with validation if Ephemeral data is marked as delayed. Therefore, block processing can be carried out in full without having received this data.

This gives miners the chance to begin mining a child of a block even if not all the previous block information has been received. This is similar, but not equal to validation-less mining, since a validation-less block cannot include transactions, while a miner that has not yet received delayed Ephemeral data can freely include new transactions in a child block. However, the miner must backtrack if some time elapses (e.g. 5 seconds) and the delayed Ephemeral data was not received, to prevent a miner from attacking the network by withholding the Ephemeral data.

**The Price of Ephemeral Data**

Since delayed Ephemeral data in a block does not increase the time required to propagate the critical transaction data, delayed data can be priced much lower than normal transaction data.

We estimate that in delayed Ephemeral data will be priced at 10 gas/byte, so a block with a 4M gas limit may contain 400 Kbytes of delayed Ephemeral data. Non-delayed Ephemeral data will be priced at 30 gas/byte. When the Ephemeral data is referenced by the EDLOAD, the contract must pay an additional 50 gas/byte. For comparison, in RSK or Ethereum, each non-zero transaction data byte costs 68 gas/byte, therefore unreferenced delayed Ephemeral data will cost less than 1/6th of standard data cost. 

## Ephemeral Data Use Cases

Ephemeral data has many use cases. We’ll describe only a few of them here.

**Fair Large Multi-player Games**

When the number of participants taking part of a multi party protocol is high, the risk of communication interruptions rises considerably. Imagine a fair multi-party game with the following requirements:

Each participant must make a move at frequent intervals. Each move consist of  a large binary data string representing a set of player actions, such that move verification is too expensive to do on-chain

Each participant will verify that the moves made by the remaining participants are valid according to the game rules, because that’s in each player best interest.

However, the participants do not trust each other, so if Alice detects that Mallory tried to cheat, the remaining participants will not just trust Alice, but will use a smart contract as a trusted arbiter to validate the move, even if that is expensive.

Clearly the move data can be made Ephemeral data. This use case is closely-related to state channels. A state channel is frequently implemented as a contract that works as an arbiter when a party tries to cheat. But allowing data to be sent as Ephemeral data is particularly useful when the amount of data and the number of participants is high. In that case, the cryptocurrency network can even be used as a secure broadcast medium when required.

Ephemeral data improves state channels to reduce the on-chain load.  In the state channel approach, all messages are exchanged privately (e.g. in an IRC channel or game room), and if a player Alice does not receive a move from Bob before a timeout, she must challenge Bob to present the move using a secure broadcast medium, such as the blockchain. But if Bob later presents a perfectly valid move to the broadcast medium, it’s not clear whose fault it was and who should pay the verification cost.  If Bob can present a valid move in the broadcast medium, then there are two possible reasons why Alice did not receive the date before:

Alice wait for the reception of the move data timed-out because of a Internet connection problem

Bob withheld the move data to Alice on purpose

Because the Internet does not guarantee timed delivery, neither Alice nor Bob can prove which was the case. Then it’s unclear who should pay for the cost of the challenge and the response transactions. Let’s recall that the move consisted of a large data chunk, representing a high cost for the blockchain to store forever. If Alice should pay, then there will be an incentive not to challenge other players, as they can withheld information on purpose, which breaks fairness. If Bob should pay, then there will be an incentive to challenge other players even if there is no reason to, to make them pay more, so fairness is also broken. Ephemeral data solves this problem by lowering the cost of broadcasting the move in reply to a challenge. The cost of a false challenge will be so low that the cost can be split and paid by both of them. 

**Cheap Secure Push Oracles**

Smart contracts require oracles. Oracles bring information of the outside world into the consensual state machine.  There are two models for absorbing information from oracles in a smart contract: the pull model and push model.

In the pull model, the user contract queries an oracle contract in a certain transaction in a block. The owner of the oracle (in practice, a computer system obeying the commands of the oracle contract) performs the real-world query and sends a message to the oracle contract containing the information. This message is transmitted in another transaction, generally several blocks after the query. The oracle contract then calls-back the user contract, providing the requested information. This model is slow and expensive, as it requires one additional external transaction and an additional internal call. Both increment the amount of  gas consumed on average. This model is generally useful when the information is seldom requested more than once: for instance, a complex call to a server exposed JSON-RPC API.

A cache in the oracle contract can lower the cost (on average) when multiple equal requests happen over a short period of time (e.g. the same block), but the management of the cache itself  also costs gas. But in any case, the user contract must be programmed to consider the case of a delayed response.

In the push model, the oracle owner periodically pushes information into the oracle contract. This model is better suited for values that are commonly requested and easily tabulated,  for example, exchange rates. Also the push model allows users to know more precisely which information will be consumed by the contract before making a call to it. If the oracle feed is delayed, the user has no way to know for sure the actual value the oracle will provide before the fact. Oracle creators can use the push model making oracle  data Ephemeral. Therefore the oracle data is only persisted in the blockchain if it is consumed, and the cost of maintaining the push oracle is greatly reduced.

**Cost savings in Consensual Verification of Large Datasets**

Imagine a distributed application with the following needs:

A user must commit to a large result data that a smart contract can verify

but such verification is expensive to do on-chain so

the verification will be performed by a set external verifiers  instead.

However, these verifiers do not trust each other, so if a verifier lies, the remaining verifiers will request a smart contract to be the trusted arbiter.

Temporary locked deposits by all participants allows that any cheater is punished by confiscating the deposit. This is exactly the the use case for the RSK full node rewards system, that is based on Proof-of-Unique-Blockchain-Storage (PoUBS).

**Ephemeral Data and the RSK Full node Reward System**

RSK will use Ephemeral data for its Proof-of-Unique-Blockchain-Storage (PoUBS) protocol and technology. The PoUBS protocol allows a smart-contract to reward nodes that store the blockchain. The PoUBS protocol requires full nodes to periodically send to a special smart contract hash digests of pseudo-random chunks of blockchain data in order to prove that they hold an unique copy of the blockchain. The remaining full nodes that participate in the reward scheme can check other node’s responses. Most of the times proofs will be valid and therefore there will be no need to store them in the blockchain nor to verify them by a smart contract. In the rare cases a user tries to cheat, and provides invalid data as proof, the smart contract can be commanded to verify the proof and punish the offending user.

Soon I will post about more about the PoUBS protocol, the parameters chosen, and how the contract implements it.

## Summary

Ephemeral data is a new tool to achieve scale smart-contract blockchains. Ephemeral data enable new use cases where users cheaply commit to large chunks of data that are visible to the other users, but that remain out of the blockchain unless required during arbitration.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
---
rskip: 63
title: Double Signing for Delayed Signature Aggregation
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2018-05-07
---

# Double Signing for Delayed Signature Aggregation

|RSKIP          |63           |
| :------------ |:-------------|
|**Title**      | Double Signing for Delayed Signature Aggregation|
|**Created**    |07-MAY-2018 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

Blockchain historic size is one of the restrictions for scaling. A RSK transaction uses 70 bytes for ECDSA signature data and approximately 30 for the transaction payload itself. Therefore by removing the signatures from transactions the blockchain can scale 2.3 times. One technique to reduce the size of signatures is signature aggregation.  This RSKIP proposes aggregating signatures in a novel construction to reduce blockchain space consumption.

## Introduction

There are several types of signature aggregation: sequential (ordered), underdered, interactive and non-interactive. In this RSKIP we’re interested in non-interactive aggregation. The type of aggregation that can be accomplished by a miner on third party transactions. Therefore we are interested in signature aggregation of signatures of different messages and different public keys. Some signature schemes, such as BLS, allow it. However the state-of-the-art signature schemes that provide this kind of signature aggregation are slow. Aggregating a BLS signature may cost 10 times more (in terms of CPU cycles) than validating a ECDSA signature. Signature verification can be cached, so signature verification is generally not part of the critical path when broadcasting a block. The same happens with signature aggregation: the heavy non-constant operations can be cached. However, in the worse case a block can contain all new transactions, and each node would need to perform the slow operations. The slow operations would grow linearly with the number of transactions, and will affect the critical path of block verification. This RSKIP brings a new solution to this problem, based on double-signing transactions

## Description

The solution proposed involves allowing users to sign a transaction with ECDSA and also signing with BLS, adding a second signature. We’ll call these kinds of transactions Aggregatable Transactions (or just ATs). The public key of the BLS signature would be stored along the account state of the source account. The consensus rules of the network must be carefully changed. Every N blocks, an "aggregation event" (AE) occurs, and all valid and mature BLS signatures in blocks prior block (N-2) are aggregated, and all signatures (either ECDSA or BLS) that were related to the aggregated BLS signatures are removed from the blockchain. By “mature” we mean that they have an M block confirmation (the need for this will be more clear when we discuss attacks). For example, if an aggregatable transaction T will use 70 bytes for a ECDSA signature and 20 bytes for a BLS signature, then the whole 90 of bytes of signature data will be removed at a future block. Because ECDSA signatures are used to derive the transaction sender by ECDSA public key recovery, transaction will also need to specify the source account using an short field, such as an unique prefix, or account creation index number. If there are no AEs after a transaction T, then a node verifies the ECDSA signature of T. If there is an AE after T, then nodes only verify the aggregated signature and won’t expect signature data for T along with T payload. Therefore the critical path in block propagation over synced nodes does not need to perform BLS signature verifications. Nodes that are synchronizing compute the aggregation of signatures in background. The block which contains the AE cannot bring new transactions for aggregation because the aggregation interval stops 3 block before. Therefore processing blocks that contain an AE do not affect much the critical path.

## Improvements

Let’s recall how BLS signature aggregation works:

**Key Generation.** For a particular user, pick random x R← Zp, and compute v ← gx . The user’s public key is v ∈ G. The user’s private key is x ∈ Zp. 

**Signing.** For a particular user, given the public key v, the private key x, and a message M ∈ {0,1} ∗ , compute h ← H(v,M), where h ∈ G, and σ ← hx . The signature is σ ∈ G. 

**Verification**. Given a user’s public key v, a message M, and a signature σ, compute h ← H(v,M); accept if e(σ,g) = e(h, v) holds.

**Aggregation**. Arbitrarily assign to each user whose signature will be aggregated an index i, ranging from 1 to n. Each user i provides a signature σi ∈ G on a message Mi ∈ {0,1} ∗ of her choice. Compute σ ← ∏n i=1 σi . The aggregate signature is σ ∈ G. 

**Aggregate Verification**. We are given an aggregate signature σ ∈ G for a set of users, indexed as before, and are given the original messages Mi ∈ {0,1} ∗ and public keys vi ∈ G. To verify the aggregate signature σ, compute hi ← H(vi ,Mi) for 1 ≤ i ≤ n, and accept if e(σ,g) = ∏ n i=1 e(hi , vi) holds. 

To verify a signature, the verifiers must compute e(hi , vi) for each i, which is expensive. We’ll call this term the per-signature pre-computation term (PSPC). 

### A flawed Idea

To reduce the load on the verifiers, each transaction T could also include the PSPC term, signed also by the ECDSA signature. Synced full nodes could use these terms to build the aggregated signature before disposing the PSPC, the BLS signature and the ECDSA signature. This however does not work, because users may send transactions with invalid PSPC terms that may allow them to impersonate other users and force a full node to accept a transaction Z which was never signed.

## Discussion

The ideal candidate signature scheme for aggregation would be one that requires a lot of time to build the aggregated signature but very little time to verify. However BLS posses the opposite property.  More research on new signature schemes would be needed.

## Cost

Because the space savings represents only a part of the transaction payload, the reimbursed amounts can only represent a part of the transaction cost. In case of RSK, this is 21K gas. Transferring value in RSK costs 9K gas. The reimbursement for removing signature data can be estimated as between 6K and 10K gas. This means that performing reimbursements costs as much as the amount reimbursed. To reduce the reimbursement cost, the platform must be modified so that the change of account state of the last 1000 aggregatable transactions is not written to disk until an AE comes: account state information is kept in RAM. Therefore performing the reimbursements corresponds to RAM-only operations, and the cost can be assumed to be much lower. The drawback, which is minor, is that if a node crashes, it would need to re-process some number of last blocks in order to recreate the world state.

## Attacks

One vector of attack is to build a block B that contains a transaction Q with a bad ECDSA signature, a bad BLS signature, or both, and try to force the network (or a part of the network) to accept this block.

Clearly synchronized nodes won’t accept B unless B is followed by additional blocks until an AE is present such that signatures on Q are not required anymore. If a miner possess a valid BLS signature for Q, and this miner mines the AE involving Q, then this miner can reintroduce the branch that includes B that was discarded by the network. This is no different than a block withholding attack, so it presents no new danger. However to prevent from network partitions nodes should tag blocks that contain invalid ECDSA/BLS signatures not as invalid, but as "temporary invalid until height X", which means that the block may be reconsidered later if the branch containing the block reaches height X.

To prevent any other unforeseen network partition attack, signatures can be aggregated only if they are mature, such as having M confirmations. Therefore an attacker would need to mine at least M blocks selfishly before an AE can remove the invalid signatures from Q. By making M high enough, such as M=100, the attack is mitigated.

# **Specification**

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

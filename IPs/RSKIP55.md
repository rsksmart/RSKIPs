---
rskip: 55
title: Native On-Chain Probabilistic payments
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-03-11
---

# Native On-Chain Probabilistic payments

|RSKIP          |55           |
| :------------ |:-------------|
|**Title**      |Native On-Chain Probabilistic payments|
|**Created**    |11-MAR-2017 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

Transaction costs will increase with the demand for the platform services because the block gas limit cannot be arbitrarily increased without affecting other properties of the platform. The second layer payment networks, such as Lumino, enable off-chain payments at low cost. However these networks require pre-locking funds. In this RSKIP we propose an alternative to the second layer networks to perform cheaper payments without the need to lock funds, based of probabilistic payments. Also this RSKIP proposes one variant of on-chain probabilistic payments that does not require an additional communication channel between sender and receiver. The [RSKIP56] proposes an off-chain variant  of probabilistic payments.

# **Motivation**

Many new micropayment use cases can be realized with payment channels, such as pay-per-minute of video, bandwidth, CPU or consulting services. However second layer networks are complex and bring second layer networks have different security guarantees than blockchain transactions. Probabilistic payments are payments that only occur with certain probability. Suppose a payment stream of $0.01 payment per second. This can be transformed into a stream of one $ 0.10 payment per 10 seconds. If the probability of each payment is 1/10, then each payment can be a $ 1 probabilistic payment, therefore it can be transformed into a stream of $ 1 1/10 probabilistic payment per 10 seconds, which results in one actual $1 payment every 100 seconds, on averaga.

A probabilistic transaction can involve smart contract execution.

## Discussion

One possibility is to define new field "odds" (O) is added to the transaction. If O is zero or one, then it’s a normal non-probabilistic transaction. Otherwise, then the probability of the transaction is 1/O. Nodes that broadcast transactions must only broadcast a probabilistic transaction if  the gas price is is higher than for normal transactions. It doesn’t have to be “odds” times higher, because the transaction is executed only once, while many transactions can be propagated for each transaction that is selected to be executed. If propagation represents the factor F of the total transaction cost, then the gasPrice (gp) can be to avg_gp*(1+(O-1)*F). The savings for the sender is therefore (1+(O-1)*F)/O= (1-F)/O+F. This is probably dominated by the term F, so if F is 20%, then a probabilistic transaction would for O large cost only 20% of the normal cost.

The problem with this approach is that, because each probabilistic transaction can specify a different odds, miners cannot know in advance how much gas will be consumed in the probabilistic transaction. If the odds are fixed, then the miner can separate the probabilistic transaction into slots, where only one slot is always chosen, and where the gas consumed by in total by the transactions in each slot is approximately the same.

Therefore this RSKIP uses the fixed 1/16 probability for probabilistic transaction. This number was chosen because it brings low payment variability and because it’s high enough so that gas savings are relevant.

To allow the miners to include probabilistic transactions without the need to execute them first, the Gascount in probabilistic transaction is fully paid. There is not reimbursement of unused gas.

Because the network propagates probabilistic transactions, these transactions will be tentatively executed in blocks until:

* The source creates a non-probabilistic transaction with the same nonce.

* The nodes remove the transaction from the mempool.

# **Specification**

A new field is added to each transaction: isProbabilistic. If true (1), then the transaction is probabilistic with odds 1/16.

A new field is added to the block header: probabilisticTxs. This field is the root hash of a Merkle Tree that contains in leave nodes, the transaction ids. In intermediate nodes, it contains the hash D. The hash D is computed as D=Hash(rlp(rlp(L),rlp(H),rlp(Hash(LHash,RHash)).  LHash and RHash are the values of the left and right childs. If there is no right child, the hash must be all zeros. rlp() is the RLP encoding function. L is a byte array which contains a prefix of a key that is lower or equal than the left-most transaction ids in the tree below it (when key is padded with zeros), but higher than all the intermediate and transaction ids keys to the left of this node. R is a byte array which contains a prefix of a key that is higher than all transaction ids in the three below it (when padded with 0xFFs), but lower than all keys and transactions ids to the right. For example, if the leave transactions IDs are AAA, AAB, BCC and DDD, then the intermediate node L and H keys can be ([ ] , A) for (AAA,AAB) and (B,[]) for (BCC,DDD), and ([],[]) for the root node. This assignment of keys is valid  because 000 <= AAA,  000 <= 000, AFF > AAB , AAF <= BCC, B00 <= BCC, AAB <=B00, DDD < FFF.

Also, all transaction ids must be sorted. This data structure allows for the sender to reveal a consecutive segment of the increasing transaction ids (but not all), together with Merkle proof  that the segment was correctly ordered within the complete list. 

The set of transactions will be split into 16 consecutive sub-segments. Another header field: probabilisticIndex contains a list of 15 consecutive integers, much mark the sizes of the different sub-segments. From the sizes, the first element of each segment can be computed.

Miners choose subsegments such as each sub-segment consumes approximately the same amount of gas.

The gas limit applies both for normal transactions and the trie of probabilistic transactions chosen. 

Let blockID be the block id of a s solved block (and contains the PoW). The last 4 bits (LSB) of the blockID indicate the sub-segment to execute. The miner must include the list of probabilistic transactions in this segment, and the Merkle-proof that allows to verify the inclusion of such sub-segment in the Merkle tree and also that the transactions in the segment are consecutive to the remaining (non-revealed) ones.

The transactions that were not executed can be included in subsequent blocks. Transactions on the ProbabilisticTx tree could be duplicated, but a successful revelation of the block id will only success for at most one of the duplicates. Therefore it’s unimportant if the Merkle tre is fully correct: only that the revealed segment is correct. 

When executing the transactions in the probabilisticTx tree the transaction index is set if the probabilistic sub-segment followed the the list of non-probabilistic transactions. Therefore if there are 100 non-probabilistic transactions, and the index is zero-based, then the first probabilistic has index 100.

When a block is propagated, it’s propagated along all the transaction in the executed sub-segment. The cost of the additional data that must be propagated for the Merkle proof is very low. For example, for 256 probabilistic transactions with uniformly distributed IDs, it would be about (32+6)*16=576 bytes, which represents an overhead of 2 bytes per transaction (normally about 100 bytes in length), and it’s negligible.

Also all non-executed transaction ids could be temporarily propagated, and their sequential order and Merkle tree inclusion be enforced, for any block not having more than 10 childs. Full nodes accept this block into the main chain if :

1. The non-executed probabilistic ids are valid or

2. The block has 10 childs or more, and the non-executed ids doesn’t matter.

 

Nodes should place any block with invalid ids, in a temporary pool, until it receives 10 confirmations, or an alternate branch of the blockchain surpasses the block or can discard the block, but should never permanently mark the block as invalid.

Because the execution probability is 1/16, the real cost (to the sender) of a simple probabilistic transaction is 21K/16= 1312 gas.

Lightweight nodes must be able to find executed transactions both in the non-probabilistic tree and in the probabilistic tree chosen.

The stateRoot fields will still correspond to the blockchain state after executing the non-probabilistic transactions. A second field finalStateRoot could added to the header (but excluded from the header that enters the PoW). It would correspond to the blockchain state after executing the probabilistic transactions. However, there is no benefit for this. The stateRoot for the next block, however, will consider the execution of the probabilistic transactions.

## Example Use Case

Because probabilistic transactions may be referenced in many blocks, the protocol for fair probabilistic payment must be carefully designed. 

Pay-for-minute between Alice and Bob:

* Let assume the rate is $ 0.1 per 10 seconds. Let the average block interval be 10 seconds.

* Alice creates a probabilistic transaction T for Bob. The transaction would pay $ 0.6 (the amount if every block would reference the transaction). Let’s say she specifies $ 1. then she broadcasts the transaction.

* Bob waits until T is broadcast, and assumes miners are referencing it in probabilisticTx. Then it starts giving Alice a 1 minute service.

* Each time T is referenced, Bob considers that Alice probabilistically paid $ 0.1. Let A(t) be the accumulated amount of probabilistic payments at delta time t (t is specified in seconds).

* When T is executed, Alice immediately creates a transaction T’ with the next nonce, specifying the same amount, and she broadcast it. Alice can increase or decrease the amount her transactions are being referenced too often or too little. 

* If Bob has provided a service for t time and A(t)< t* 0.1/10+0.6 then Bob interrupts the service, and waits for more  references.

* Alice must make sure that A(t) is high enough, either by increasing the gasPrice, or increasing the amount paid per transaction.

## Comparison between on-chain and off-chain probabilistic payments

On-chain probabilistic payments do not require a second communication channel between sender and receiver. Therefore the service delivery and payment can be made completely anonymous. For example, a video service may be received by encrypted satellite radio transmission, and the payment made to a static address.

Also on-chain probabilistic payments are more secure. When an off-chain PP must me effective, it must be broadcast to the blockchain and included in blocks. The sender can collude with miners to censor the payment, and therefore the server would have provided its service for free. In the case of on-chain probabilistic payments, the service is pre-paid. However, when once PP is executed, the user must issue a new probabilistic payment. During this period the receiver is at risk: however the risk is 16 times lower, because it’s not known if the broadcasted payment will be executed or just referenced. Therefore on-chain PP better protects the seller from anonymous buyers. 

Off-chain probabilistic payments cost a bit more than the on-chain counterparts, since the off-chain probabilistic payment must include both the hash digest and a pre-image of it.

Off-chain probabilistic payments are more flexible, because the user can specify the odds.

[RSKIP56]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP56.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
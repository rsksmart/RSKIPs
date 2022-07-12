---
rskip: 14
title: Reward Manager Smart Contract (REMASC)
description: 
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2016-09-06
---
# Reward Manager Smart Contract (REMASC)

|RSKIP          |14           |
| :------------ |:-------------|
|**Title**      |Reward Manager Smart Contract (REMASC)|
|**Created**    |06-SEP-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Rejected |


# **Abstract**

RSK will be the first blockchain based on proof-of-work consensus that has no subsidy. Also RSK will be the fastest existent PoW blockchain. All network parameters and protocols that we design to allow these goals must be supported by a miners rewards structure that incentivizes the honest planned behaviour. Therefore the main design parameter that can provide such guarantees is how the transaction fees are split between miners. At the same time, RSK wants to maintain the other usual design goals, such as preventing excessive centralization. Such stringent requirements ask for novel solutions, and is has become clear that due to the numerous possible attack vectors, such solution will not be a simple one.  However, the solution must fit into the smart contract framework easily, without interfering or preventing the achievement of other platform goals, such as allowing nodes to easily download the blockchain state from the worstate trie, without additional state data.. Therefore this RSKIP proposes that the reward splitting algorithm is coded into a smart contract, which will be called REMASC (Reward Manager Smart Contract). By doing this, all extra state will be part of the REMASC persistent storage. 

# **Motivation**

The following is a list of the design goals behind RSK reward splitting strategy and consensus protocol:

- The blockchain consensus is driven by SHA256D merge-mined proof-of-work

- The desired average block interval is 10 seconds.

- The consensus protocol should not include new incentives for mining centralization.

- The consensus protocols should reduce incentives from block withholding [[https://petertodd.org/2016/block-publication-incentives-for-miners](https://petertodd.org/2016/block-publication-incentives-for-miners)]

- The consensus protocol should reduce centralization incentives derived from stale blocks. [[https://blog.ethereum.org/2014/07/11/toward-a-12-second-block-time/](https://blog.ethereum.org/2014/07/11/toward-a-12-second-block-time/)] and [Accelerating Bitcoin's Transaction Processing, Fast Money Grows on Trees, Not Chains, Sompolinsky, Zohar](https://eprint.iacr.org/2013/881) 

- The consensus protocol should reduce the probability of transaction reversal as much as possible.

- Therefore, the consensus protocol should not allow profit from selfish-mining [Majority is not Enough: Bitcoin Mining is Vulnerable, Eyal, Gun Sirer, avilable at [https://arxiv.org/pdf/1311.0243v5.pdf](https://arxiv.org/pdf/1311.0243v5.pdf)]

 

- The consensus protocol must be incentive compatible [ Incentive Compatibility of Bitcoin Mining Pool Reward Functions, Schrijvers, Bonneau, Boneh, Roughgarden, available at [http://theory.stanford.edu/~tim/papers/bitcoin.pdf](http://theory.stanford.edu/~tim/papers/bitcoin.pdf) ]

- The consensus protocol should not allow profit from Camacho selfish-mining [DECOR+LAMI: A Scalable Blockchain Protocol, [https://scalingbitcoin.org/papers/DECOR-LAMI.pdf](https://scalingbitcoin.org/papers/DECOR-LAMI.pdf) ]

- The consensus protocol should not allow profit from Uncle mining. [[https://bitslog.wordpress.com/2016/04/28/uncle-mining-an-ethereum-consensus-protocol-flaw/](https://bitslog.wordpress.com/2016/04/28/uncle-mining-an-ethereum-consensus-protocol-flaw/) ]

- The consensus protocol should prevent instabilities from no-subsidy mining [On the Instability of Bitcoin Without the Block Reward, Carlsten,Kalodner,Weinberg, Narayanan, available at [http://randomwalker.info/publications/mining_CCS.pdf](http://randomwalker.info/publications/mining_CCS.pdf) ]

- It should allow for mining blocks without executing validating parent block transaction (for validation-less mining up to certain depth)

- The mining rewards should not be paid to miners until a certain maturity period (in terms of blocks) has elapsed. This time-delayed rewards adds more pressure for miners not to attack the network, and rewards held by the platform act as a security bond.

- The penalization for not including a competing uncle should be higher than the loss for sharing the reward. This considering the cost of bribing subsequent miners not to include the uncle.

- The consensus protocol should be protected from starvation 

This RSKIP proposes a protocol and an implementation that tries to satisfy the design goals.

The core protocol extends [DECOR+]. 

The fees collected by any block are not paid directly to the miner of the block, but they are transferred to a native smart contract called REMAC. The transfer is done automatically, and corresponds to an emulated simple transaction transaction that carries all of the block’s transaction fees as value.

# **Rationale**

The strategy proposed here is a variation of [DECOR+]. In a nutshell, the strategy is to share the block reward between all miners that have solved a block of the same height, and also with subsequent miners. To help the explanation we'll first assume for a moment that all block fees are almost equal so each miner receives almost the same net payment for a block (there is no difficulty adjustment, nor subsidy, only a never ending transaction backlog). We'll also simplify the explanation by limiting ourselves to a conflict between two miners, and we'll later show this is the most common case. Whenever two miners (Alice and Bob) mine two competing blocks (a block conflict) the following happens:

- If one block has a reward higher than two times the other block reward, both miners decide to mine on top of the on one with highest reward

- Else, miners decide to mine on top of the block with the lowest hash.

This will be the conflict block selection rule. For miners to be able to compare competing blocks, all conflicting blocks headers that are not too old (no more than 10 steps back of the chain tip) are forwarded by the network. If a miner Carol (which could be also Alice or Bob) solves a following block, she can decide to include in her block a reference to the uncle block header that was left out of the main chain, can collect an extra prize when the coinbase of the conflicting block matures . The prices and rewards are computed according to the following steps:

* For every block there is a pre-existent Reward balance (RB).

* Also there is always a Burned Balance (BB).

* All the fees paid in a block are accumulated into the RB.

* 10% of the RB is extracted as the Full Block Reward (FBR).

* If the conflicting block and sibling headers do not obey the selection rule a **punishment fee** of 10% is subtracted from the FBR. The punishment fee is burned by transferring it to the BB. If there are no siblings other than the main one, there is no punishment fee.

* A 10% of the remaining FBR is the **publisher's fee**, and it is shared in equal parts between the miners that include sibling headers. If there are not sibling headers other than the main one, there is no publishers fee. 

* The remaining FBR is the **reward share** and it is split in equal parts between the miners that solved the all the sibling blocks (including the miner of the block in the chain and the miner that solved the uncle headers).    

The rewards are handled by a the REMASC smart-contract. The balance of the REMASC contract must always be (BB+RB). 

**Changes to the Block header**

The stateRoot field is removed, instead the following fields is added:

stateRootCommitment =SHA3( parentHash | stateRoot |  gparentStateRoot )

Block headers have a new **empty** boolean  field. If empty is true, then the follow conditions must be valid for the block to be valid:

- block must not have any transactions.

- stateRoot value in stateRootCommitment must be zero (especial value)

- Must not reference any uncle.

This implies that a miner can perform validation-less mining by setting the empty flag to true, and the stateRoot to zero, if they know the gparentStateRoot value. gparentStateRoot value corresponds to the stateRoot hash of the grand-parent block. Because no stateRoot value is visible in blocks, can only be known if it computed. A miner cannot discover the gparentStateRoot value when receiving a competing block, since it will be masked by the commitment. 

A miner can mine an empty block that is child of an empty parent. The stateRoot will be empty, and the gparentStateRoot will still hold a distinct value. However, a grand-child of an empty block is forbidden, because it won’t have a new unique gparentStateRoot to refer to.

<img src="../RSKIP14/BlockChainRSKIP14.png">

 
The fact that empty blocks cannot reference uncles means that miners of empty blocks not only earn less because of missing transaction fees, but also may earn less because of missing uncle references. 

The field **height** is added to the block header. This is the block number,

The REMASC contract is not executed in empty blocks. In the first non-empty block after a row of empty blocks the REMASC will scan backwards the blockchain in order to collect the miner addresses. To easily allow this, a new opcode is added **BLOCKINFO**. This opcode receives in the stack a block number and a bit-mask and pushes back the following information depending on which bits of the mask are enabled:

1. block hash (bit 0)

2. empty flag

3. coinbase address

4. stateRoot

5. txTrieRoot

6. receiptTrieRoot.

7. logsBloom

8. difficulty

9. timestamp

10. gasLimit

11. gasUsed

12. paidFees

13. extraData

14. nonce

15. height

If the block number pushed is older than 256 blocks, BLOCKINFO halts with out-of-gas exception.

The new opcode **UNCLECOUNT** is added. This opcode returns the number of uncles included in this block. 

The new opcode **UNCLEINFO** is added. This opcode receives a bit-mask and returns the same information as BLOCKINFO. It does not however allow to return data from an old uncle, only the ones in the current block.

# **Specification**

REMASC pseudo-code

```java

public class  REMASC {

private Class Uncle {

Blockheader hdr;

Address minerCoinbase;

public Uncle(BlockHeader ahdr, Addres aminerCoinbase) {

  hdr = ahdr;

  minerCoinbase = aminerCoinbase;

}

}

uint bb;

uint rb;

Dictionary<uint,List<Uncle>> allUncles;

public void processBlock() {

  payMatureBlock(maturityBlocks);

  if (getBlockHeader(BLOCKNUBER-1).empty) // was unpaid, pay 

     payMatureBlock(maturityBlocks+1);

  if (getBlockHeader(BLOCKNUBER-2).empty) // was unpaid, pay 

     payMatureBlock(maturityBlocks+2);

  BlockHeader h = getBlockHeader(BLOCKNUMBER)

  // now collect uncle rewards

  List <BlockHeader> uncleHeaders = getCurrentBlockUncles();

  for(int u=0;u<uncleHeaders.size();u++) {

  BlockHeader uh =  uncleHeaders.get(u);

  // save the miner who included this uncle 

  allUncles.put(uh.height,new Uncle(uh,h.coinBase))

  }

}

void payMatureBlock(uint amaturityBlocks) {

  // pay the matured block

  uint matured = BLOCKNUMBER-amaturityBlocks;

  //if the previous block is an empty block, look at th

  uint fbr = fullBlockReward.get(matured)

  BlockHeader h = getBlockHeader(matured)

  List<Uncle> uncles =allUncles.get(matured);

  if (uncles.size()>0) 

    payUncles(h,uncles,fbr);

  else

    paySoleMiner(h,fbr);

  uncles.remove(matured);

}

void paySoleMiner(BlockHeader h,uint fbr) {

  Send(h.coinbase,fbr)

}

void payUncles(BlockHeader h,List<Uncle> uncles, uint fbr) {

     

  for(int u=0;u<uncles.size();u++) {

    Uncle uh = uncles.get(u);

    if (uh.hdr.paidFees>2$*$h.paidFees) {

       // broken selection rule, apply punishment

       uint punishment = fbr /10;

    fbr -=punishment;

    bb  +=punishment;

       break;

    } 

  }

  uint publishersFee = fbr / 10;

  fbr -=publishersFee;

  uint publishersFeeShare = publishersFee /uncles.size();

  // burn surplus

  bb += publishersFee-publishersFee $*$uncles.size()

  for(int u=0;u<uncles.size();u++) {

    Uncle uh = uncles.get(u);

    Send(uh.minerCoinbase,publishersFeeShare);

  }

  // now pay the mainchain miner

  Send(h.coinbase,fbr);

}
```

# **Security**

The Bitcoin backbone (BB) analysis shows that the common prefix property hold when f = 1 (complete desynchronicity)  assuming the adversary controls less than about 29% of the hashing power if there is a deterministic tie-breaking rule, such as the one proposed by DECOR+. This means that RSK blockchain with a 10 seconds average block interval is not susceptible to a common-prefix attack.

The chain-quality property is generally attacked by the following selfish-mining attack described in BB paper: Initially, the adversary works on the same chain as every honest party. However, whenever it finds a solution it keeps it private and keeps on extending a private chain. Whenever an honest party finds a solution, the (rushing) adversary releases one block from the private chain; if the private chain is depleted the adversary returns to the public chain. This maximizes the adversarial blocks in the blockchain. This attack is fully irrational and aims to prevent honest miners from adding transactions to the ledger, opposed to the selfish-mining attack of Eyal/Gun Sirer. However, this attack is also based on the fact that rushing has a benefit and the miners will chose the adversary blocks because of blockchain uses the first seen rule as tie-breaking. In RSK, because of DECOR+ selection rule, the attacker can generally force the network to accept his blocks spending a higher amount in transaction fees. Therefore it seems that the in RSK chain-quality attack is stronger at the expense of higher cost to the attacker.  If he can spend epsilon additional fees than the honest blocks, he can control the tie-breaking rule. One solution is to use a lexicographic tie-breaking rule except when one block has more than double the fees of the other, in that case the block with higher fees is chosen. This ensures that most of the time there is a conflict miners will not be earning as much as they could. However, because the synthetic block fees will be only a small percentage of the block real fees, they won’t be losing more than, say, 10% of the fees. However, if a block is mined with uncommon high fees (such as 10 times more), the the fee-rule prevails, so there is no incentive to choose another.

Now the attacker has to double the average fees to be chosen. For each of his blocks, on average he pays approximate the double than what he collects, but as his blocks always have uncles, he collects even half of that, so he’s throwing away approximately ¾ of the average block fees (not counting the fees that comes back in term of synthetic subsidy).

Another attack to chain-quality would be if the attacker always tries to extend a competing chain instead of the preferred one. If there is no competing fork, he creates one. All these attacks increase the power of the attacker as the fraction of the attacker hash rate approaches 50%.

Another way to prevent chain-quality attack is by forcing miners to include transactions by making the transaction pool on-chain, and accepting all transactions from uncle blocks to the pool. For instance, one can split transaction processing in transaction publication and transaction execution. Publication only implies that nodes can refer to the published transaction by its hash. This requires, however, that the full contents of uncle blocks are included in each block.

It would be interesting to be able to prove a transaction is not a double-spend, without showing it in full, so only the transactions in uncles that are not in the main chain are included to the on-chain memory pool. One way to achieve this is by including in the block header a new field Spends, which is a hash of p(i)=Hash(source-account(i),nonce(i)) for all transactions i. Therefore the uncle only has to include Spend, and all the transactions in Spend which are not double-spends. Those transactions can be referred in future blocks by transaction id, without the need to be fully specified. There are many additional problems, such as how to limit the amount of elements in Spend, to prevent denial of service attacks.

## New opcodes

BLOCKINFO

UNCLECOUNT

UNCLEINFO

## Samples 

Discarded

* The full block reward of a conflicting block must pay a **forward pressure fee**. This is variable and depends a difficulty adjustments occurring before the coinbase matures. If not such event occurs, this fee set to zero (more on this later). The forward pressure fee is burned.


[DECOR+]: https://bitslog.wordpress.com/2014/05/07/decor-2/

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

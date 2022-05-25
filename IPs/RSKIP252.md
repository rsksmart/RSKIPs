---
rskip: 252
title: Transaction Gas Price Cap
description: 
status: Draft
purpose: Sec, Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-06-29
---
# Transaction Gas Price Cap

|RSKIP          |252           |
| :------------ |:-------------|
|**Title**      |Transaction Gas Price Cap|
|**Created**    |29-JUN-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec, Fair |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes the limit of the transaction gas price as a multiple of the minimum gas price established in the block header. The objective is twofold: remove incentives for [fee sniping](https://bitcoinops.org/en/topics/fee-sniping/) and [fee-whale transactions](https://medium.com/r/?url=https%3A%2F%2Fwww.cs.umd.edu%2F~jkatz%2Fpapers%2Fwhale-txs.pdf), and to prevent users from accidentally draining their wallets with unintended high transaction fees.

# **Motivation**

RSK is economically protected from several known attacks involving high transaction fees, including fee sniping, fee-whale transactions and mining atomization attacks. For these attacks to be profitable in RSK, the bribe in fees must be at least 100 times higher than the average block transaction fees. While this security margin is wide, currently RSK blocks are not congested and average fees in RSK are low. Therefore the attack is practical.  We propose to invalidate any block containing a transaction whose gas price is 100 times higher than the minimum gas price of the block.


# **Specification**

A valid block cannot contain any transaction whose gas price is higher that 100 times the minimum gas price specified in the block. 

Any network node that is synchronized must block the propagation of any transaction whose gas price is higher than 80 times the minimum gas price of the latest block. Network nodes that are not synchronized do not currently broadcast transactions.




# Rationale

If during network congestion, RSK transactions reach the highest gas price allowed, this would prevent the gas auctioning and the price discovery mechanisms to work as expected. However, considering the proposed 100x allowed spread, the chances that this happen are negligible. During sudden times of high network congestion in Ethereum (i.e. in June 27 14:00), transaction fees spiked [10x](https://ethereumprice.org/gas/) in 30 minutes. These events won't cause complications related to this proposal. Miners can adjust the minimum gas price at the new maximum rate of +1%/block, so they can double the upper limit every 70 blocks (approximately 35 minutes), restoring the well functioning market exponentially fast. This is also in their best economic interest.

## Historical Data

[Historical data](https://bitinfocharts.com/comparison/ethereum-transactionfees.html#3m) on Ethereum shows that fees can double in a day. Standard fee spreads at various Ethereum websites show real world data regarding price spread. At the time of writing, a first Ethereum website [reports](https://etherscan.io/gastracker) a spread of 2X between the highest and lowest fees paid by transactions. A second website shows a similar [spread](https://ethgasstation.info/calculatorTxV.php). This website also shows a high number of transactions in the mempool offering [tiny fees](https://ethgasstation.info/), categorized as "standard". These tiny-fee transactions are probably not included in blocks for weeks, and we won't consider them in this analysis because they would have gas price below the RSK minimum allowed. We conclude that a allowing a 100 spread in RSK fees is more than enough to keep the fee market working at all times and reduce the impact of attacks.

## High-fee Transaction Propagation Limit Rule

The rule that limits transaction propagation of high-fee transactions prevents that 51% of the miners decrease the minimum gas price and cancel a large number of transactions in the mempool. This could be performed as an attack or accidentally. In both cases large cancellations means network resources were wasted in the propagation of the cancelled transactions. With the introduced transaction propagation limit, high-fee transaction will still be able to be included in blocks with high probability, even when the minimum gas price is being decreased continuously. This limit is not a consensus rule and can be modified easily in a reference node software release.



# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

## Security Considerations

This proposal reduces the risk of pre-existing security vulnerabilities. This proposal does not introduce a new fast and reliable mechanism to cancel mempool transactions, therefore it cannot be used to perform DoS attacks against the node mempools. 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

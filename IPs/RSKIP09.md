---
rskip: 9
title: Negotiated Minimum Gas Price
description: 
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-10-21
---

# Negotiated Minimum Gas Price

|RSKIP          |09           |
| :------------ |:-------------|
|**Title**      |Negotiated Minimum Gas Price |
|**Created**    |21-OCT-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

# **Abstract**

This RSKIP defines a change in the block validation rule and a change in the block header format so miners can negotiate a minimum gas price, to prevent dishonest miners from taking blockchain resources (CPU, state, storage, bandwidth) at zero cost when there are no transaction in the backlog (memory pool) paying enough fees. Also the minimum gas price prevent miner bribery attacks, and transaction fee side-channels.

# **Motivation**

In Ethereum, miners have incentives to use the network for their own private uses when there are no transactions in memory pools to profit from. One of this private use is memory time arbitration, wherein a miners acquires a chunk of blank persistent memory at zero cost to be able to sell it later at a cost. Also a minimum gas price, if obeyed by miners,  can be a huge help to wallets for choosing a transaction gas price based on transaction priority. Last, because miners only get 10% of the fees collected by each block, having a minimum gas price prevents bribery attacks against the REMASC contract. In these attacks a side-channel is used to pay for transaction fees. Only if more than 50% of the miners are dishonest and lower the minimum gas price, side-channels can be built.

# **Specification**

A new field minGasPrice is added to the block header. Each miner can vote to increase or decrease the minGasPrice up to 0.01% (1 per 10K). This allows miners to increase the minGasPrice 100% in approximately one day, assuming a block every 10 seconds.

Transactions that specify a gas price lower than the block minGasPrice cannot be included in blocks as they make the block invalid. The detection of invalid transactions can be done before processing any transaction in the block.

Nodes that forward transactions could check that the advertised gas price in a transaction is at least 1% higher than the minimum. This assures the transaction a lifetime of 100 blocks assuming a constantly increasing block minGasPrice. However this generates a tiny unfair advantage to miners as transaction collectors. Low priority transactions created by users and broadcast to the network will pay 1% more in fees than transactions included by miners. This seems not very important, as is obviously also true for Bitcoin, as miners may accept direct transactions and include them at whatever price they wish.

## Security 

Miners have an incentive not to decrease the minGasPrice below their inclusion thresholds because wallets will use the minGasPrice when creating transactions. If miners do set a very low minGasPrice value, they are incentivizing the creation of transactions that are not profitable to them. Also setting a very low minGasPrice enables users to flood the network with spam-transaction (a weak form of DoS attack), because those transactions will never be included in blocks.

On the other side, if a majority of miners try to set the minGasPrice to a very high value, users will be discouraging from using the network, so miners will be reducing their own revenue stream. This assumes there are human monitoring transaction fees and not only autonomous systems. It seems logical to think there will be a point that maximizes miners profit, yet make the network usable.  It is unknown if this maximum will correspond to $ 100 transactions, making RSK another bank settlement layer, or 1c transactions, making RSK an inclusive payment system, but this is also true without the minGasPrice, as miners naturally adjusts their fee thresholds collecting information about what users and miners do. Also there could be local maximums to the revenue function, if there are very distinct use cases for the platform, but it seems more plausible that there will be a single global maximum. 

A minority of merge miners cannot increase or decrease the price, if the majority oppose.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
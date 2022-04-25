---
rskip: 6
title: Block Size Limit 
description: This RSKIP defines changes in consensus to limit the block size. This prevents possible DoS vectors of attack.
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2016-06-22
---

# Block Size Limit

|RSKIP          |6           |
| :------------ |:-------------|
|**Title**      |Block Size Limit |
|**Created**    |22-JUN-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

# **Abstract**

This RSKIP defines changes in consensus to limit the block size. This prevents possible DoS vectors of attack.

# **Motivation**
An invalid block is defined as a block with correct PoW but is considered invalid by consensus. It is generally considered that the attack of creating invalid blocks is expensive because miners have to consume electricity to build a block that passes the PoW check. That same amount of electricity could have been spent to mine a block containing reward, and therefore the attack has at least the cost of electricity or loss reward. As RSK blockchain is merge-mined, mining RSK blocks have very little additional electricity cost. The loss of reward will be low during an initial period of the platform. Therefore, in RSK, the miners could try to perform attacks that are normally economically discouraged in other blockchains. Miners that do not participate in the normal merge-mining of RSK could decide to attack the RSK blockchain by mining invalid blocks. A single attack has no longstanding consequences, but if sustained, an attack can delay the creation of honest blocks. 

# **Specification**

This RSKIP proposes the following changes:

* The maximum block size is limited to gasLimit/68.
* The zero discount rule in the transaction data field is removed.

A miners-chosen block size limit may require the transaction selection algorithm to be much more intelligent when the algorithm hits both the gas limit and block size limit frequently. Therefore this limit should be set automatically to a high value based on gasLimit. For a 4M gasLimit, If a block had a single transaction containing data, the block would be 59 KBytes, or 5.9 KBytes/sec with a 10 sec block interval (68 gas/byte). If the block has a transaction containing only zeros as data, the block maximum size would be 1 MByte. If the block is full of simple transactions (100 bytes each), the block would be 19 KBytes. (21K gas/tx). The zero compression discount should be removed because RSK does not perform any compression. Therefore the block can be limited to contain no more than gasLimit/68 bytes.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


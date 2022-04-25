---
rskip: 46
title: Block Mining Fees Information Mechanism
description: 
status: Adopted
purpose: Usa
author: MM <martin@iovlabs.org>
layer: Node
complexity: 1
created: 2017-10-04
---

# Block Mining Fees Information Mechanism

|RSKIP          |46           |
| :------------ |:-------------|
|**Title**      |Block Mining Fees Information Mechanism|
|**Created**    |04-OCT-2017 |
|**Author**     |MM |
|**Purpose**    |Usa |
|**Layer**      |Node |
|**Complexity** |1 |
|**Status**     |Adopted |

# **Abstract**

Miners earn fees for every block they mine. The value of the fees they receive in their account is calculated by REMASC pre compiled contract. 

In RSK node current version, miners will see their account balance increased every time they mine a block but there is no easy way to understand how does fees were calculated. This can be even worse if multiple blocks are mined in a short window of time. The main problem here is that matching between block and it’s corresponding fee must be deduced from REMASC contract.

This document presents a solution to clearly expose the fees per block information to miners.



# **Specification**

See discussion [here](https://github.com/rsksmart/RSKIPs/issues/83).


**SubmitBitcoinBlock method**

Response should return:

Status. can be any of the following values: imported_best, imported_not_best, exist, no_parent, invalid_block.

Blockhash. RSK block hash for submitted block.

Height. Blockchain height where the block was added. Only returned when Status result is imported_best or imported_not_best.

**Information log**

Log should be done after the balance is transferred to the miner address. Destination address and blockhash should be topics of this log while amount can be data.

**Interface for miners**

(1) MinerServerImpl.java should have a blocksForFeesInfoCollection of blockhash and inputHeight. On a call to "getFeesInfoForMinedBlocks" each element of the previously mentioned collection will trigger a search for the fees paid. That search consists of looking for the block where the fees were paid, that would be at heigth = inputHeight + maturity. Once the block is found ask for the tx receipt of the REMASC transaction and search its logs by blockhash. Add blockhash and amount to a resultCollection. Once all the elements of the blocksForFeesInfoCollection were iterated empty the collection and return resultCollection.

The "addBlockForFeesInfo" will add the values recceived by parameter to blocksForFeesInfoCollection.

(2) This new piece of software must expose two API methods. One of them is "getFeesInfoForMinedBlocks" which can use the JSON-RPC method “[eth_newFilter](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_newfilter)” to create the desired search filter when first called. The filter must filter using the miner coinbase/payment address. Once the filter is created, a call to the method “[eth_getFilterChanges](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getfilterchanges)” will bring all the matching logs. For every log, blockhash and amount must be extracted and returned as the method result.

The method "getBlockFeesInfo" will receive a blockhash and inputHeight. With that information plus the RSK network maturity value a new filter object must be created. This object will filter by topic = blockhash and look only at blockchain height = inputHeight + maturity. Recently mentioned object will be the input of the JSON-RPC method “[eht_getLogs](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getlogs)”. For every log returned, blockhash and amount must be extracted and returned as the method result.

(3) Expose same methods on mnr interface as detailed in (1). Implement the search by using "[eth_newFilter](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_newfilter)", “[eth_getFilterChanges](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getfilterchanges)” and “[eht_getLogs](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getlogs)” as shown in (2). Be aware that from the RSK node code the java implementation for those methods can be used instead of going through JSON-RPC.



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
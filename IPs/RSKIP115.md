---
rskip: 115
title: Removal of Unused Headers from the Bridge Contract
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2019
---
# Removal of Unused Headers from the Bridge Contract

|RSKIP          |115           |
| :------------ |:-------------|
|**Title**      |Removal of Unused Headers from the Bridge Contract |
|**Created**    |2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

Currently the Bitcoin network generates 52K headers per year.  The RSK Bridge currently accumulates 500k Bitcoin headers. These headers are required to find the best chain and to proving inclusion of transactions. However after a certain period has passed (i.e. a month) proving inclusion and re-oganizing the blockchain seems unprovable.  Therefore, we propose that at certain height with timestamp T, corresponding to a network upgrade, all headers prior  T-(1 month) are partially removed. One every 100 headers is kept. The headers kept are conceptually moved to a different list, the oldList. A new Bridge method to register a BTC transfer transfer for very old headers is added. This new method can lookup a header in the oldList, provided the user shows a list of headers that are children of a header in the oldList. Other future methods that allow proving the inclusion of any transaction (i.e. for BTO) must are also extended to support old headers.

# **Motivation**

Reducing the time it takes to download the state by fast sync is paramount. Currently the Bitcoin headers stored in the bridge account for 90% of all state data (**check this**). Therefore we can trim the state and reduce downloading time by removing unnecessary and unused headers, yet keeping the same functionality in the Bridge. 

# **Specification**

A new method is added to the Bridge: 

registerOldBtcTransaction(bytes btcTxSerialized, int height, byte pmtSerialized, bytes firstHeader, bytes connectHeaders) 

- firsHeader must represent a 80-byte Bitcoin header. 
- connectHeaders must be a height-increasing list of headers without the 32-byte parent value (which can be derived from the previous header). Therefore each record in connectHeaders is 48 bytes in length. The last header in connectHeaders must be the parent of a header with (height % 100==0) that is stored in the oldList vector.
- the transaction btcTxSerialized must be included in firstHeader.
- the height corresponds to the height of the firstHeader.
- If (height % 100 ==0) then both firstHeader and connectHeaders[] should be empty.
- The maximum length of connectHeaders is 98.
- No verification of the validity of the headers is made (difficulty, timestamp, etc.) except to check that each header is a child of the next one and the last is a child of the checkpoint in oldList.

Every time receiveHeaders is called, old headers (older than 1 month) should be either removed or moved to oldList. The headers moved to oldList are the ones with (height %100 ==0).

**TO DO**

It is important that the user is notified somehow if the operations registerBtcTransaction or registerOldBtcTransaction have failed, and in that case, why. 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

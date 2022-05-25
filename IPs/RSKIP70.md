---
rskip: 70
title: Default TX Data
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-11-25
---
# Default TX Data

|RSKIP          |70           |
| :------------ |:-------------|
|**Title**      |Default TX Data|
|**Created**    |25-NOV-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

Ephemeral Data [RSKIP28] allows transaction to send information to the blockchain, where the information  is removed by default. In this RSKIP we present a new mechanism for transactions to send data that will persist by default, but can be removed on request. The benefit is for the creation of on-chain state-channels that don’t suffer from the risks of parties dropping off from online monitoring.

# **Motivation**

State channels allow multiple parties to perform an off-chain protocol while requesting on-chain arbitration if one of the parties fails or tries to cheat. The drawback of state channels is that if one party loses Internet connection for long, the other may try to cheat her. To prevent this, we create a new mechanism based on the Shrinking-chain Scaling Framework that allows transactions to specify, in a new field, data that, if there are no indication to remove, will be persisted by default

# **Specification**

**Using Ephemeral Data in a Contract**

Any contract can use Default  data by executing the opcode DDLOAD. The opcode first argument is the ID (hash-digest) of the Default data chunk, the second being the offset and size of the part to retrieve. To retrieve all, the offset should be set to zero and the size should be set to the value returned by DDSize. Contracts can obtain the ID of a Default chunk in a transaction by executing the opcode DDID. This opcode pushes the ID of the Default  data of the current transaction into the EVM stack. If the transaction has no Default data, it pushes the all-zero value. DDLOAD fails if a contract tries to load Default data marked as Delayed before the second block after inclusion. It can, however, store the ID value for future use, or receive it later in another transaction payload. The DDSize receives an ID and pushes on the EVM stack the size of such object. If the object is not present, then the resulting value is all zeros.

You can see that these opcodes (DDSIZE, DDLOAD and DDID) are similar to the opcodes used to access Ephemeral Data (EDSIZE, EDLOAD and EDID). However Default data has a new opcode, DDREMOVE, which receives the id of the data and removes the data from the blockchain, It returns 1 if the data was removed successfully, 0 if it was not because the data does not exists. Users should note that it’s impossible to distinguish between data that was removed and data that never existed in the first place: in both cases DDREMOVE returns zero.

One possibility is to unify Ephemeral and Default data opcodes into a single set.

  

**New Transaction Fields**

A transaction that uses Data data can specify three new fields:

DDSize (uint) specifies the Ephemeral data size. This is required because the cost of the transaction depends on this size. If not specified, it’s assumed to be 32 bytes.

DDDelayed (boolean) indicates if this data is available immediately or it’s delayed 2 blocks. If not specified, it is assumed false.

DDBlockCount (uint): specifies how many blocks the data will be become permanent and won’t be able to be removed anymore. The default count must be lower than a certain fixed bound. If Default data is not used (by DDLOAD) before the deadline, the Default data can be removed safely from the blockchain forever. However the actual removal from memory in full nodes should occur several thousands of blocks later, until it’s clear that the block will not be reverted by a chain reorg. If this field is not specified, it is assumed to represent 24 hours in blocks.

**Considerations**

It must be noted that if Default Data (and Ephemeral data) was blockchain-wide, and not confined to certain sender account or destination contract, then several transactions can at different times send the same ephemeral data and smart-contracts using the data may interfere with each other. For example, if a certain data is loaded by DDLOAD by certain contract, it will become persistent for any other contract, forever. This may not be desirable, as it creates a new way to store information, not subject to storage rent. Also it may interfere with efforts to parallelize transaction processing, as shared objects must be detected.

So we make Default Data (and Ephemeral Data) contract specific. In this regard the persisted ephemeral data and the non-removed default data would be part of the storage of the contract, and would be forever accounted for storage rent. This is an obvious drawback. To we should extend DDREMOVE to EDREMOVE and make all default/ephemeral data removable. Internally, ephemeral and default data records will be stored in two distinct tries, whose root hashes would be part of the Account record. Internally each record in the trie will contain both the data hash, the "creationTimeStamp" field and the “blocksToLive” field. If the current block is greater than creationTimeStamp plus blocksToLive, then it means the data has been removed. If must be noted that the record is not removed from the trie when the data is removed by default (in case of Ephemeral Data). The record will be removed only if EDREMOVE is called. EDSIZE therefore should distinguish between the fact that data is not there, and the record is not there (e.g. returning 0xfff.fff is the record is present, but not the data).

The existence of specific tries for default and ephemeral data is against the spirit of the functionality: providing very cheap temporary storage. An entry on storage costs 20K, so the transaction cost will double with the presence of ephemeral data. We can however safely reduce the cost to 5000 gas (the cost of SSTORE for a non-new value) plus a fixed cost per byte, because the data we’re storing will not live much time, and storage rent will be paid.

One of the solution for ephemeral storage would be not to use a trie, but to force the contracts to copy the content before the data becomes extinct. So EDLOAD would bring the data, and the contract would SSTORE the data in somewhere else.The drawback is that it becomes difficult to bootstrap a node from the state (wrap sync), without the existence of some commitment in the block header on what the current state of ephemeral/default data is.

**Possible solution**

Another solution is to update the state only once every N blocks (a key block), for a certain N (e.g. 100 to 1K), so that some ephemeral records would have gone by then, and the storage cost is reduced. Therefore nodes would sync always from a key block. All ephemeral data between key blocks would be kept in memory. In this case the Ephemeral and Default tries would be stored separately in a trie accessed by contract address, that only changes one very 1K blocks. Executing 1K blocks should not take more than 1K seconds or 16 minutes. Having a key block every 100 blocks seems better, as it decrease the disk access cost to 1%, but still synchronizing requires only the execution of 100 blocks, or about 1.6 minutes.

We therefore set a cost per byte for ephemeral data of 4, and a fixed cost of 50.

We set a cost per byte for default data of 68, and a fixed cost of 200.

**Still Another solution if storage rent is implemented**

Another solution is to just lower the initial cost of storage, because it really doesn’t account for disk access if blocks are stored on disk once every 100 blocks or so, and because we’ll add storage rent. If a CALL costs 700, and it requires to retrieve the whole code to memory, why should a storage retrieval costs 5000 ?

Therefore we lower byte cost to 20 or so, and lower the SSTORE / SLOAD costs something like to 2000 and 1000.

Having these costs means that a 256 byte ephemeral message can be broadcasted to the network at a cost of 7K gas, or 30% more than the standard transaction cost. If an onchain transaction costs 5c, then it’s not that much. Combined with LTCP, the cost could go down to 2c.

[RSKIP28]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP28.md	


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
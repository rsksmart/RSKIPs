---
rskip: 25
title: Memory caches
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-12-27
---

# Memory caches

|RSKIP          |25           |
| :------------ |:-------------|
|**Title**      |Memory caches |
|**Created**    |27-DIC-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Pre-git revisions

Date: 25/NOV/2016

Revision date: 01/JAN/2017

Revision: 3

Status: Draft

# **Abstract**

Storage rent has shown to be impractical, since most rent payments are microtransactions. This RSKIP defines another incentive structure for reducing the state size: the use of a hierarchy of caches.
Motivation

The blockchain should try to keep the state limited in size to allow fast transaction processing. Limiting via storage rent is expensive. We can move contracts that are not frequently used to a second layer cache, and increase the cost of accessing such cache. This allows us to decrease considerably the cost to access accounts/contracts in RAM (e.g. 40 gas), use a low limit of calling an account/contract in SSD (400 gas) and a higher cost for calling an account/contract in HDD (4000 gas). 

The main problem is how to move data efficiently from one cache to the other. One way to do this is by tagging each node of the tree with the timestamp of the last access. Every N blocks, a process scans all the tree nodes in memory and sends to SSD the unused ones.  Every 10*N a process scans all the nodes in in SSD and sends to HDD the unused ones.
A better approach would be to target certain amount of RAM, and scan nodes when half of the target RAM has been filled. This would work like a garbage collector, that is executed only when needed.

## Discussion

How much would a  garbage collection process take?

Suppose that the RAM limit chosen is 1 GB, then clearly scanning 1 GB of data and writing 1 GB of data to disk should not take more than a few seconds.
Suppose that the SSD limit is 128 GB. Then scanning through 128 GB can take several minutes. A data structure could be maintained to keep ordered the accounts accessed, from oldest to newest. A priority queue will work, having logN access, add and removal times.
However one problem may be that even if some data is accessed less frequently, if it is a big chunk of data, maybe the best strategy would be to keep it in SSD?
I think the best would be just to have two levels: RAM and SDD, 

### Proposed Solution 1

A number of trie nodes are stored in memory. For the nodes that are not stored, a dummy node is stored, which only contains the hash of the data that should be there, and the last access time. Whenever a node is brought to memory, the lastAccessTime field of that node and all parent nodes are updated. Also the totalRAMConsumed global variable is updated, adding the amount corresponding to the loaded node. If at the end of the processing of a block the totalRAMConsumed is upper than half of the maximum value, then the garbage collector is run.
The garbage collectors scans all terminal nodes in memory, and creates a table (node-ptr,size,lastAcceddTime). Then the table is sorted from oldest access time to newest, and nodes are removed from the tree starting from the oldest until the amount of memory used is less than a quarter of the totalRAMConsumed. During the removal process, if a node has two children that have been removed, then the parent node also is.

New gas costs. Since contract storage are leaf nodes, no account can be removed if all the storage addresses have been removed. Accessing a node in RAM will cost 40 units of gas, while accessing a node in SSD will cost 400 units.

This solution is complex and requires a lot of housekeeping. A better approach is letting the user decide.

### Proposed Solution 2

Contracts specify if they wish to be stored in RAM and eventually SSD, or only in SSD. To be stored in RAM, they have to pay a high fixed cost (or recurrent cost in case storage rent is used). A contract can switch from RAM to SSD and vice-verse at any time, if programmed to do so. This is the solution chosen for this RSKIP.

# **Specification**

A contract (in full) can be stored in RAM or in SSD. When a contract is stored in RAM, the CALL cost is reduced 5 times to 140 (instead of 700) and the SLOAD/SSTORE costs are also reduced by 5 (400 for a 2K SSTORE cost). To move a contract to RAM, a new opcode is used SETSTORAGE. The only argument for this opcode is the destination: 0 is SSD, 1 is RAM. If the argument is equal to the current storage state, nothing happens, and the opcode costs 10 gas.  If the movement is from SSD to RAM, it has a proportional to the size of the contract, but with a very low multiplier. If  the movement is from SSD to RAM, the cost is proportional as specified:
- 700  base cost (to bring code to RAM, plus additional page cost if paging is used)
- the equivalent of reading each storage cell by SLOAD. 

The recurrent cost is also adjusted by a 5x multiplier. 
The reduction in cost for SSTOREs is based on the assumption that this system caches RAM data and only writes it to SSD every N blocks (e.g. N=60, but a minimum of N=5 is assumed). This means that the cost of storage writes is amortized.  If a full node is turned off, or the application exits abnormally, then the full node can always reconstruct the state by re-executing up to N past blocks.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
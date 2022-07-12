---
rskip: 58
title: Handling Bitcoin Forks
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-11-14
---

# Handling Bitcoin Forks

|RSKIP          |58           |
| :------------ |:-------------|
|**Title**      |Handling Bitcoin Forks |
|**Created**    |14-NOV-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**


This RSKIP discusses how a future Bitcoin fork affects the RSK platform and discusses different plans to move forward in case of a future Bitcoin fork. Finally, it suggests the inclusion of a special method to recover funds in smart-contract in case of a unsupported fork.


# **Specification**

Parallel Bitcoin forks, such as Bitcoin Cash or Bitcoin Gold, are gaining popularity. The idea that a parallel Bitcoin fork would have any value is relatively new. A couple of years ago creating a parallel Bitcoin fork was considered a plain scam and wouldn't receive much attention. Some core developers thought about hard-forking Bitcoin to achieve a supposedly better Bitcoin, but not about having two competing “Bitcoins”. This changed in 2017, as a part of the Bitcoin community created the Bitcoin Cash split, whose value not only resisted large sells but kept growing over he following months. Therefore the ideal single Bitcoin scenario cannot be assumed anymore. RSK is currently a Bitcoin federated side-chain, with a hybrid SPV-based transfer BTC->SBTC transfer system. There are two problems: the first is that the SPV peg follows the longer chain, as long as the headers are well-formed according to the Bitcoin protocol, no matter what chain the RSK community wants it to follow. The second problem is that, in case of a persistent valuable parallel fork, the RSK bitcoins held in custody by the Federation cannot be easily returned to their lawful owners. This is because some SBTC may be held in smart contracts that have no explicit owner. To mitigate the first problem, we at RSK will inform the community of any upcoming fork. If we see a huge demand to push the RSK platform to follow a specific fork, contrary to what the majority of hashing power is going, we’ll try to coordinate with the community one of the following options:
    1. Create a new RSK blockchain as a side-chain to one of the forks, and keep both.
    2. Create a new bridge between the the RSK blockchain and the un-bridged fork, add a new native cryptocurrency and duplicate the account state space into two parallel account spaces, each using its own cryptocurrency. 
    3. Follow one fork and allow users to withdrawal the cryptocurrency corresponding to the other fork.

The second option, while the most interesting from the point of view of RSK, is the most technically challenging. For the third option, we suggest that all smart contracts created in RSK have the following public method (as specified in Solidiy language):

function public getOwnerAddressInFork() returns (bytes20 address);

Here the address must be a standard 20-byte P2SH address in binary.

This of course gives the creator of smart-contract a certain advantage over all the remaining users, because he may be able to get the cloned balance on a fork.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
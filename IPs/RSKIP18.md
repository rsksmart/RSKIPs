---
rskip: 18
title: Fast Hibernation Wakeup using Trie
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-09-28
---

# Fast Hibernation Wakeup using Trie

|RSKIP          |18           |
| :------------ |:-------------|
|**Title**      |Fast Hibernation Wakeup using Trie |
|**Created**    |28-SEP-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Pre-git revisions

Date: 29-09-2016

Revision: 1

Status: Draft (unfinished)

# **Abstract**

This RSKIP defines a mode of the opcode WAKEUP to wake up contracts consuming less gas. 

# **Motivation**

[RSKIP17] defines opcodes tailored to hibernate and wake up contracts (HIBERNATE and WAKEUP). That RSKIP proposes that when hibernation occurs, the hash of the persistent storage trie is hashed along the code hash and the resulting hash is persisted for later wake up. To wake up the hibernated contract, an RLP description of the trie must be given as WAKEUP argument. This requires the WAKEUP opcode to parse the RLP data and rebuild the tree. There are two problems with this approach:

1. The cost of the WAKEUP opcode would depend on the complexity or size of the RLP data. 

2. Since contract volatile memory has a quadratic cost, filling the volatile memory with the RLP description may be very costly.

##  Discussion

To prevent using volatile memory, we can let the WAKEUP opcode specify that the persistent contract memory is copied on hibernation. Copying the whole persistent memory tree from one contract to another could be as fast as duplicating a node in the world state trie. Another solution is to change the protocol so that volatile memory cost is not quadratic, but linear.

# Proposed solution

When WAKEUP receives 0xff..ff (32 bytes) as the root trie, and zero length, as arguments, it means that the whole trie of storage must be moved from one contract to the contract that is woken up. Since memory is moved (not copied), it does not pay the SSTORE cost.
 
# **Specification**

### WAKEUP

Arguments: contract_address code_address code_size trie_address trie_size

Value: 

if trie_address is 0xff...0ff the cost is m*a+c

if trie_address is other, the cost is m*(a+b)+c, 

Value accepted: 6 month of rent.

Returns: error_code

[RSKIP17]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP17.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
---
rskip: 243
title: Intra-transaction  Gas Refunds
description: 
status: Draft
purpose: Sca, Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-05-16
---
# Intra-transaction Gas Refunds

|RSKIP          |243           |
| :------------ |:-------------|
|**Title**      |Intra-transaction  Gas Refunds|
|**Created**    |16-MAY-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca, Fair |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes eliminating inter-block gas refunds and replace them with a 100% intra-transaction gas refunds. It also proposes to reimburse the user for storage and contracts that are created but cleared in the same transaction avoiding requiring state trie updates and storage operations, increasing platform fairness.

# **Motivation**

The motivation for the removal of storage refunds can be found in [EIP3298](https://eips.ethereum.org/EIPS/eip-3298). Essentially, gas refunds that span over blocks bring numerous problems. However, instead of removing storage gas reimbursements completely, we propose keeping them but only if the storage creation and removal occurs within a single transaction. By doing so we reduce the gas cost of common Solidity language constructions, such as reentrancy semaphores. This change improves the platform fairness because only performs refunds when the state trie is not impacted, and the I/O cost is avoided. 

Currently, when a contract is destroyed, the reimbursed amount is constant. However, the variable cost related to the number of bytes installed can be very high. We propose to reimburse 99% of this amount, which also increases the platform fairness.

Because intra-transaction refunds cannot increase block processing time, the cap to 50% of the consumed gas is removed.

# **Specification**

Starting from an activation block (TBD) storage cleanup refunds are modified both for `SSTORE` and `SELFDESTRUCT`, and the cost of `SSTORE` is also simplified. Refunds can only be collected in the same transaction the gas is spent. 

A map of sets *storage_cells_touched\[contractAddress\]\[storageAddress\]* is created in each call frame.
The map is empty when each call begins.
If a call reverts/OOG, the map is discarded.

Each time `SSTORE` is called, its cell address is looked up in *storage_cells_touched* in the current call frame. If it exists, then the cost of `SSTORE` is 400. If it doesn't exists, then it is 20000. 

When a call returns without reverting changes, the contents of *storage_cells_touched* is merged into the caller's map.

At the end of transaction processing, all storage cells that exists in the top-level *storage_cells_touched* are scanned.
For each cell, if its value is equal to their pre-execution value, then 19600 gas is refunded.

Also at the end of transaction processing, if a contract was selfdestructed and the contract did not exist prior to execution, then 24K gas (out of the 32K originally paid), plus 99% of the variable cost required to install the code (198 per byte), is refunded.

The cap of the refund to 50% of the transaction gas consumed is removed.

Note that if the same cell is overwritten in recursive calls, the `SSTORE` operation may be charged 20000 more than once. The refund will be applied only once. However the recursive overwrite of cells is a very uncommon pattern.

# Rationale

Several related topics are analyzed in separate sections.

### This proposal vs. [EIP2929](https://eips.ethereum.org/EIPS/eip-2929)

This proposal does not overlap with EIP2929 and both proposals can be combined. Both proposals attempt to achieve a closer match between gas costs and actual resources consumed.
This proposal refers to write operations, while EIP2929 refers to read operations. Even if both proposals use maps or sets to hold storage keys, these maps serve different purposes. EIP2929 aims to reduce the gap between storage access gas cost and actual access cost. This proposal, storage write gas cost and actual write cost. However, state rent encompass the incentives in EIP2929 and extends beyond them to consider intra-block caches, while EIP2929 focuses only in inter-block caches. Therefore we suggest to combine this proposal with state rent, instead of EIP2929.


### Reentrance protections

It is common that contracts use a storage cell to create a semaphore to protect contracts from unexpected reentrancy. In this pattern, the semaphore is incremented and decremented in the same call frame. The current cost of a simple semaphore is 20000 for the increment, then 5000 for the decrement, with a refund of 15000 (considering the transaction consumes at least 30K gas), resulting in 10000 net gas cost in the best case, and 25000 gas in the worst case.  With the scheme proposed here, the cost of a semaphore becomes only 800 gas (20K for increment, 400 for decrement, and 19600 refunded).

### Inexistence of a map to track contracts created 

It's not necessary to create a new map to track contracts created and destroyed during transaction processing (like cells) because RSK does not allow the creation of a contract with the same address as one destroyed in the same transaction (nor in the same block) per [RSKIP131](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP131.md).

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

## Security Considerations

No new denial of service or resource abuse risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

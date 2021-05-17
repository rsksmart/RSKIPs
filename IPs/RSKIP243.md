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

This RSKIP proposes eliminating inter-block gas refunds and replace them with a 100% intra-transaction gas refunds. It also proposes to reimburse according to the paid cost, increasing platform fairness.

# **Motivation**

The motivation for the removal of storage refunds can be found in [EIP3298]([https://eips.ethereum.org/EIPS/eip-3298). Essentially, gas refunds that span over blocks bring numerous problems. However, instead of removing storage gas reimbursements completely, we propose keeping them but only if the storage creation and removal occurs within a single transaction. By doing so do not harm common Solidity language constructions, such as reentrancy semaphores. This change keeps the platform fair because on refunds the trie is not impacted, and the I/O cost is avoided. 

Also currently when a contract is destroyed, the reimbursed amount is constant. However, the variable cost related to the number of bytes installed can be as high as 4915200 gas. We propose to reimburse this amount, increasing platform fairness.

Because intra-transaction refunds cannot increase block processing time, the cap to 50% of the consumed gas is removed.


# **Specification**

Starting from an activation block (TBD) storage cleanup reimbursements are modified both for `SSTORE` and `SELFDESTRUCT`. Refunds can only be collected in the same transaction they the gas is spent. 

Two data structures are created by the virtual machine when processing a transaction: a set *contracts_created\[contractAddress\]* (one for all contracts) and one map of sets *storage_cells_created\[contractAddress\]\[storageAddress\]*. Both are empty on creation.

If a contract is created, it is added to *contracts_created*. If a contract is destroyed, the contract address is looked up in *contracts_created*. If it exists, then 24K gas plus the refund of the variable cost required to install the code is refunded at the end of transaction processing. 

If a storage cell is created (zero value before) its address is added to the set *storage_cells_created\[contractAddress\]*, where contractAddress is the current contract. If in the same transaction the same cell address is cleared (zero value afterward), a refund of (*creationCost* -3000) is added to the refund amount, and the storage cell address is removed from *storage_cells_created\[contractAddress\]*. *creationCost* may be the 20K established at RSK genesis, or a new cost established by other RSKIP. If *creationCost* is variable, then the reimbursement will also be variable to match *creationCost* .

If a contract call revers, the changes on *contracts_created* and *storage_cells_created* are not reverted. This implies that clearing a storage cell may trigger a refund even if the cell was already cleared. 

The cap of the refund to 50% of the transaction gas consumed is removed.


# Rationale

Several related topics are analyzed in separate sections.

### This proposal vs. [EIP2929](https://eips.ethereum.org/EIPS/eip-2929)

This proposal does not overlap with EIP2929 and both proposals can be combined. This proposal aims to reduce the gap between storage allocation gas cost and storage allocation real cost. EIP2929 aims to reduce the gap between storage access gas cost and real access cost. However, state rent includes incentives in EIP2929 and extends beyond them to consider intra-block caches, while EIP2929 focuses only in inter-block caches. Therefore we suggest to combine this proposal with state rent, instead of EIP2929.

### Rollbacks

This proposal avoids rolling back the changes in *contracts_created* and *storage_cells_created* on OOG or REVERT. The reason is that if these changes need to be reverted, then one copy of each map must exist per call frame, and lookups cease to be constant time. While not reverting changes leaves side-effects, since the program has already paid for the storage creation (even if reverted later), the refund cannot be used to create gas.

### Reentrance protections

It is common that contracts use a storage cell to create a semaphore to protect contracts from unexpected reentrancy. The current cost of a simple semaphore is 20000 for the increment, then 5000 for the decrement, with a refund of 15000 (considering the transaction consumes at least 30K gas), resulting in 10000 net gas cost.  With the scheme proposed here, the cost of a semaphore becomes only 8000 gas (20K for increment, 5K for decrement, and 17K refunded).

### Removal of Contract Address from *contracts_created*

It's unimportant if the address of the contract destroyed is removed from *contracts_created*, because RSK does not allow the creation of the same contract in the same transaction (nor in the same block) per [RSKIP131](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP131.md).

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

## Security Considerations

No new denial of service or resource abuse risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

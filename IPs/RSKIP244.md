---
rskip: 244
title: Variable Storage Costs
description: 
status: Draft
purpose: Sca, Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-05-16
---
# Variable Storage Costs

|RSKIP          |244           |
| :------------ |:-------------|
|**Title**      |Variable Storage Costs|
|**Created**    |16-MAY-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca, Fair |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes reduced the gas cost when creating short storage cells and removing storage reimbursements. Reimbursements can be maintained if this RSKIP is implemented jointly with RSKIP243.

 

# **Motivation**

Currently the cost of creation of a new storage cell is 20k gas. However a cell with short key and value occupies on storage 13% of a cell with 256-bit key and values. The main reason is because storage cells of 44 bytes or less are embedded in parent trie nodes, instead of requiring an independent trie node. This RSKIP proposes that the cost of storage is variable depending on its size, and cells that can be embedded in parent nodes pay 50% lower initial cost.  At the same time, this RSKIP removes the storage reimbursements when storage is created in a separate transaction. These reimbursements have shown to be problematic when used for gas arbitration (gastokens).

The motivation for the discount for short cells is Financial inclusion, which requires the ERC20 contracts to store millions of ledger records for stable coins.  The RSK platform already optimizes storage of trie nodes whose size is lower than 44 bytes. An ERC20 balance record in storage generally requires less than 44 bytes in the trie node (20 bytes of address and 12 bytes of balance, with 12 bytes of node/key overhead). However, Solidity hashes the key with Keccak and this prevents using the shorted key. By using operation for direct storage access, such as sstore()/sload(), it's possible to take advantage of the optimization without colliding with other storage field variables.


# **Specification**

Starting from an activation block (TBD), the cost of SSTORE a non-zero value is computed as follows:

```Solidity
totalSize = address.size + data.size;
If (totalSize  <= 32 )
   creationCost = setResetCost/2+totalSize*30;
else
   creationCost = setResetCost+totalSize*30;
```

The *setResetCost* is defined as 15K if the cell value was previously zero, and 7500 if the cell value was non-zero.

The fixed cost to `SSTORE` a zero-value is fixed at 3000. 

Storage cleanup reimbursements are removed both for `SSTORE` and `SELFDESTRUCT`, unless this RSKIP is implemented together with RSKIP243, which is recommended.


# Rationale

Several related topics are analyzed in separate sections.

### Reentrance protections

It is common that contracts use a storage cell to create a semaphore to protect contracts from unexpected reentrancy. Using a semaphore, the current cost of such method is 20000 for the increment, then 5000 for the decrement, with a refund of 15000 (if possible), resulting in 10000 net gas cost. Using a 1-2 semaphore, the current cost is 5000 for increment and 5000 for decrement, also resulting in 10K net gas cost. 

**With refunds**

If this RSKIP is combined with RSKIP243, the the cost of a 0-1 semaphore becomes 7502+3000-4502, resulting in 6000, 40% lower. 

**Without refunds**

Even if we removed storage refunds completely, with the scheme proposed here, the cost of a 0-1 semaphore is 7502+3000, resulting in 10002. Therefore the cost of semaphores is almost equal than before. A 1-2 semaphore will cost 3752+3752=7504, which is represents a 25% cost reduction. In the future, state rent will provide a cost decrease for 0-1 semaphores afterwards and will level both methods.

### Witness size

While the possibility to create a state-less client for RSK has not been analyzed in-depth, any discount on SSTORE or SLOAD operations increases the potential witness size. The changes proposed in this RSKIP lower the SSTORE cost, but has little effect on witness data, since SSTORE is much more expensive than SLOAD. 

### Shared Key paths

The actual space used by a trie node does not directly depend on the key size, because if multiple storage cells share a large part of their keys, only one copy is stored in a shared key path, however taking into account the trie structure would require inspecting it at the time of SSTORE, which would be expensive. 

### Storage cleaning reimbursement removal

While these changes could be made compatible with storage reimbursements, by reimbursing 50% of the amount paid for storage, storage reimbursements are removed by this proposal. Instead, we propose that in the future each contract keeps a counter of the number of storage cells contained, or the platform uses the node size field, to add a variable cost to all SSTORE and SLOAD operations depending on the tree size. Contracts would be incentivized to be programmed to remove unused cells to make all other operations cheaper.

### Optimized ERC20 contracts

It's possible to use this cost change to reduce gas costs of transfer operations in ERC20 contracts. However, the man benefit is when transferring tokens to addresses that do not contain any tokens. Transferring tokens between existing addresses will not be significantly affected.

Solidity mappings cannot take advantage of the incentive for using short storage cells. This is because Solidity always expands the key of a mapping to a uint256 by applying a hash function. The Solidity documentation: (https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html) states:

> The value corresponding to a mapping key k is located at keccak256(h(k) . p) where "." is concatenation and h is a function that is applied to the key depending on its type:  (note: p is considered a uint256)
>
> * For value types, h pads the value to 32 bytes in the same way as when storing the value in memory.
>
> - For strings and byte arrays, `h` computes the `keccak256` hash of the unpadded data.
>
>   
>
> If the value is again a non-elementary type, the positions are found by adding an offset of keccak256(k . p).

Solidity uses Keccak256(..) for storage keys because it uses adjacent cells when the value in the mapping occupies more than 32 bytes. It can't use simply (k. p), because an attacker could choose the key k=(u+1) for a pre-existing key u, in order to store a certain value overlapping the next 32 bytes of the data stored at key u. 
But because ERC20 contract store uint256 balances (one cell),  the balance mapping doesn't need this protection as long as the address space does not collide with the address space of a Solidity native contract field.
Instead of using Solidity mappings, the program can use SSTORE/SLOAD with a user-defined key mapping such as: `<addr20> 0xff`. This mapping does not collide with Solidity mappings because Solidity mapping storage keys are always the hash digest of 64 bytes of data, and the proposed pre-image is only 21 bytes in length. However, it can collide with a field variable at slot 0xff if addr20 is all zeros. The reason is that the EVM always expands the keys to 256 bits RSK, filling the MSBs with zero. For such as contract field to exists the programmer must define 255 contract fields, which is uncommon. We could avoid potential problem this by adding a non-zero byte before addr20 like this: `0xff <addr20> 0xff`. If we ditch the trailing 0xff, then it can't collide also. So a final proposal is simply: `0xff addr20`. 

By using the proposed scheme to store an ERC20 balance the platform saves:

* 32 bytes for the trie node hash
* 11 bytes of storage cell key
* 2 bytes of a node flags and path length
* At least 32 bytes of LevelDB overhead 

The total saving is 32+11+2+32=77 bytes. When stored embedded in a node the space used by a storage cell is: 20 bytes (key) + 12 bytes (value) + 10 bytes (hashed key prefix) + 2 bytes (node flags and path length) = 44 bytes. When not embedded the space required is 44 + 77 = 121 bytes.
Therefore, the cost for using small nodes for balances can be as low as 36% of the current cost. 

To conclude, the storage cost of an ERC20 token transfer is approximately 10K gas (with pre-existing non-zero balances). With this proposal, an optimized ERC20 contract would cost 9420 gas. However, if storage rent is implemented, rent could reduced more than 50%, providing a great benefit for the financial inclusion use case. When transferring to addresses without previous balance, the current scheme costs 25K, while the proposed scheme costs only ~13K gas. 

### Space consumed by storage cells

The maximum cost of a storage cell is reached when the key and the data are both 32 bytes, and the node size becomes 64+10+2+32=108 bytes. The smallest possible trie node for a storage cell corresponds to a value of 0x01 at address 0x00. This cell would consume 14 bytes. Therefore the mimimum possible cost of a storage cell corresponds to 13% of the maximum cost.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

## Security Considerations

No new denial of service or resource abuse risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

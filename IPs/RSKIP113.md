---
rskip: 113
title: Unified Cache-Oriented Storage Rent for the Unitrie
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2019
---
# Unified Cache-Oriented Storage Rent for the Unitrie

|RSKIP          |113           |
| :------------ |:-------------|
|**Title**      |Unified Cache-Oriented Storage Rent for the Unitrie |
|**Created**    |2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes that users should pay storage rent for the use of account, contract and contract storage cells, to reduce the risk of storage spam and to make storage payments more fair. This contract is based on RSKIP61, but adapts it to work uniformly for any kind of node in the Unitrie, not only account or contract nodes.

# **Motivation**

For a summary of the discussions about Storage Rent see [RSKIP61](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP61.md). This RSKIP extends RSKIP61 and also reduces its complexity by applying storage rent uniformly to any kind of terminal non-empty node of the Unitrie.

# **Specification**

When a transaction is executed, all the Unitrie trie key/value pairs that are queried are stored in a cache. Also all Unitrie key/values that are written. After the transaction has been fully processed, the cache is iterated, and a storage rent is collected for every item. In case the item does not change the state of the node then the rent will only be paid if the rent is higher than 10000 gas (see later for the formula to compute the rent). If the state of a node is changed, then the rent will be paid if the rent is higher than 1000. This protects the network from performing costly micro-transactions. The place-holder key/values, such as the contract storage root node, are considered in the rent computations.

This RSKIP does not interfere with the plan to add parallel transaction execution if the transactions are scheduled properly, because there won't be key/value overlaps.

The rent is paid by extending the transaction to add a new field "rentGas".  The total gas consumed by a transaction will be equal to the execution gas consumed plus the rent gas consumed. The rentGas consumption is checked at the end of transaction execution. If the rent gas to consume becomes higher than rentGas, the transaction reverted. When the transaction is reverted due to not enough gas for storage rent, the storage rent is consumed in full.  If the transaction ends because of a previous OOG exception or REVERT, then only 25% of the storage rent is paid, even if the trie nodes are untouched.  The 25% of storage rent is paid to compensate the cost of accessing the contracts from the caches, but does not include the cost of writing-back the modified fields indicating last payment. As with normal gas, the full rentGas amount is deducted from the origin address and then the remaining is reimbursed at the end of the transaction processing. 

Each Unitrie node has a new field lastRentPaidTime. Let d be the timestamp of the block in which is being processed. Both fields are given in seconds. Note that the access of intermediate nodes (without value payload) do not pay rent, but their lastRentPaidTime is always updated when one of its children is modified. This will enable hibernation of nodes (all kinds) in the future

The following pseudo-code illustrates how rent is computed and paid for each node "dest".
```
if (d>lastRentPaidTime) {
    useRentGas =  nodeSize*(d-lastRentPaidTime)/2^21
    if ((dest was modified) && (useRentGas>=1000)) || 
       ((dest was NOT modified) && (useRentGas>=10000)) {
        dest.lastRentPaidTime = now
        consumeRentGas(useRentGas);
      }
}
```

The nodeSize is computed as the node value length plus 128. Therefore the nodeSize only approximates the actual space consumed, as it doesn't take into account embedded nodes.

Let SecondsAYear be 31536000. Each byte pays 1/2^21 gas per second. Therefore a storage byte pays SecondsAYear/2^21=15.03 gas units a year.  A simple account whose value length is 10 bytes, has a nodeSize of 138. Such account would consume 2070 units of gas a year. If the account performs payments regularly, the rent will be charged about twice a year. If is is inactive and only a contract checks its balance with the BALANCE opcode, it would pay rent once every 5 years. A contract with 10K bytes in code and 100 storage cells would pay approximately 380k units of gas/year. A contract with 100k cells will pay approximately 44k gas/day.

This value useRentGas is consumed from the transaction rentGas. 

When a node cell is created for the first time, the lastRentPaidTime is set 6 months in the future. This means that some rent is prepaid. 

The block gas limit does not apply to rents: the amount of rents paid in gas may be higher than the gas limit. Therefore the rent is an additional uncapped revenue stream for the miners.

## New Transaction Format

The transaction format is modified. Currently the transaction contains the following fields:
1. Nonce
2. GasPrice
3. GasLimit
4. ReceiveAddress
5. Value
6. Data
7. v
8. r
9. s

If the transaction has 10 fields or more, then field at index 10 (starting from 1) will correspond to the field rentGasLimit. The same size restrictions on the field gasLimit will apply to rentGasLimit. Also the rentGasLimit is subtracted in full from the sender's balance, and then the amount unspent is reimbursed. If the transaction does not specify a rentGasLimit, then rentGasLimit is assumed to be equal to the gasLimit. 

## New Receipt status values

If a transaction is reverted manually (REVERT), a new status of (-1) is recorded in the transaction receipt.
If a transaction is reverted because of standard OOG, the old empty-vector status is still used.
If a transaction is reverted because of rent OOG, a new status of (-2) is recorded in the transaction receipt.

## Future Impromenets

If a contract unpaid node rent becomes higher than a certain very high threshold, the node could be hibernated. 

Also this RSKIP can be combined with the SPV Compressed block propagation using state trie update batch (COBLOP)  method.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

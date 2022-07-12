---
rskip: 240
title: Implement Storage Rent in RSK
description: 
status: Draft
purpose: Sca, Sec, Fair
author: SDL (@sergiodemianlerner), SM (@shreemoy), DM (@diego)
layer: Core
complexity: 2
discussions-to: https://research.rsk.dev/t/rskip-240-implement-storage-rent-in-rsk/163
created: 2021-04-27
---
# Implement Storage Rent in RSK

|RSKIP          |240           |
| :------------ |:-------------|
|**Title**      |Implement Storage Rent in RSK|
|**Created**    |27-APR-21 |
|**Author**     |SDL, SM, DM|
|**Purpose**    |Sca, Sec, Fair|
|**Layer**      |Core|
|**Complexity** |2|
|**Status**     |Draft|
|**Discussions-to**|https://research.rsk.dev/t/rskip-240-implement-storage-rent-in-rsk/163 |

## Abstract
Storage rent is a *variable fee* collected from transaction senders to offset the cost of *retrieving relevant information* from blockchain state to execute transactions. The fee is called "rent" because it depends on the *time duration* for which the relevant data are stored in the *unitrie*. Rent is computed for each value-containing unitrie node "touched by" a transaction. This includes nodes containing account information, contract code and storage cells. Transaction senders currently pay *fixed costs* in gas for trie access through opcodes like `SLOAD`, `BAL`, `CALL` etc. Rent is an additional *variable cost* with a *cap* or limit. A trie node's rent is computed using a *timestamp* of that node's previous rent payment. The general idea is that these variable costs will be negligible for nodes that are accessed frequently and thus are more likely to be cached. However, the longer a node remains *unused*, the more outstanding rent it accumulates, and the more it will cost (with some cap) to read. Such nodes are less likely to be cached in memory.

## Motivation

Storage rent introduces variable fees for trie access. These fees can help in the following ways

- make storage payments fairer by introducing a *time dimension* to storage fees
- improve state caching by *introducing a cache hierarchy*
- help secure the network from some IO attacks by making disk IO more expensive
- reduce the risk of storage spam and gas arbitrage
- encourage developers and product designers to treat state storage more judiciously - important for long term scalability, sustainability and fairness

### Why introduce rent now

RSK is a relatively young blockchain and the above factors are not really pressing issues at the moment. However, the RSK community is working towards massive growth and user adoption. If successful, then introducing new economic mechanisms (such as storage rent) later can become more challenging and potentially disruptive. For illustration, one can look at Ethereum's experience. Despite continuing support from several core developers and researchers, the Ethereum community has struggled to reach consensus on introducing rent. Their experience illustrates that introducing a new pricing system in a mature and popular blockchain can be extremely challenging. 

Whenever practical, the RSK community tries to maintain compatibility with Ethereum. In the future, should the Ethereum community (or another public chain) come up with a clearly superior system for state size management, then we can always implement the same in RSK.

## Specification

Storage rent is not a new idea - it is mentioned in the RSK whitepaper [1]. It has also been proposed in Ethereum multiple times with varying designs [2, 3]. The current proposal has been influenced by several prior RSKIPs including [RSKIP21](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP21.md), [RSKIP52](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP52.md), [RSKIP61](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP61.md) and [RSKIP113](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP113.md). See [4] for additional background information.

This proposal requires a hard fork.

### Trie changes

- Add a field `lastRentPaidTime` to the unitrie. 

This is a time stamp in unix seconds for each unitrie node. This is interpreted as the time until which rent has been "fully paid". This timestamp will typically lie *in the past* because rent is not paid in advance. The timestamp is moved forward with each rent payment, with the rate of advance depending on both the payment amount and the size of data stored in the node.

- Add another field `nodeVersion` to the unitrie. 
 for at the point a transaction execution stops
Node versioning is required for serialization of new trie nodes with rent timestamps and older nodes without timestamps. We can use version number 0 for Orchid encoding, 1 for the current encoding (RSKIP107), and 2 for nodes with rent time stamp (current proposal). Note that the current implementation of RSKJ already uses implicit node version 1 in bit positions 6 and 7 of `flags` in Trie class. This will be changed to 2 (`10` in binary) for storage rent.

### Rent computation

**Reference Time:** The timestamp of the current block (where the transaction is included) should be used as the reference time for calculating time deltas for rent computations.


Rent is computed and tracked at the granularity of individual *value-containing* unitrie nodes. Rent accounting follows a *pay-after-use* model. The rent for a trie node depends on three factors:

1. the node's size (in bytes) 
2. the duration (in seconds) since rent was last paid (fully). 
3. and the **rental rate** `R = 1/2^21 gas/byte/second` (approximately **15 gas per byte per year**)

A node's size (`nodeSize`) is computed as the node's value length plus 128 bytes for storage overhead. Therefore, `nodeSize` only approximates the actual space consumed, e.g. it doesn't take into account embedded nodes.

Illustrative **approximate** examples of rent
- `(10+128)*15 ~ 2100 gas/year` for an account with 10 bytes of data
- `(32+128)*15 ~ 2500 gas/year` for a storage cell (32 bytes maximum)
-  `(2500+128)*15 ~ 40000 gas/year` for a contract of size 2500 bytes (about the size of an ERC20 token contract)

### Rent Collection Thresholds and Caps

The dependence on timestamps makes rent payments *variable*. These are **capped** on a *per-transaction* basis, so a trie node's rent payments get spread across different transactions and across users (transaction senders).

Updating a node's rent timestamp requires a trie *put* operation, which can affect performance. To reduce the number of trie writes "driven by" timestamp updates, we require some minimum "collection thresholds" on the amount of rent that is due. This threshold is different for trie reads and trie writes.

Rent *does not* depend directly on opcodes. It depends on the type of node accessed. The reference to opcodes here is mostly for illustration. Some operations such as `CALL` can touch several trie nodes (accounts, code, storage). It is also common for the same trie node to be touched *multiple times*, in *different ways*, and at *different call depths* within the same transaction. Within a single call frame, the amount of rent collected in a transaction should not depend on the order in which or the number of times a trie node is accessed. Naturally, the order of access does matter at different call depths. 

When a *pre-existing* value is updated in the trie, then the additional performance cost of updating its rent timestamp is low. Thus, the threshold for *trie-write* operations is set lower than those for trie-reads. To be clear, this only applies to updating *previously existing values* (e.g. balances, storage).  Contract code cannot be 'updated' and new nodes (of any type) get a timestamp for free when they are created.

| State Data Type | Fixed Cost (reads) |  Fixed Cost (updates) | Rent Threshold (reads) | Rent Threshold (updates) | Rent Cap |
|:------------ |:-------------|:-------------|:-------------|:-------------|:-------------|
|Account Balance|  400 gas (**700**\* )| 9000 gas (`CALL` with value) | 2500 gas |1000 gas |   5000 gas |
|Contract Storage |  200 gas (**800**\*) | 5000 gas (`SSTORE`)  |2500 gas | 1000 gas |    5000 gas |
|Contract Code Hash | 400 gas (**700**\*) (`EXTCODEHASH`) | N/A | 2500 gas| N/A |  5000 gas |
|Contract Code| 700 gas (e.g. `CALL`, `EXTCODECOPY`) | N/A | 15,000 gas| N/A |  30,000 gas |

**Note**: "N/A" is not applicable (code cannot be "updated"). Values with (*) are those proposed in [RSKIP239]((https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP239.md)), which reprices trie read costs (`BAL`, `SLOAD`, `EXTCODEHASH`). 

Updating the timestamp is more resource intensive for `EXTCODECOPY` (or `EXTCODESIZE`) than for `EXTCODEHASH`. This is because `EXTCODEHASH` does not require loading contract code. See the Rationale section for *additional remarks* about these values.



**New Nodes**: New nodes (of any type) receive the timestamp of the block they are created in. No advanced rent is collected. If needed (in future) advanced rent payment for new nodes can be incorporated into fixed costs.

**Security and DoS:** To increase the costs of IO-DOS attacks, the maximum rent cap is charged for attempting to read data from nodes that do not exist in the trie.

**Duration cap:** There is a limit on the amount of time for which rent is accrued. This is set at 3 years. If a node remains untouched for 3 years, then the amount of outstanding rent does not grow any further, it remains capped at that level. Letting rent accumulate forever seems unnecessarily punitive.

### Computing a node's rent time stamp

Suppose a trie node has nodesize `Size` (bytes). Let its current timestamp be `lastRentPaidTime = t_0` (seconds) and let the current block's timestamp be `t_now` (seconds). Denote the rental rate by `Rent` (gas/byte/second). Denote the *relevant* (read or write) collection threshold for this node type by `Cutoff` (gas), and the rent limit  by `Cap` (gas).

Then the *outstanding rent* for this node is `rent_due = (Size * Rent) * (t_now - t_0)`. The amount of rent to be collected (if any) depends on whether the data in the node is modified by the transaction or not. 

- if `rent_due < Cutoff`, then no rent is collected and `lastRentPaidTime` remains unchanged.
- if `rent_due > Cutoff` but `rent_due < Cap`, then the entire outstanding amount is collected and `lastRentPaidTime` is set equal to the current block's timestamp.
- if `rent_due > Cap`, then only `Cap` is collected. In this case, the timestamp is advanced as follows (using "`//`" for **division with round down**)

```
/* If rent due exceeds `Cap`*/
t_paid = Cap//(Size * Rent) /* time for which rent is paid (`//` is division with round down)*/
t_new = t_0 + t_paid /* update the timestamp */
```

**Example 1.** Suppose a transaction uses the `SLOAD` opcode to read the value of a storage cell with outstanding rent of 1600 gas. If this cell is not modified by the transaction, then no rent is collected, since the amount due is less than the threshold for reads (2500 gas). However, if the same transaction *modifies* the value contained in the cell using `SSTORE`, then the minimum threshold for rent collection is 1000 gas. In this case, the full amount of rent due (1600) is collected  and the timestamp is updated to the current block's timestamp.

**Example 2.** A contract with 5000 bytes of code has not been touched since it was deployed 2 years ago. The amount of rent due is approximately `(5000+128) * 15 * 2 = 15400`. The first user to call this contract will be charged the rent cap of 40,000 gas. This payment will *advance* the contract's rent timestamp by `40000/(15 *5128) = 0.52` years  i.e about 6 months. 

### Tracking and collection mechanics

All trie nodes *accessed*, *modified* or *created* by a transaction are automatically checked and added to rent-tracking caches. Users cannot select or exclude individual rent payments. 

One way of implementing the system is to have three caches of value-containing nodes: 

1. a *set* of all trie nodes seen thus far during transaction execution
2. a *map* of all trie nodes whose rent timestamps are to be updated along with their individual (updated) timestamps. Some of these nodes will also have updated values (e.g `SSTORE`).
3. a *set* of newly created tries nodes. All of them will receive the timestamp of the current block.

These caches are passed along to all child calls, so that rent tracking does not kick in more than once for any node irrespective of call depth. After a transaction has been fully processed, the caches are iterated and the updated values and timestamps are committed to the state trie.

### Gas counters and Error Handling

Rent is consumed from the same gas counter as usual, sourced initially from a transaction's `gaslimit` field. The `usedGas` tracker should include both execution and rent gas consumption.

Usually, when there is an *unhandled exception* at any call depth, TX stops and all state changes are reverted. Any gas used thus far is not refunded. This philosophy must be maintained for storage rent as well. If state changes are reverted at any call depth, due to an error, then all associated rent updates must also be reverted. Not doing so can introduce unintended side effects.

However, if timestamps are not updated following a revert, then it seems unfair to consume all the rent "collected" for those updates. Therefore, a *separate* `usedRentGas` counter should be implemented to handle call reversions or exceptions. If a transaction ends because of a OOG exception or REVERT, then 25% of the storage rent gas used so far (`usedRentGas`)  is consumed as compensation for computation and IO costs. Recall that rent is also intended as a deterrence against DoS attacks, so we cannot refund all of the rent accounted for, even if timestamps remain unchanged.

If an error at some call depth is handled in a way that transaction execution can proceed, then only the timestamp changes associated with the failed call (and its subcalls) need to be reverted. Again, 75% of gas used for rent in the failed call should be refunded to the caller.

#### Dependencies
The limits on variable costs in this proposal are based  on "re-priced" gascosts from a related proposal, RSKIP239. While these proposals can be adopted independently, we think it is beneficial to make the current proposal (RSKIP240) dependent on RSKIP239.

## Rationale

The specification already explains much of the rationale. Here we include some additional details.

### Rent triggers and caps

The cost of writing a rent timestamp to the trie is an unavoidable accounting cost of implementing storage rent. Lacking data to perform an accurate cost-benefit analysis, we adopt a heuristic and conservative approach.

The cost of *updating* a value in a storage cell is 5000 gas (`SSTORE`). Meanwhile, a *value-transferring* `CALL` costs 9000 gas, which apart from a call stipend of 2300 gas, must also cover the cost of *two* account updates (the caller's and callee's). Thus, for account nodes and storage cells, the cost or re-writing data to state trie is around 4000 or 5000 gas. The cost for writing contract code to the trie is much higher - about 200 gas *per byte* is charged for `CREATE`.

However, larger thresholds decrease the frequency with which rent is collected, which lowers the informativeness of the rent timestamp. So, we propose a smaller value of 2500 gas for storage cells and account nodes. This value corresponds to their approximate annual rent. For the rental cap, we use 5000 gas, which is what the rent would be if an account or storage cell remains untouched for 2 years. For contract code, we set the threshold higher, at 15,000 gas with a cap of 30,000 gas. Trying to spread the storage rent for a contract across many users (for "fairness") would require too many trie updates.

## Backwards Compatibility

As already mentioned, this change needs a hard fork. In most cases, increasing the `gaslimit` should enough to account for rent payments. RPC method to `estimate_gas` will provide reasonable guidance. However, there can be some breaking changes.

*Value-transferring* `CALL`s cost 9000 gas. From this 9000, an amount 2300 gas is subtracted as a *call stipend* and passed to the *receiving* contract. The stipend is intended to cover the cost for logging the value transfer. If the receiving contract performs any  `BAL` or `SLOAD` operations as part of logging, and if enough rent has accumulated in those accounts or storage cells, then some CALLs may fail.

Contracts that make CALLs to external contacts with specific gas limits can also fail. Such patterns have been discouraged for quite some time now, at least since EIP-150, which increased awareness that gas costs can change  and developers should not make their execution logic rely on the gas schedule. 

We think such breaks will be rare in RSK. We also note that the Ethereum community has deliberated and implemented potentially breaking changes of similar magnitude, most recently EIP-2929 which increased trie read operations by 3X.

## Implementations

Previously, the authors implemented a version with significant overlap with the current proposal. That implementation is being modified and will be shared with the community. This RSKIP will be modified with updated information at that point.

### Implementation notes (RSKJ)

Trie related
- Use `long` for `lastRentPaidTime` with a default value of -1 (minus one). This value can help distinguish missing timestamps (which will default to 0). Recall that the trie data structure is also used for other tries (transactions, receipts)  
- RSKJ currently uses an *implicit* node version number `01` as part of `flags` (bit positions 6, 7). For serialization, these bit positions should be changed to version 2 i.e. `10` in `flags`. 
- Can use `nodeVersion` 0 when reading nodes with `Orchid` serialization format using `fromMessageOrchid()`. This will not affect encoding or hashing for Orchid.
- the trie `put(key, value)`  method has to be *generalized* to `put(key, value, lastRentPaidTime)` to incorporate  `lastRentPaidTime`. The default value `put(key, value, -1` can be used for puts that do not need a timestamp (e.g. version 1 nodes, transaction tires, receipts tries)

## Test Cases

Incomplete (tests have to be specified more clearly and additional tests to be added) 

- trie tests, check timestamps, node versioning (serialization, trie hashes)
- check rent computation with and without caps on variable costs
- check gas estimation
- check OOG refunds


## References

[1] Sergio Demian Lerner, "RSK Rootstock Platform Whitepaper", https://www.rsk.co/Whitepapers/RSK-White-Paper-Updated.pdf  

[2] Alexey Akhunov, "State Fees (formerly State rent) pre-EIP proposal version 3" https://ethresear.ch/t/state-fees-formerly-state-rent-pre-eip-proposal-version-3/4996

[3]  Avatar Felföldi Zsolt, "A pay-for-storage model for evolving tree hashed data structures", https://gist.github.com/zsfelfoldi/a207d216b3fa9ae4be6abe7a5d8e68d8

[4] Sergio Demian Lerner, "Blockchain State Storage Revised", https://bitslog.com/2018/01/22/storage-rent-revised


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

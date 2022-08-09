---
rskip: 240
title: Implement Storage Rent in RSK
description: 
status: Draft
purpose: Sca, Sec, Fair
author: SDL (@sergiodemianlerner), SM (@smishraiov), DM (@diegomasini), FJ (@fedejinich)
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
|**Author**     |SDL, SM, DM, FJ|
|**Purpose**    |Sca, Sec, Fair|
|**Layer**      |Core|
|**Complexity** |2|
|**Status**     |Draft|
|**Discussions-to**|https://research.rsk.dev/t/rskip-240-implement-storage-rent-in-rsk/163 |

## Abstract
Storage rent is a new *variable fee* collected from transaction senders to help manage the growth of blockchain state size. The fee is called "rent" because it depends on the *time duration* for which data are stored in the state *unitrie*. Rent is computed for each value-containing trie node "touched by" a transaction. This includes nodes containing account information, contract code and storage cells. Transaction senders currently pay *fixed costs* in gas for trie access through opcodes like `SLOAD`, `BAL`, `CALL` etc. Rent is an additional *variable cost* with a *cap* or limit. A trie node's rent is computed using a *timestamp* of that node's previous rent payment.

## Motivation

Storage rent introduces variable fees for trie access. These fees can help in the following ways

- make storage payments fairer by introducing a *time dimension* to storage fees
- help secure the network from some IO attacks by making disk IO more expensive
- reduce the risk of storage spam and gas arbitrage
- encourage developers and product designers to treat state storage more judiciously - important for long term scalability, sustainability and fairness
- In the future, rent timestamps can be used for new *cache hierarchy*. Long unused parts of the state (old timestamps) can be stored at the lowest levels of caches or slowest disks. 
- If required in the future, rent timestamps can be used for other ways of managing state size growth such as *state expiration (hibernation)*.

## Specification

Storage rent is not a new idea - it is mentioned in the RSK whitepaper [1]. It has also been proposed in Ethereum multiple times with varying designs [2, 3]. The [Solana blockchain](https://docs.solana.com/storage_rent_economics) currently has a rudimentary version of storage rent. The current proposal has been influenced by several prior RSKIPs including [RSKIP21](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP21.md), [RSKIP52](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP52.md), [RSKIP61](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP61.md) and [RSKIP113](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP113.md). See [4] for additional background information.

This proposal requires a hard fork.

### Trie changes

- Add a field `lastRentPaidTime` to the unitrie. 

This is a time stamp in unix seconds for each unitrie node. This is interpreted as the time until which rent has been "fully paid". This timestamp will typically lie *in the past* because rent is not paid in advance. The timestamp is moved forward with each rent payment, with the rate of advance depending on both the payment amount and the size of data stored in the node.

### Trie node versioning
Trie node versioning can be used for serialization of new trie nodes with rent timestamps and older nodes without timestamps. We can use version number 0 for Orchid encoding, 1 for the current encoding (RSKIP107), and 2 for nodes with rent time stamp (current proposal). The current implementation of RSKJ already uses *implicit node version* 1 in bit positions 6 and 7 of `flags` in Trie class. This will be changed to 2 (`10` in binary) for storage rent.


### Internal node timestamps

Timestamps are to be added to all trie nodes, including internal nodes that may not contain any value. We propose that the timestamp of internal nodes be updated to match that of the *most recently updated child* node. Such timestamps can help with future improvements, such as introducing new types of caches, or state expiration schemes (hibernation) based on node timestamps.

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
-  `(2500+128)*15 ~ 40000 gas/year` for a contract of size 2500 bytes (about the size of a basic ERC20 token contract)

### Rent Collection: Thresholds and Caps

Rent payments are state dependent and therefore *variable*. The same transaction sent at different times may trigger the collection of different amounts of rent, or very likely, none at all. That is because the amount of rent due depends on a node's previous timestamp. Furthermore, in any transaction, we restrict the amount of rent collected (if any) for a single trie node to lie in predetermined ranges.

### Cap on rent collected for a node
The amount of rent collected in a transaction for an individual trie node is **capped**. This rent collection for each node gets spread across different transactions. Note that this cap is for each individual trie nodes accessed by the transaction, and there is no rent cap at the level of a transaction itself.

### Minimum threshold for rent collection
Updating a node's rent timestamp requires a trie *put* operation. When possible, the amount of rent collected should cover the cost of this put operation. Thus, we should collect rent for a node only if it exceeds some minimum "collection threshold". This threshold can be different beased on whether the transaction is updating the value of a trie node or merely reading it. Since updating the value requires a trie put, we can use a lower threshold for rent collection. The threshold will be higher for collecting rent (and updating the timestamp) for a pure read operation. For simplcity, we propose a common set of thresholds for all types of nodes -- irrespective of their content (account info, storage, or contract code). A common threshold can can be justified even for contract code, because the bytecode is not actually stored in the trie itself. All values (of trie nodes) that are longer than 32 bytes are stored separately.

Rent *does not* depend directly on opcodes. The reference to opcodes here is merely for illustration. Some operations such as `CALL` can touch several trie nodes (accounts, code, storage). 

| State Data Type | Fixed Cost (reads) |  Fixed Cost (updates) | Rent Threshold (reads) | Rent Threshold (updates) | Rent Cap |
|:------------ |:-------------|:-------------|:-------------|:-------------|:-------------|
|Account Balance|  400 gas (`BAL`) | 9000 gas (`CALL` with value) | 5000 gas |1000 gas |   10,000 gas |
|Contract Storage |  200 gas (`SLOAD`) | 5000 gas (`SSTORE`)  |5000 gas | 1000 gas |    10,000 gas |
|Contract Code| 700 gas (e.g. `CALL`, `EXTCODECOPY`) | N/A | 5000 gas| N/A |  10,000 gas |

**Note**: "N/A" is not applicable.

It is common for the same trie node to be accessed *multiple times*, in *different ways*, and at *different call depths* within the same transaction. Within a single call frame, the amount of rent collected in a transaction should not depend on the order in which, or the number of times, a trie node is accessed. Naturally, the order of access does matter at different call depths. During the course of any transaction, the rent for a node will be collected *at most once* (provided it exceeds a threshold). 

**New Nodes**: New nodes (of any type) receive the timestamp of the block they are created in. No advanced rent is collected. If needed (in future) advanced rent payment for new nodes can be incorporated into fixed costs.

**Security and DoS:** To increase the costs of IO-DoS attacks, a fee of 2500 gas is charged as part of "storage rent" for attempting to read data from nodes that *do not exist* in the state trie. Unless the entire state trie fits on RAM, such queries force the client to search on disk.  This IO-DoS fee effectively adds 2500 gas to such queries. This is kept low because there are legitimate use cases for trying to read values that may not exist, e.g. checking if an address owns any amount of an ERC20 token.

**Duration cap:** Some parts of the state trie may remain untouched for long periods of time. Letting rent accumulate forever for unused nodes does not make much sense. Therefore, we place a limit on the amount of time for which rent can accrue. This is set at 3 years. If a node remains untouched for 3 years, then the amount of outstanding rent does not grow any further, it remains capped at that level irrespective of the amount computed based on its timestamp.

### Computing rent and advancing timestamps

Suppose a trie node has nodesize `Size` (bytes). Let its current timestamp be `lastRentPaidTime = t_0` (seconds) and let the current block's timestamp be `t_now` (seconds). Denote the rental rate by `Rent` (gas/byte/second). Denote the *relevant* (read or write) collection threshold by `Thresh` (gas), and let the transaction-level cap on rent for a trie node be `Cap` (gas).

The *outstanding rent* for this node is `rent_due = (Size * Rent) * (t_now - t_0)`. The amount of rent to be collected (if any) depends on whether the data in the node is modified (a trie write or delete) by the transaction or just a trie read. 

- if `rent_due < Tresh`, then no rent is collected and `lastRentPaidTime` remains unchanged.
- if `rent_due > Tresh` but `rent_due < Cap`, then the entire outstanding amount is collected and `lastRentPaidTime` is set equal to the current block's timestamp.
- if `rent_due > Cap`, then only `Cap` is collected. In this case, the timestamp is advanced as follows (using "`//`" for **division with round down**)

```
/* If rent due exceeds `Cap`*/
t_paid = Cap//(Size * Rent) /* time for which rent is paid (`//` is division with round down)*/
t_new = t_0 + t_paid /* update the timestamp */
```

**Example 1.** Suppose a transaction uses the `SLOAD` opcode to read the value of a storage cell with outstanding rent of 2000 gas. The amount due is less than the threshold for reads (5000 gas), so it would trigger any rent collection (or timestamp update). However, if a transaction *modifies* the value contained in the same cell using `SSTORE`, then the minimum threshold for rent collection is 1000 gas. In this case, the full amount of rent (2000 gas) is collected  and the timestamp is updated to the current block's timestamp.

**Example 2.** A contract with 5000 bytes of code has not been touched since it was deployed 2 years ago. The amount of rent due is approximately `(5000+128) * 15 * 2 ~ 154000` gas. The first user to call this contract will be charged some rent for it. Since the amount due (154,000 gas) is more than the (per transaction, per node) cap of 10,000 gas, the user will be charged the cap. This payment will *advance* the contract's rent last paid timestamp by `10000/(15 * 5128) ~ 0.13` years  i.e by about 48 days. It will still owe 144,000 gas in rent.

## Rent Tracking, Gas counters, Reverts and Error Handling

All trie nodes *accessed*, *modified*, *deleted*, or *created* by a transaction are automatically checked and added to rent-tracking objects. The details of how to create these objects is an implementation issue.  Users can always *trigger* (or "poke") rent tracking for some nodes but trying to access it from the state. But it is important that rent tracking be automated so that users (transaction senders) cannot *exclude* specific nodes from rent tracking.

Just like regular transaction fees, storage rent is paid from a transaction's `gaslimit` field. Gas collected as rent will be included in the block's used gas counter. However, unlike regular execution gas, which is collected on the fly,  rent is only collected at the end of transaction execution. Until then, we only keep a track of the trie nodes for which rent has to be collected. Actual rent payments and corresponding timestamp updates only happen at the end of transaction execution. This means that the `usedGas` tracker will continue to measure execution gas only. 

Usually, when there is an *unhandled exception* at any call depth, transaction execution stops and all state changes are reverted. Any gas used thus far is not refunded. Since all state changes are reverted there is no question of updating any rent timestaps. Technically, the state of the transaction sender's account does change (due to gas used and nonce), but we do not update the rent timestamp.

Tracking, computing, and updating rent status requires some extra computation. Therefore, even if timestamps are not updated following exceptions or reverts, we collect some rent as compensation for the additional resource use. Therefore, if a transaction (including *internal transactions*) fails for some reason then 25% of the *computed storage rent* is collected as compensation for additional resource use. This can be implemented by maintaing a list of trie nodes that were accessed but the changes were *rolled back* (e.g.  an OOG exception or REVERT). Collecting rent after an "OOG" may sound very odd. But it only makes sense in the special case where an internal call has its own restricted gas limit (less than what is available). Otherwise, if a transaction (internal or external) stops executing because it runs out of gas, then no rent collection can happen at all since the transaction never "reaches" the end (when rent collection happens).

### OOG "because of" rent and updated estimateGas

Users and wallet software often use `estimateGas` to set the `gaslimit`, sometimes with an additional cushion. The `estimateGas` method should return an estimate inclusive of rent payments. Otherwise, transactions may fail with an OOG exception. Since rent is state dependent, it may happen that the actual rent computed (when a transaction is mined) is different from the estimated value. Therefore, users should increase the gasLimit in a conservative manner.

It is possible that some wallet software do not use `estimateGas` for simple transafers of RBTC. TO avoid such transactions from failing, we propose that rent tracking and collection should be *turned off* for simple send transactions. These transactions are characterized (identified) by a combination of `gaslimit` set equal to the base value (e.g. 21,000 gas) and they do not have anything in the `data` field.

### Avoid multiple collections

Rent for any node should not be collected more than once in a transaction. For example, suppose a storage write operation is reverted due to REVERT or OOG (or any reason). Then (from above) 25% of the outstanding rent should be charged. However, suppose another part of the same transaction's execution writes to the same cell, but this time it is not reverted. In this case, the regular amount of rent should be collected. The implementation should make sure that we do not collect rent more than once. If situations like the above happen, then we should drop the node from set of reverted or "rolled back" changes.  We should only collect the rent for the unreverted change, and update the timestamp. One way to ensure that we do not collect rent more than once is to check that there are no repeated trie keys in the set of nodes to be updated (actual rent payments) or the set of rollbacks (25% fee payments).

## Timing of Rent Collection and Refunds

Regular execution gas is collected on the fly as a transaction is executed. This gas must be sourced from the transaction's gaslimit, since it is compensation for irreversible resource use. We do not have to be as stringent with rent and the collection can wait until the end of transaction execution. One reason for collecting rent at the end (instead of on the fly) is to avoid introducing breaking changes when handling calls that pass a limited amount of gas. Such calls can break (OOG) if the total amount of execution gas and rent exceeds the gaslimit of the internal transaction. 

Then there is the issue of the timing of rent versus refunds. We propose that rent collection should happen *before* the sender receives credit for any refunds.  We could potentially use refunds to pay for rent. The main effect of this timing (collecting rent before refunds) is that it may increase the *effective* size of refunds. This is because refunds are restricted to be no more than one half of execution gas. So, if the total gas used increases because of rent payments, then so does the amount that can be refunded. This can serve as a small increase in the incentive to reduce state size.

### Deleted cells and `SELF-DESTRUCT`

There are two ways to remove data from state: by setting storage cells to zero (using `SSTORE`), or via self-destructing contracts. There is nothing else to delete, since ordinary accounts must always be stored in state with their nonce. In the case of  `SELF_DESTRUCT`, all of the deleted contract's storage cells are also deleted. Such deletions of storage cells and contracts reduce state size, and are incentivized via partial refunds.

When a storage cell is set to zero, or a contract is destructed, then rent collection proceeds as normal. We compute the outstanding rent and collect it if it exceeds the threshold. The amount collected is still subject to the per transaction cap. However, since the node will be removed from the trie, there is no point in updating its timestamp.

One point to note is that for a self-destructing contract, we do not attempt to compute or collect outstanding rent for all of its (soon to be removed) storage cells. For one, trying to do so may lower the incentive to clear the state (refund gets reduced or eliminated). Furthermore, for a contract with lots of storage, it may not be worth the effort to scan each of its cells to compute the outstanding rent status before deleting them.

## Rationale

The specification already explains much of the rationale. Here we include some additional details.

### Rent triggers and caps

The cost of writing a rent timestamp to the trie is an unavoidable accounting cost of implementing storage rent. Lacking data to perform an accurate cost-benefit analysis, we adopt a heuristic and conservative approach.

The cost of *updating* a value in a storage cell is 5000 gas (`SSTORE`). Meanwhile, a *value-transferring* `CALL` costs 9000 gas, which must cover the cost of *two* account updates (the caller's and callee's). Thus, the implicit cost or re-writing data to state trie is around 4000 or 5000 gas. This is why, for pure reads,  we collect rent only if it is higher than 5000 gas.   

## Backwards Compatibility

As already mentioned, this change needs a hard fork. Since rent is collected at the end of transaction execution, we do not expect any breaking changes. All existing contracts should work as before. However, the amount of gas consumed by transactions will be higher than before, but only if a transaction triggers rent collection. Therefore, users and applications should increase the `gaslimit` from previouly used vales. In most cases, the RPC method to `estimateGas` will provide reasonable guidance, however, users should add a little extra to account for the fact that rent payments are state dependent. The actual rent collected can be higher or lower than expected depending on when a transacion gets included in a block. 

## Implementations

An experimental implementation of the proposal is being developed ([work in progress](https://github.com/rsksmart/rskj/pull/1798)).

## Test Cases

Incomplete (tests have to be specified more clearly and additional tests to be added) 

- trie tests, check timestamps, node versioning (serialization, trie hashes)
- check rent computation with and without caps on variable costs
- check gas estimation
- check rent collection for OOG exceptions, reverts, and deleted cells. Ensure rent for a node is not collected more than once per transaction.

### Side Effects

The benefits of storage rent will be realized in the future. However, there are some immediate costs of introducing rent.  The timestamps will typically need 8 bytes, while a typical storage node (including the overhead) is around 130 bytes. Thus, the addition of a timestamp will increase the size of the state trie. Second, tracking and computing rent for various nodes will increase the execution time, and also an increase in the number of trie writes (to update timestamps). These performance impacts are expected to be small and will be measured by benchmarking the experimental implementation.


## References

[1] Sergio Demian Lerner, "RSK Rootstock Platform Whitepaper", https://www.rsk.co/Whitepapers/RSK-White-Paper-Updated.pdf  

[2] Alexey Akhunov, "State Fees (formerly State rent) pre-EIP proposal version 3" https://ethresear.ch/t/state-fees-formerly-state-rent-pre-eip-proposal-version-3/4996

[3]  Avatar Felföldi Zsolt, "A pay-for-storage model for evolving tree hashed data structures", https://gist.github.com/zsfelfoldi/a207d216b3fa9ae4be6abe7a5d8e68d8

[4] Sergio Demian Lerner, "Blockchain State Storage Revised", https://bitslog.com/2018/01/22/storage-rent-revised


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

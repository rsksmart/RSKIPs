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
|**Discussions**|add link to https://research.rsk.dev/ |

## Abstract
Storage rent is a *variable fee* collected from transaction senders to offset the cost of *retrieving relevant information* from blockchain state to execute transactions. The fee is called "rent" because it depends on the *time duration* for which the relevant data are stored in the *unitrie*. Rent is computed for each value-containing unitrie node "touched by" a transaction. This includes nodes containing account information, contract code and storage cells. Transaction senders currently pay *fixed costs* in gas for trie access through opcodes like `SLOAD`, `BAL`, `CALL` etc. Rent is an additional *variable cost* with a *cap* or limit. A trie node's rent is computed using a *timestamp* of that node's previous rent payment. The general idea is that these variable costs will be negligible for nodes that are accessed frequently and thus are more likely to be cached. However, the longer a node remains *unused*, the more outstanding rent it accumulates, and the more it will cost (with some cap) to read. Such nodes are less likely to be cached in memory.

## Motivation

Storage rent introduces variable fees for trie access. These fees can help in the following ways

- make storage payments fairer by introducing a *time dimension* to storage fees
- improve state caching by *introducing a cache hierarchy*
- help secure the network from some IO attacks by making disk IO more expensive
- reduce the risk of storage spam and gas arbitrage
- encourage developers and product designers to treat state storage more judiciously - important for long term scalability, sustainability and fairness

RSK is a relatively young blockchain and the above factors are not really pressing issues at the moment. However, the RSK community is working towards massive growth and user adoption. If successful, then introducing new economic mechanisms (such as storage rent) later can become more challenging and potentially disruptive. For illustration, one can look at Ethereum's experience. Despite continuing support from several core developers and researchers, the Ethereum community has struggled to reach consensus on introducing rent. Their experience illustrates that introducing a new pricing system in a mature and popular blockchain can be extremely challenging. 

Whenever practical, the RSK community tries to maintain compatibility with Ethereum. In the future, should the Ethereum community (or another public chain) come up with a clearly superior system for state size management, then we can always implement the same in RSK.

## Specification

Storage rent is not a new idea - it is mentioned in the RSK whitepaper [1]. It has also been proposed in Ethereum multiple times with varying designs [2, 3]. The current proposal has been influenced by several prior RSKIPs including [RSKIP21](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP21.md), [RSKIP52](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP52.md), [RSKIP61](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP61.md) and [RSKIP113](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP113.md). See [4] for additional background information.

This proposal requires a hard fork.

### Trie changes

- Add a field `lastRentPaidTime` to the unitrie. 

This is a time stamp in unix seconds for each unitrie node. This is interpreted as the time until which rent has been "fully paid". This timestamp will typically lie *in the past* because rent is not paid in advance. The timestamp is moved forward with each rent payment, with the rate of advance depending on both the payment amount and the size of data stored in the node.

- Add another field `nodeVersion` to the unitrie. 

Node versioning is required for serialization of new trie nodes with rent timestamps and older nodes without timestamps. We can use version number 0 for Orchid encoding, 1 for the current encoding (RSKIP107), and 2 for nodes with rent time stamp (current proposal). Note that the current implementation of RSKJ already uses implicit node version 1 in bit positions 6 and 7 of `flags` in Trie class. This will be changed to 2 (`10` in binary) for storage rent.

### Rent computation

Rent is computed and tracked at the granularity of individual *value-containing* unitrie nodes. Rent accounting follows a *pay-after-use* model. The rent for a trie node depends on three factors:

1. the node's size (in bytes) 
2. the duration (in seconds) since rent was last paid (fully). 
3. and the **rental rate** `R = 1/2^21 gas/byte/second` (approximately **15 gas per byte per year**)

A node's size (`nodeSize`) is computed as the node's value length plus 128 bytes for storage overhead. Therefore, `nodeSize` only approximates the actual space consumed, e.g. it doesn't take into account embedded nodes. One impact of the storage overhead is that the effective rent for storage cells (32 bytes maximum) is more than that for nodes containing contract code which can be thousands of bytes.

Illustrative **approximate** examples of rent
- `(32+128)*15 ~ 2500 gas/year` for storage cell (32 bytes)
-  `(2500+128)*15 ~ 40000 gas/year` for a contract of size 2500 bytes 

### Caps on rent fees

The dependence on timestamps makes rent payments variable. These payments are capped, so that the rent for various nodes is spread across different transactions.

| Node Type | Related OpCodes          | Fixed Cost for reading | Rent Cap (max variable cost) |
|:------------ |:------------ |:-------------| :-------------|
|Storage Cell | `SLOAD`      | 800* gas (200 current) | 1600 gas |
| Account Info| `BAL`    | 700* gas (400 current)| 1400 gas |
|Contract Code | `CALL*`, `EXTCODE*`  | 700 gas | 1400 gas|

**Note**: * The fixed cost for `SLOAD` and `BAL` are values proposed in RSKIP_repriceRead. Current values are in parentheses.

**Minimum collection trigger:** To avoid tiny payments and reduce the number of trie IO operations, rent is collected only if the amount due for a trie node exceeds `rent_trigger = 200 gas` (about 1 month rent for a storage cell).

**Reference Time:** The timestamp of the current block (where the transaction is included) should be used as the reference time for calculating time deltas for rent computations.

**New Nodes**: New nodes receive the timestamp of the block they are created in. No advanced rent is collected.

**Updating rent time stamp**

Suppose a trie node has `nodesize = S bytes`. Let `lastRentPaidTime = ts_0` and let the current block's timestamp be `ts_now`. Recall the rental rate is `R gas/byte/sec`. Let the rent cap be `C`.

Then the outstanding rent for this node is `rent_due = (S * R) * (ts_now - ts_0)`

- if `rent_due` is less than the minimum collection trigger (200 gas), then no rent is collected and `lastRentPaidTime` remains unchanged.
- if `rent_due` exceeds the collection trigger (200 gas) but is less than the rent cap `C`, then the outstanding amount is collected and `lastRentPaidTime` is set equal to the reference time (current block timestamp)
- if `rent_due` exceeds the cap `C`, then only `C` is collected. In this case, the timestamp is advanced as follows (using "`//`" for **division with round down**)

```
t_paid = C//(S * R) // time paid for (rounded down)
ts_new = ts_0 + t_paid // advance the timestamp
```

**Example:** Suppose a transaction uses the `SLOAD` opcode to read the value of a storage cell of size 32 bytes with an outstanding rent of 2500 gas (about 1 year of rent is due). Since this exceeds the rent cap for `SLOAD`, the sender will be charged the cap of 1600 gas for this trie node. The node's `lastRentPaidTime` will be advanced by 

```
C = 1600 gas, S = (128 + 32) = 160 bytes, R = 1/2^21 gas/byte/sec
t_paid = 1600 // (160 * 1/2^21) = roundDown(2^21 *10) = 20,971,520 seconds (8 months approx)
```

**Rent for write Ops:** The above costs for trie *read* operations. However, rent tracking and collection can be implicitly triggered for write operations as well. This is because trie write operations such as `SSTORE` check for pre-existing values first. If a write operation creates a new trie node, then the node is assigned the timestamp of the current block. No rent is collected in advance.

**Security and DoS:** To increase the costs of IO-DOS attacks, the maximum rent cap is charged for attempting to read data from nodes that do not exist in the trie.

**Duration cap:** There is a limit on the amount of time for which rent is accrued. This is set at 3 years. If a node remains untouched for 3 years, then the amount of outstanding rent does not grow any further, it remains capped at that level. Letting rent accumulate forever seems unnecessarily punitive.

**Tracking and collection mechanics**

All trie nodes *accessed*, *modified* or *created* by a transaction are automatically checked and added to rent-tracking caches. Users cannot select or exclude individual rent payments. 

One way of implementing the system is to have three caches of value-containing nodes: 

1. a *set* of all trie nodes seen thus far during transaction execution
2. a *map* of all trie nodes whose rent timestamps are to be updated along with their individual (updated) timestamps. Some of these nodes will also have updated values (e.g `SSTORE`).
3. a *set* of newly created tries nodes. All of them will receive the timestamp of the current block.

These caches are passed along to all child calls, so that rent tracking does not kick in more than once for any node irrespective of call depth. After a transaction has been fully processed, the caches are iterated and the updated values and timestamps are commited to the state trie.

### Gas counters and accounting

The `usedGas` counter should include both execution and rent gas consumption.

- the `GAS` opcode must use `usedGas` (inclusive of rent) to provide a conservative estimate of how much gas is still available.
- Similarly, the combined value must be reported for transaction receipts and block level gas consumption.

However, a *separate gas counter*,  `usedRentGas`, should be also be implemented.

- If a transaction ends because of a OOG exception or REVERT, then 25% of the storage rent gas used so far (`usedRentGas`)  is consumed as compensation for IO costs. The remaining 75% of the gas marked as `usedRentGas` is refunded. Rent timestamps are not updated.
- a separate counter can also help implement `estimateGas` methods that report estimates with and without including estimated rent payments.

Note that a distinction between `usedGas` (execution + rent) and `usedRentGas` may be useful for research. Nodes can do that internally. But it would be useful to have rent payments listed separately in transaction receipts. These issues are left for future research.

#### Dependencies
The limits on variable costs in this proposal are based  on "re-priced" gascosts from a related proposal, RSKIP_repriceRead. While these proposals can be adopted independently, we think it is beneficial to make the current proposal dependent on RSKIP_repriceRead. Should the community choose to implement only the current proposal, then the numerical limits presented here may have to be revised.


## Rationale

The specification already provides much of the rationale. More information to be added here based on community feedback.

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

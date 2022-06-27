---
rskip: 207
title: Emergency Time-locks Refresh
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-01
---
# Emergency Time-locks Refresh


|RSKIP          | 207 |
| :------------ |:-------------|
|**Title**      |Emergency Time-locks Refresh|
|**Created**    |JAN-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

The time-locked emergency multisignature introduced in RSKIP201 requires that Powpeg UTXOs are periodically spent in order to prevent the time-lock expiration. 

This RSKIP proposes a mechanism for the Bridge to command this time-lock refresh efficiently.



## Motivation

The motivation for adding an time-locked emergency multisignature is presented in [RSKIP201](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md).

## Specification

Let B be the Bitcoin blockchain height as seen by the bridge. We define a UTXO that needs **refresh** to be one that is included in the Bitcoin blockchain at block height C where (B-C > P) where P is 26000 (a rounded average number of Bitcoin blocks produced over a 6 months period). 

The bridge contract maintains a FIFO queue of UTXOs called **oldUTXOs** as a double-linked list, where Powpeg UTXOs are ordered by block height of inclusion in the Bitcoin blockchain. The Bridge maintains an **oldest** and **newest** pointers to the first and last elements in the queue. 

An expiration checkpoint is defined as a periodic event where UTXOs are scanned and those that are close to expire are refreshed. Every time updateCollections() is called, a new procedure **checktimeLockExpiration()** is called before any other processing takes place. This method performs the following actions:

1. Let H be a storage variable containing an expiration checkpoint expressed as an RSK block height. Let D be a storage variable containing an expiration checkpoint expressed as a timestamp. Let B be the block height of the tip of the Bitcoin blockchain as seen by the bridge. If the current RSK block height is higher than H or the last RSK block timestamp is higher than D, then continue, else exit the checktimeLockExpiration() method.
2. Scan the oldUTXOs list from older to newer and unlink a maximal set of elements S that need refresh. All the UTXOs in the set S will be transferred to the current active powpeg address, packed in one or more transactions. Each transaction will have a single output belonging to the active Powpeg. The set S can be empty, in which case nothing is done. 
3. Set the next expiration checkpoint block height H to be the current RSK height plus 40000 (rounded two weeks of RSK blocks). 
4. Set the next expiration checkpoint datetime D to be the last RSK block timestamp plus 2 weeks.

### Peg-ins

Every time a new peg-in requires the registration of a Powpeg UTXO, this UTXO will be inserted with insertion-sort in the oldUTXOs linked list. Most of the time peg-ins will be notified to the Bridge in the order they are confirmed in Bitcoin, so the insertion sort will store the UTXO in the tail of the list, in constant time. However, it's possible that peg-ins are notified out-of-order, and in that case the insertion-sort may require scanning over the oldUTXO list.

### Peg-outs

The algorithm that chooses UTXOs for creating peg-outs should prioritize UTXOs that need refresh, to avoid the consumption of UTXOs only for the time lock refresh. We propose that the UTXO selection algorithm is modified to scan first oldUTXOs for UTXOs that need refresh, and if none is found, the algorithm continues scanning with any other prioritization algorithm. 

## Rationale



### Data structures

The selection of a linked-list to store the UTXOs responds to the need to test the expiration of the oldest elements quickly and add newer (or close to new) elements. Normally it's expected that elements are notified to the Bridge in order, so processing will generally be O(1). However, the Fast BTC Bridge use may lead to the late notification of old UTXOs, which makes sorting O(N). A priority queue data structure, such as a heap, can be used instead to obtain O(NlogN) insertion and O(NlogN) removal. However, we suggest that instead the simpler linked-list is used, and a variable amount of gas is charged to the caller, depending on the depth of the insertion sort. This requires that the Bridge pre-compile allows throwing OOG exceptions and charging gas on-the-fly during execution.

An alternative design is to perform a full UTXO sort once every two weeks, and avoid the complexity of maintaining a new data structure. This seems to be the best design if the number of UTXO is bounded and low (i.e. less than 24), which can be achieved with the existence of a peg-out batching mechanism and a UTXO consolidation mechanism.

### Bitcoin fees cost-efficiency

The proposed method does not check for expiration in each call to updateCollections() to prevent the generation of a constant stream of independent refresh transactions consuming high transaction fees, and reducing the UTXO availability, when there is no rush to refresh UTXOs that are far from expiring. 

The checkpoints work as a batching mechanism. If a different daily batching mechanism is implemented, then the checkpoints may no longer be necessary. However, peg refresh may still generate a stream of unnecessary re-peg transactions if there are no other transactions to batch with. It is undesired that transactions stay long in a batching queue because they consume UTXOs and therefore the peg-out process could become blocked.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Implementation

TBD

## Security Considerations

This RSKIP protects the RSK network from the expiration of the time-lock by two methods: block count and date. Each method prevent a different attack where miners tweak the block timestamps to delay the check and force an expiration. If the time-lock expired without a real emergency event affecting the Powpeg devices, then the signatories of the emergency multisig will have full access to the pegged funds, bypassing the security of the Powpeg.

If this RSKIP is not adopted, but the emergency multisignature is activated, it is still possible for users to extend the time-locks by performing several low-value peg-out operations until all UTXOs are consumed and recycled. While the community can force the refresh of UTXOs, the blockchain should not rely on this human-dependent behavior.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
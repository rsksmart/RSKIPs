---
rskip: 264
title: Simplified Emergency Time-locks Refresh
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-08
---
# Simplified Emergency Time-locks Refresh


|RSKIP          | 264 |
| :------------ |:-------------|
|**Title**      |Simplified Emergency Time-locks Refresh|
|**Created**    |AUG-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

The time-locked emergency multisignature introduced in RSKIP201 requires that Powpeg UTXOs are periodically spent in order to prevent the time-lock expiration. 

This RSKIP proposes a mechanism for the Bridge to refresh the time-locks of UTXOs. The proposed mechanism is simple and efficient if the Bridge already has a protocol to periodically consolidate the UTXOs into a small set.



## Motivation

The motivation for adding an time-locked emergency multisignature is presented in [RSKIP201](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md).


This RSKIP is a simplification of RSKIP207, which was written at a time where UTXO consolidation was not seen as an improvement that should be prioritized.

## Specification

Let B be the Bitcoin blockchain height as seen by the bridge. We define a UTXO that needs **refresh** to be one that is included in the Bitcoin blockchain at block height C where (B-C > P) where P is 26000 (a rounded average number of Bitcoin blocks produced over a 6 months period). 
The Bridge stores for every Powpeg UTXO (and in the same data structure used to hold the UTXOs) the height in the Bitcoin blockchain where this UTXO was created.

An expiration checkpoint is defined as a periodic event where UTXOs are scanned and those that are close to expire are refreshed. Every time updateCollections() is called, a new procedure **checktimeLockExpiration()** is called before any other processing takes place. This method performs the following actions:

1. Let H be a storage variable containing an expiration checkpoint expressed as an RSK block height. Let D be a storage variable containing an expiration checkpoint expressed as a timestamp. Let B be the block height of the tip of the Bitcoin blockchain as seen by the bridge. If the current RSK block height is higher than H or the last RSK block timestamp is higher than D, then continue, else exit the checktimeLockExpiration() method.
2. Scan the powpeg UTXOs and select a maximal subset of elements S that need refresh. All the UTXOs in the set S will be transferred to the current active powpeg address, packed in one or more transactions an one or more UTXOs following the UTXO consolidation strategy. The set S can be empty, in which case nothing is done. 
3. Set the next expiration checkpoint block height H to be the current RSK height plus 40000 (rounded two weeks of RSK blocks). 
4. Set the next expiration checkpoint datetime D to be the last RSK block timestamp plus 2 weeks.

If this proposal is implemented together with RSKIP265, then the UTXO Mantainance Account (UMA) defined in that RSKIP should be used to pay for UTXO refresh transaction fees. If the UMA does not have enough balance to pay for the fees, then all the UMA balance will be consumed and the remaining fees will be consumed from the peg balance.  

### Peg-outs

The algorithm that chooses UTXOs for creating peg-outs should prioritize UTXOs that need refresh, to avoid the consumption of UTXOs only for the time lock refresh. 

## Rationale

### Data structures

Since we assume the UTXOs are consolidated in a short list (i.e. no more than 40 elements), we don't optimize for fast access, or fast removal of elements. We assume we can periodically load the list of UXTOs, scan them, remove those that need refresh, and write back the list of the remaining elements to storage.

### Bitcoin fees cost-efficiency

The proposed method does not check for UTXO expiration in each call to updateCollections() to prevent the generation of a constant stream of independent refresh transactions consuming high transaction fees, and reducing the UTXO availability, when there is no rush to refresh UTXOs that are far from expiring. 

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

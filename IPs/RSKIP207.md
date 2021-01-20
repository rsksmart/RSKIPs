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

This RSKIP proposes a mechanism for the Bridge to command this time-lock refresh in a way that maintains high-availability of funds for peg-out, and low risk of time-lock expiration.



## Motivation

The motivation for adding an time-locked emergency multisignature is presented in RSKIP201.

## Specification

The bridge contract maintains a FIFO queue of UTXOs called **oldUTXOs** as a double-linked list, where Powpeg UTXOs are ordered by block height of inclusion in the Bitcoin blockchain. The Bridge maintains an **oldest** and **newest** pointers to the first and last elements. Every time updateCollections() is called a new procedure **checktimeLockExpiration()** is called before any other processing. In this procedure checks are performed in sequence:

1. Check if composition voting phase should be cancelled.

2. Check if its not undergoing a membership change protocol and if a timestamp D has been reached.


In check 1, if the Powpeg is undergoing a composition voting phase, and if this voting phase is taking more than 1 month then the composition change protocol is immediately cancelled and the check 2 will be performed. This ensures that the expiration check cannot be blocked by subsequent attempts to change the Powpeg composition.

In check 2, if both conditions are true, it will scan the oldUTXOs list from older to newer and unlink a maximal set of elements S where the inclusion block height for each element is lower than (D-6 months). All the UTXOs in the set S will be spent packed in one or more transactions, and moved into a single output per transaction, belonging also to the Powpeg. The set S can be empty. The next expiration check deadline D will be postponed by adding 1 month to it.

### Peg-ins

Every time a new peg-in requires the registration of a Powpeg UTXO, this UTXO will be inserted with insertion-sort in the oldUTXOs linked list. Most of the time peg-ins will be notified to the Bridge in the order they are confirmed in Bitcoin, so the insertion sort will store the UTXO in the tail of the list, in constant time. However, it's possible that peg-ins are notified out-of-order, and in that case the insertion-sort may require scanning over the UTXO list.

## Rationale

The selection of a linked-list to store the UTXOs responds to the need to retrieve the oldest elements quickly and add newer (or close to new) elements. Normally it's expected that elements are notified to the Bridge in order, so processing will generally be O(1). However, the Fast BTC Bridge use may lead to the late notification of old UTXOs. A priority queue data structure, such as a heap, can be used instead to obtain O(NlogN) insertion and O(NlogN) removal. However, we suggest that instead the simpler linked-list is used, and a variable amount of gas is charged to the caller, depending on the depth of the insertion sort. This requires that the Bridge pre-compile allows throwing OOG exceptions and charging gas on-the-fly during execution.



## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Implementation

TBD

## Security Considerations

This RSKIP protects the RSK network from the expiration of the time-lock. If the time-lock expired without a real emergency event affecting the Powpeg devices, then the signatories of the emergency multisig will have full access to the pegged funds, bypassing the security of the Powpeg.

If this RSKIP is not adopted, but the emergency multisignature is activated, it is still possible for users to extend the time-locks by performing several low-value peg-out operations until all UTXOs are consumed and recycled. While the community can force the refresh of UTXOs, the blockchain should not rely on this human-dependent behavior.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
---
rskip: 375
title: Use the pegout creation transaction hash as the key in the map structure that stores the pegout transactions waiting for signatures
created: 30-MAY-23
author: MI
purpose: Sca,Usa
layer: Core
complexity: 1
status: Adopted
description: 
---

|RSKIP          |375           |
| :------------ |:-------------|
|**Title**      |Use the pegout creation transaction hash as the key in the map structure that stores the pegout transactions waiting for signatures |
|**Created**    |30-MAY-23 |
|**Author**     |MI |
|**Purpose**    |Sca,Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

For the PowHSMs to sign a pegout transaction, the PowPeg node needs to provide the transaction to sign along with the Rootstock block number where the pegout was created. This information allows the PowHSM to validate that the transaction to be signed has enough PoW accumulated on top.

The mechanism currently used by the PowPeg to obtain the block where a pegout was created is not effective and has some disadvantages. This RSKIP proposes a simpler and more effective way to obtain the Rootstock block where a given pegout was created.

## Motivation

Since the Rootstock network launched in early 2018, Rootstock’s 2-way peg protocol has evolved from a federated model to what we call a PowPeg [1]. One of the most distinctive characteristics of the PowPeg is the introduction of special purpose hardware devices, called PowHSMs, to protect private keys. 

The goal of the PowHSMs is to remove any human control of the private keys comprising the federation multi-sig holding the Bitcoins locked in the RSK’s 2-way peg. Additionally, the PowHSMs follow a subset of Rootstock’s chain consensus rules, often referred to as SPV validation, and therefore signatures can only be commanded by cumulative proof of work. 

For a block to be registered in the PowHSM's Rootstock chain, it needs enough proof of work accumulated since the block creation. So every block registered in the PowHSM is considered final.

To sign a peg-out transaction, the PowHSM needs to have registered the block where the pegout transaction was created. The PowPeg node then sends to the PowHSM the block number where the pegout was created and the transaction to sign in order to validate and sign the pegout.

There is currently no easy way for the PowPeg node to obtain the Rootstock block where a pegout transaction was created. At the moment, it keeps an index in local storage that connects pegout transactions hash (without signatures) to the hash of the transaction where the pegout was created by the Bridge contract. This solution presents some disadvantages, for example, this index is kept encoded in a local file and if for any reason this file is deleted or corrupted it leaves the PowPeg incapacitated to sign pegouts.

This RSKIP proposes setting the Rootstock transaction hash where a pegout is created as the key in the map structure that holds the pegouts that have already accumulated enough confirmations and are now waiting for signatures. The PowPeg node then reads this map by accessing the Bridge storage and can get from it all the information needed to supply the PowHSM to sign.

## Specification

When a pegout transaction has enough confirmations in Rootstock and is moved into the map that holds the pegout transactions that are ready to be signed, use as the key of the map the hash of the Rootstock transaction where the pegout transaction was created.

This change should be handled as a consensus rule.

## Rationale

When the Bridge detects a pegout that has already accumulated the required amount of confirmations in Rootstock it moves it into a map structure that holds the pegout transactions that are ready to be signed. The value used as the key in this map is the hash of the Rootstock transaction where the pegout is confirmed (i.e. moved from waiting for confirmations to waiting for signatures). The rationale for this decision was to ensure that the key is unique since only one pegout is confirmed per transaction. Using the hash of the transaction where the pegout was created could result in duplicated keys since sometimes multiple pegout requests can be queued to be processed in a single Rootstock transaction.

Since the introduction of pegout batching feature as part of HOP network upgrade [2], multiple pegout requests can be included in a single pegout transaction. This ensures that it is not possible to have more than one pegout transaction created in the same Rootstock block.

## References

[1] [Rootstock developer portal](https://dev.rootstock.io/rsk/architecture/powpeg/)

[2] [RSKIP 271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

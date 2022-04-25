---
rskip: 199
title: registerBtcTransaction Is Public 
description: 
status: Adopted
purpose: Sca, Usa, Sec
author: MI <marcos@iovlabs.org>
layer: Core
complexity: 2
created: 2021-01-07
---
# registerBtcTransaction Is Public

|RSKIP          |199           |
| :------------ |:-------------|
|**Title**      |registerBtcTransaction Is Public |
|**Created**    |07-JAN-21 |
|**Author**     |MI |
|**Purpose**    |Sca,Usa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes making `registerBtcTransaction` bridge method public to increase decentralization in RSK. This will allow any user to register in the Bridge a bitcoin transaction that sends funds to the Federation, to complete the peg-in process. Currently, only Federation members are in charge of registering these types of transactions.

To increase security and performance, this RSKIP proposes the creation of an index in storage to obtain a bitcoin block hash for any given height in constant time.

## Motivation

Nowadays when a user wants to perform a peg-in in RSK, the first step is to send a transaction to the Federation address with the value that wants to peg-in. After the transaction is confirmed, Federation members are in charge of monitoring the Bitcoin network and asking the Bridge to register those transactions that send value to the Federation. 

With the proposed change a user can directly register his transaction in the Bridge contract making the peg-in a completely decentralized process.

## Specification

### Make `registerBtcTransaction` bridge method public

After this RSKIP activation, `registerBtcTransaction` will be a public method in the Bridge contract, available to be called by any user of the RSK network.

### Creation of a block index

An index will be created in storage that keeps a record of the corresponding bitcoin block hash to each height value after this RSKIP activation height. When users try to register a transaction this index will be used to get the corresponding block at the given height in constant time. Only transactions at a certain maximum depth before this RSKIP activation will be accepted.

## Rationale

When a transaction created before this RSKIP activation is registered there will be no entry in the block index for the given height. In that case, it is necessary to iterate through bitcoin blocks that are in storage until the given depth is reached and the block containing the registered transaction is found. 

Since this can be an expensive operation, there will be a limit set to the maximum depth before this RSKIP activation a transaction can be registered. This max depth will be reduced as the blockchain advances, eventually reaching 0 at which point only transactions created after this RSKIP activation will be accepted to register. This is to prevent possible DoS attacks by attempting to register very old transactions that would take a long time to look up in the storage.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

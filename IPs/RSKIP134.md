---
rskip: 134
title: Locking cap
description: 
status: Draft
purpose: Sca, Usa, Sec
author: JD (@josedahlquist)
layer: Core
complexity: 2
created: 2019-07-11
---
# Locking cap

|RSKIP          |134           |
| :------------ |:-------------|
|**Title**      |Locking cap |
|**Created**    |03-SEP-2019 |
|**Author**     |JD |
|**Purpose**    |Sca,Usa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

RSK has a two-way peg system to lock funds in the Federation, using BTC, which are then released as RBTC in RSK through the Bridge native contract. This system was implemented using a whitelisting process that adds a nuisance to the usability of the network.
`
This RSKIP defines a process to restrict the amount of funds locked in the Federation at any time. This new process enables fully decentralized peg-in, and aims to replace the previous peg-in whitelisting requirement. 

## Motivation

RSK was created as a second layer on top of Bitcoin, bringing smart-contract capabilities to the Bitcoin ecosystem, increasing the overall value of Bitcoin.
The gateway between BTC and RSK is the two-way peg. If the peg-ins process is burdersome, it adds unnecesary friction.

The proposed changes are required to remove the whitelisting requirement while reducing other risks.

## Specification

This RSKIP proposes two changes:

* Setting a limit for the funds locked in the Federation at any given time.
* Setting a mechanism to increase this limit.

These changes will be defined in the following sections.

### Setting locking cap

An initial locking cap should be configured and would be defined upon implementation. This cap should be stored in the Blockchain and should be modifiable from the Bridge.

### Amount of UTXOs waiting for signatures

During the BTC release process, the required UTXOs are taken from the Federation and removed from its wallet. This is correct, but being unable to use them doesn't mean those funds are not still in the Federation. This means we should consider the value of said UTXOs when checking the Federation balance.

### Rejecting locks that surpass the cap

A new mechanism should be put in place, to decide if the received lock should be processed or rejected.

To do this the Bridge would have to get the existing value of the wallets for the active and the retired, if any, Federations. Not only that, the amount of funds used in releases but waiting for signatures should also be considered. The addition of all these values should always be lower than or equal to the stored cap.

If the transaction that is under process makes the value of said wallets to be bigger than the limit we should create a release transaction right away, using the UTXOs that said transaction generated as the input for the release transaction. This transaction, as opposed to what happened with the not whitelisted operations, should NOT undergo the confirmations process, instead, it should be immediately queued waiting for the Federation members' signatures.

#### New method to get the current locking cap

The Bridge should implement a new method named *getLockingCap*. This method should return the current locking cap, represented in satoshis, stored in the Bridge.

#### New method to increase the locking cap

The Bridge should also implement another method named *increaseLockingCap*.

This method will receive a UINT parameter, let's call it *lockingCap*. The lockingCap value represents an amount in Satoshis to be used as the cap.
It returns a boolean value to indicate success or failure.

As for authorization, this method will be only callable by some preauthorized addresses stored as a Bridge configuration; if any other but the accepted addresses call this method, it will immediately return *false*.

The *lockingCap* value will be validated comparing it with the existing locking cap value stored in the blockchain, if the value for *lockingCap* is smaller than the existing value, the transaction will be rejected returning *false*. Additionally, the increment has a limitation established as a configuration in the Bridge (e.g.: if the configured value is 2, it represents a max increment of existing value times two), if the new value surpasses it, it will also be rejected. Any value in between these boundaries will be stored and the method will return *true*.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

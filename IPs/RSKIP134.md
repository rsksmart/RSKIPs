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

RSK has a two way peg system to lock funds in the Federation, using BTC, that are then released as RBTC in RSK through the Bridge native contract. This system was implemented using a whitelisting process that adds a nuisance in the usability of the network.

We will define a process that replaces the whitelisting by restricting the funds locked in the Federation at any time, taking us closer to a fully decentralized two way peg, while avoiding unnecesary risks for the network users.

## Motivation

RSK was created as a second layer on top of Bitcoin, bringing the smart contracts capabilities to the ecosystem, increasing the overall value for the end user.
The gateway between BTC and RSK is the two way peg, if this system is complex and troublesome, it defeat its own purpose.

We propose changes to remove the whitelisting, while minimizing the risks for the users.

## Specification

To remove the whitelisting we have to:
* Remove all the whitelisting logic.
* Setting a limit for the funds locked in the Federation at any given time.
* Setting a mechanism to increase this limit.

We will define each of these changes in the following sections.

### Removing whitelisting

Removing the whitelisting implies two main changes, removing the method from the Bridge API and removing the existing controls while processing a lock.

We have 7 methods in existence that handle the whitelisting operations which include: adding, removing, getting entries, and finally, disabling the overall whitelisting system. These methods won't have any purpose once the whitelisting is disabled for good.

Removing the whitelisting can be done in two ways:
* Setting a block delay to indicate when the whitelisting should be disabled.
* Removing its logic using a hard fork.

We have chosen to do the latter to ensure that one mechanism is replaced by the other, to avoid any risk.

### Setting locking cap

We have to store an initial locking cap that should be defined upon implementation. This cap should be stored in the Blockchain and should be modifiable from the Bridge.

### Rejecting locks that surpass cap

Once we have removed/disabled whitelisting, we need to put in place the new mechanism to decide if the received lock should be processed or rejected. To do this we will have to get the existing value of the wallets for the active and the retired, if any, Federations. The addition of both wallets should always be lower than or equal to the stored cap.
If the transaction that is under process makes the vaue of said wallets to be bigger than the limit we should create a release transaction right away, using the UTXOs that said transaction generated as the input for the release transaction. This transaction, as opposed to what happened with the non whitelisted operations, should NOT undergo the confirmations process, instead, should be immediately queued waiting for the Federation members signatures.

#### New method to get the current locking cap

We need a new method in the Bridge named *getLockingCap*. This method should return the current locking cap, represented in satoshis, stored in the Bridge.

#### New method to increase locking cap

We have to expose a new method in the Bridge named *increaseLockingCap*. This method will receive a UINT parameter, let's call it *lockingCap*. This method returns a boolean value to indicate success or failure. It will be only callable by a number of preauthorized addresses stored as a Bridge configuration; if any one but the accepted addresses call this method, it will immediately return *false*.
The *lockingCap* value will be validated comparing it with the existing locking cap value stored in the blockchain, if the value for *lockingCap* is smaller than or equals than the existing value, the transaction will be rejected returning *false*, elsewhere it will stored and the method will returning *true*.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

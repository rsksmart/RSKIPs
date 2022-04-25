---
rskip: 35
title: Managing BridgeMaster Federation Members
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-02-02
---

# Managing BridgeMaster Federation Members

|RSKIP          |35           |
| :------------ |:-------------|
|**Title**      |Managing BridgeMaster Federation Members|
|**Created**    |02-FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

This document describes the procedure to add or remove bridge federators, and the procedure which federators can use to change their private keys. This document does not address the "extended" federation, what provides other services. 

## Discussion

As changing the federation is an expensive process, the system must allow to apply several changes to a federation.

# **Specification**

BridgeMaster Federators are the federators that are involved in the multi-signature of the Bitcoin bridge. From now on, "federators" is limited to this kind of federators members.

One of the drawbacks of current design is that the bridge contract stores the public keys of the federators. In the future, and via hard-forks, the federators may control other bridge systems.

Therefore is good practice to re-factor the BridgeMaster into at least three contracts. 

* BridgeFedFront

* BridgeFed (one or more active)

* BridgeMaster

All services provided by these contracts must respect the ABI interface.

The following diagram shows how different contracts interact.

![image alt text](./RSKIP35/federationRSKIP35.png)

## BridgeFedFront

The BridgeFedFront contract will provide the following services:

**1. getCurrentFederation() → bridgeFed :BridgeFed**

Get the federation that is active and recommended now to send funds to.

**2. getActiveFederations() → array of BridgeFed**

Returns the list of federations that can still receive funds, but funds may be moving to the current fed.

**3. getNewFederation() → bridgeFed :BridgeFed**

The first time called by a member, it creates a new Fed clone of the current fed into a new BridgeFed contract. This is a candidate federation. The second time it’s called by a member it returns the latest newly created Fed that is still inactive. The new fed will auto-activate after 48 hours. To prevent auto-activation, the BridgeFed voteInvalidate() method must be called by the majority of federators of the previous Fed. Once the new fed is activated, any call to getNewFederation() will result in the creation of yet another fed.

## BridgeFed

Each federator is represented by an id. 

**1 .getMemberCount() → uint count**

Returns the number of active federators in this federation.

**2. getMemberBlockchainPubKeys() → array of public keys**

Returns an array of all the public keys of this federation.

**3. getMemberBlockchainPubKey(uint id) → PublicKey**

Returns a public key of a specific member

**4. getMemberIdentityAddressess() → array of address**

Returns an array of all the public address of this federation long term management contracts.

**5. getMemberIdentityAddress(uint id) → Address**

Returns the address of the account of contract that a federator uses to control his he federation role. 

**6. getMemberURL(uint id) → String**

Returns the member URL by id.

**7. getMemberIdByIdentityAddress(Address addr) → uid**

**8. getMemberIdByBlockchainPubKey(PublicKey key) → uid**

Return the public key of a single member of the federation, by mid.

**9. getMemberMetadataHash(uint id) → uint256**

**//deprecated 10. CanReplaceFederation(uint256 hash,array of Signatures) → boolean**

**10. VoteReplaceFederation(uint256 addressNewFed) → boolean**

Returns true if all the signatures sign the same hash and the majority of federation signatures are present and valid.

These methods are necessarily equal for all federations, but must be implemented in the first federation in the following way. 

The new federation contract contains by default all the members of the old federation and provides the following methods that will be called by RSK on creation.

**11.AddMember(**

**PublicKey newBlockchainPubKey,**

**Address newIdentityAddress,**

**String memberURL)**

Adds a new member to this federation. It’s acceptable that there are two active polls. However, only ONE of them can be added, the other represents a mistake or an attack, and there can’t be two federators with the same public key. There can be two members with the same memberName but different public keys. This is acceptable.

**8. RemoveMember(PublicKey aPubKey)**

It must be noted that a member can be added and then removed, or vice-versa. Even multiple times.

 
**14. GetFedAddress() → String**

This method is only available when this federation is activated. Returns the Bitcoin address (format to be determined) of this federation

**15. SetFedAddress(addr String)**

This method can be called only by the BridgeMaster contract to set the Bitcoin address for this federation, after the federation is activated. Once the value is set, it cannot be changed anymore.

Important: add EXTGETCODEHASH opcode to be able to check that the new federation code is equal to the old federation code.

## BridgeMaster

The BridgeMaster will hold a pointer to the BridgeFedFront contract.

The BridgeMaster will keep active any federation that still has funds associated, but will try to move any UTXO to the latest active federation. Once a new active federation appears, the BridgeMaster will accept payments to old federations for a period of 72 hours (configurable). Afterwards, any received bitcoins will not be accepted and those amounts will be locked forever. Once an old federation has no more UTXOs associated and the 72 hours period has elapsed, it will be removed from BridgeFedFront and the BridgeMaster will dispose all information related to it.

Notes just Oscar will understand - FAQ

**Q: why don't allow miner a stake in the federation members?**

A: I don’t understand the question

**Q: Think bitcoin hardfork / softfork activation**

A: I don’t understand the question

**Q: communication, monitoring and action tool**

 

**Q: how do I notify to other federators I want to change the federation and why?**

Notification is done through two different channels:

- Whenever a new federation is created getNewFederation() method, an event is logged. All FedNodes listen to these events and therefore can notify the federators that something is happening. This DOES NOT solve the problem of how federators (users) are notified. I think that the FedNode should have an e-mail alert system or be able to communicate with other standard alert systems.

- Generally, with the exception of key changes, all modifications of the federation will be coordinated through out of band channels (e-mail, forums, etc). So when the federation is created, all federators will already know why they are creating the new federation.

 - how a federator is notified? There is no human monitoring the blockchain, so we should provide an alert system or something

 - how do a federator vote? how the transaction is created? there will be a program?

**Q: What are the reasons to update the federation?**

A: The following are possible causes for a new federation to emerge:

*  new member added

*  federator no longer interested in participate

*  federator key compromised or maybe compromised

*  several federator keys compromised or maybe compromised

*  1 federator misbehaves

*  Several federators misbehaves

*  Situation changes after a new proposal and need a proposal update

* Consolidate all funds (e.g. for an audit) when there are thousands of UTXOs.. 

- Initial Federation: put it in the genesis block. 

  option 1: in the format used to export/import contract state

  option 2: in a human readable format

**Q: Block my key: Show we add a message sent that invalidates a key without a new federation being created?**

A: I don’t see the benefit of such invalidation. If 51% of the keys are compromised by an attacker, they can spend the bridge funds without any help from the BridgeMaster. If less than 51% of the keys are compromised, then they can’t, but also they can’t force the BridgeMaster to do anything malicious, except not to vote. This functionality makes sense only if the same keys were used by federators to vote on something else.

- New Federation Proposal

  - List of federators (pubkey, name)

  - number of signatures required

  - delayUntilActivated (in or rsk blocks) - aka time1

  - delayUntilPreviousFederationAddressInvalidated (in rsk blocks) - aka time2

  - proposalHash (calculated)

  - votes: List of public keys that voted for this proposal

- Activation

 - before delayUntilActivated: previous federation can vote/sign

 - after delayUntilActivated: new federation can vote/sign

- Vote proposalHash

**Q:What are the  Federation states?**

A: Osky proposed states

*  not full voted proposals 

*  fully voted but not yet activated

    * SDL: I don’t think this state exists, since it becomes active at the moment it is fully voted

    * Oscar: Whatever big change you do on a blockchain should give users enough notice. Eg when a soft/hard fork is deployed to bitcoin, it is programmed to be active N blocks after the threshold has been met. Then, users/developers have time to update their systems / do whatever they need. A federation proposal / not fully voted is not considered notice since it is almost equal to "no news".

* active

* in deprecation 

* deprecated(SDL: or better to call it "inactive?)

SDL proposed simpler states. The states are:

* Inactive: can’t  receive funds

* Active: can receive funds

If it is inactive, there are two possible sub-states:

* Candidate (not yet activated)

* Deprecated (activated and deactivated)

If it is active, there are three possible sub-states:

* New: funds are being transferred to this federation

* Current: all funds are in this federation

* Old: funds are being taken out from this federation

If there  is a single federation in state "active, new" there can’t be any federation in state “active, current”

I have however the following doubt: while funds are being moved from an old federation to a new federation there may be some UTXOs in an old fed, and some UTXOs in a new fed, therefore it is possible that moving funds back to Bitcoin is impossible until all old funds are transferred to the new fed. We could allow mixing UTXOs from an old fed and a new fed into a single unlock transaction, but this adds a lot of complexity. 

**- how to move proposals from one state to another on time1 and time2:**

  transaction by any federator invoking a contract method: activateNewFederation / deactivateOldFederation

**Q: what happens if a new proposal is required while another proposal is not fully active?**

  (not yet activated, previous fed address still valid, etc)

A: If funds are being transferred from an old federation A to a new federation B and suddenly a new-new federation C is activated. This case MUST be handled. I suppose in this case what will happen is this:

A: active, old

B: active, old

C, active, new

Immediately after C is activated, this happens:

- Funds that are being transferred from A to B (but not yet confirmed) are waited

- Funds still unmoved in A will be moved as soon as possible to C (instead of marked to be moved to B)

**Q: How are UTXOs moved from a federation A to a federation B?**

A: When B is activated, a Bitcoin transaction will be created containing the maximum possible of old UTXOs up to a certain transaction size limit. This transaction will be proposed to federators to sign. If not all old UTXOs fit into this transaction, then a second transaction will be proposed to sign and so on (if this occurs at the same time, or after the poll of the previous transaction will depend on how the current BridgeMaster contract works, I don’t know if it can handle parallel polls).

 

**The following things do not look like questions to me:**

- Things todo

 - update bridge

 - update BtcLockClient

 - Do a call/internal tx from one precompiled contract to another (bridge to Federation contract)

   We have never done that yet

- Federation address migration

 - until delayUntilActivated no changes are applied

 - After delayUntilActivated and before delayUntilPreviousFederationAddressInvalidated

   - both federation adddress can receive funds

   - Federators should inform the bridge of the txs to both addresses

   - Bridge should check that UTXOs belongs to the new federation when selecting UTXOs to spend

     (Another idea: keep a list of which UTXOs belong to each federation address)

   - We should spend just new UTXOs during the migration process

 - UTXOs should be transferred to the new address

   - option 1: as they arrive. Problems: high tx cost and created UTXO may be bellow dust output

   - option 2: after delayUntilPreviousFederationAddressInvalidated. 

     Problem: if there are too many UTXO tx may be huge or not fit in a block

     Problem: we need to keep track of the old federation until we finish the UTXO migration

     - bridge new methods: createMigrationTx (may create several txs) / addMigrationSiganture

     - federators monitor there is a migrrationTx to sign and sign it

     - feerators monitor there is a migrationTx to broadcast and broadcast it

   - who pays the bitcoin miner fee of migration tx/s? 

     - if nobody pays that, there will be more SBTC than BTC the federation will have

     - Idea: RSK account created for that purpose, funded by RSK Labs

  - After delayUntilPreviousFederationAddressInvalidated

   - The new address is THE address

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
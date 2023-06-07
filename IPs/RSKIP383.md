---
rskip: 383
title: Increase POWpeg activation age
created: 10-MAY-23
author: JD
purpose: Fair
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |383           |
| :------------ |:-------------|
|**Title**      |Increase POWpeg activation age |
|**Created**    |10-MAY-23 |
|**Author**     |JD |
|**Purpose**    |Fair |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

Everytime the POWpeg composition changes a migration takes place. This migration consists of three stages:
- Preparation: A new POWpeg has been voted but the new POWpeg has no control yet.
- Activation: The new POWpeg is now active but the funds from the retiring POWpeg have not started to migrate yet.
- Migration: The UTXOs from the retiring POWpeg are migrated to the active POWpeg

Each of these stages have a certain duration set in Rootstock blocks to enable users and/or systems to act on them.

This RSKIP proposes a change in the duration of said stages for Mainnet to make it Fair for all users to determine whether they want or not to keep participating in the Bridge.

## Motivation

As part of the correction of the Hop POWpeg redeemscript (see [RSKIP-353](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP353.md)) a change in the migration stage was introduced (see [RSKIP-357](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP357.md)). This change extended the migration stage to ensure no funds would be at risk during the non-standard script managed UTXOs were migrated to the standard redeem script.
Now, a new RSKIP proposes to restablish the migration duration (see [RSKIP-374](Reestablish the number of block confirmations for a PowPeg migration period)).

With all these changes, a new question arised, what about users wanting to decide whether they want to keep participating in the 2wp or not? When a new POWPeg composition is voted, community members may or may not agree with this.

The users have 18,500 rootstock blocks (approximately 6 and half days) to remove their funds from the POWpeg if they don't agree with the new POWpeg composition.
This is not a short time, but we need to take into account the time it takes to actually be able to remove funds from the POWpeg.

Let's assume user Alice has a certain amount of RBTC staked in a DeFi protocol on Rootstock. When Alice finds out about a POWpeg composition change taking place she may not want to keep on participating. She would have to withdraw her funds from the protocol, and then proceed to perform a peg-out.
With the addition of the peg-out batching the peg-out will take up to 3hs to be processed, and then the usual 34hs to be confirmed, signed and broadcasted. In total, Alice may take up to 40hs until she gets her BTC back on Bitcoin.
Been optimistical Alice has enough time to get out.
This may not be the case if the UTXOs are few. If Alice has a high amount of RBTC to peg-out, there is a chance there are not enough UTXOs in the Bridge ATM (i.e. UTXOs are consumed in a pending peg-out or a change UTXO is still waiting to be confirmed).
In this case Alice may have to double the waiting time or worse. A change UTXO takes near 60 hours to be registered in the Bridge (around 40 hs for it to be broadcasted to Bitcoin and around 17 more hours to be confirmed), this means that Alice may have to wait 100 hours 

There are many RSKIPs proposing improvements to avoid having Alice in the waiting line for long (e.g. [RSKIP-265](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md), [RSKIP-270](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP270.md) or [RSKIP-272](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP272.md)) but those are not implemented yet.

For all the reasons listed above, we consider relevant to extend the activation age to 40320 Rootstock blocks (approximately 14 days).

## Specification

Change the federationActivationAge in the Bridge to 40320 blocks.

This change should be handled as a consensus rule.

No further change is contemplated in this RSKIP.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

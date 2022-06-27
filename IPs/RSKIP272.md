---
rskip: 272
title: Bridge UTXO Management Account
description: 
status: Draft
purpose: Sec, Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-08
---
# Bridge UTXO Management Account


|RSKIP          | 272 |
| :------------ |:-------------|
|**Title**      |Bridge UTXO Management Account|
|**Created**    |AUG-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec, Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

One of the problems of the RSK bridge is that peg-out fees are unknown to the user until the peg-out is executed, and the fees are decremented from the peg-out amount. The user has no control of the final amount to be received. This RSKIP proposes a new algorithm for the Bridge to pay for all peg-outs, and charge the users a predictable dynamically-adjusted fee. The results is a much better user experience.

## Motivation

The current RSK bridge can suffer from a number of problems described in [RSKIP265](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md). In this RSKIP we're mostly concerned with the variable peg-out costs. The proposed solution is to provide to the user a method to query the fees to be paid, and charge that amount to the user. This is done by maintaining a separate Bridge balance used for peg-out fee payments. In the future this balance can be used for other UTXO management activities, such as refresh of UTXO time-locks. 

## Specification

The Bridge contract will try to maintain a **UTXO Management Account (UMA)** with a balance high enough to pay for peg-outs, UTXO consolidations (if [RSKIP171](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md) is activated), time-lock refresh (if [RSKIP264](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP264.md) is activated) and other maintenance operations without the need to charge peg-in or peg-out users. 

The bridge computes two values: Fi and Fo. Fi represents the cost in fees of consuming a single Powpeg input, and Fo is the cost of one P2SH output.

We define a threshold T(n) as the fee cost of a transactions consuming n Powpeg Bitcoin inputs. We compute T(n) as (n\*Fi+2\*Fo).
If the UMA balance is lower than T(C), then peg-outs users will be charged 10% more, and the Bridge will move the collected extra fee to the UMA account. If the balance is higher than T(B), then the peg-outs will be charged 10% less. The values C and B are defined in the following way:

If peg-out batching is activated ([RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md)) then the constants selected are B=60 and C=30.

If peg-out batching is not activated, then B=600 and C=300.

If [RSKIP270](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP270.md) is activated, then B and C must match those defined in RSKIP270.

The UMA account amount will be stored in the bridge contract, and a new method `getUMABalance()` will be enabled to query its current balance. If [RSKIP265](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md) is activated together with this RSKIP, this account will also be used to pay for UTXO refresh required to avoid the emergency time-lock from expiring.

### Peg-out Fees

The peg-out fees are fixed to let user known in advance how much they may need to pay for each peg out. A new bridge method `getFeesForNextPegOutEvent()` is added for this purpose.


This method returns (adjustFactor*avgFee), where, avgFee is the average fee paid by each transaction in the the past 24 peg-out events, except for the last event which is excluded. To compute avgFee, when an peg-out event ends, the Bridge records the average fee per transaction that would have been required not to modify the UMA balance. The value for avgFee is updated each time a peg-out event ends.

At the RSKIP activation block, avgFee is set to (Fi\*1.5+2*+Fo). The value of avgFee is updated once the first 24 peg-out events after activation have been executed.

When the peg-out events is triggered, all users waiting to peg-out are immediately charged `getFeesForNextPegOutEvent()`. 

The value adjustFactor is 0.9 if the UMA balance is higher than T(B), 1.1 if the UMA balance is lower than T(C) and 1 otherwise.

The peg-out fees for the peg-out transactions are paid from the UMA. If the UMA does not have enough funds, then the remaining funds are paid from the peg balance.  


## Rationale

### Fixed vs variable peg-out fees

Having variable peg-out fees, together with batching, degrades the user experience, as the user does not have an immediate good estimation of the fees that the peg-out will collect. This proposal allows peg-out fees to vary according to a moving average that is updated on each peg-out event, but does not take into account the last event. Therefore the user should not expect fees to vary between querying the Bridge and performing the peg-out even if one peg-out event occurs in between. 

The `getFeesForNextPegOutEvent()` fees are chosen to keep enough funds in the UMA to avoid consuming from the peg balance.


### Magic Numbers

The lower and upper bounds are chosen according to RSKIP265.

If there is no batching, then the number of UTXOs should be higher to avoid reaching the limit under normal operation. Assuming 450 UTXOs available, the peg can sustain a peg-out every 8 minutes continuously, which is a much higher rate that the current rate. While the values of B and C do not offer protection against a DoS attack, much higher values of B and C can lead to excessive CPU processing at peg-out time.

If we analyze this RSKIP together with [RSKIP264](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP264.md), [RSKIP270](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP270.md) and [RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md), the choice of B and C enable to perform different UTXO management actions consuming the UMA balance, such as:

* collect UMA fees for 60 inputs allows the Bridge to perform 15 consolidations when the UTXO set reaches 80 UTXO, removing up to 45 UTXOs from UTXO set.  
* Assuming there are UTXO 60 in the Bridge, if the UMA is used to refresh emergency time-locks, then 100% of the UTXO will be able to be refreshed in the absence of peg-outs. 
* If the UTXO expirations are uniformly distributed over the year, the current threshold allows all UTXOs for one year without peg-outs. 
* The current threshold also allows to perform a full Powpeg migration. 

At current BTC/USD prices and with the current Powpeg multisig structure, the average balance of the UMA would be 800 USD. 

### Fee Estimation

We've solved the problem of how users can estimate the peg-out transaction costs before the peg-out transaction is built. Now the Bridge computes the fees for wallets to show in their UIs. However, if two peg-out events occur after the query but before the peg-out command, the fees may have changed. 

It's also possible to provide a new version of the releaseBTC() method that supports an argument to indicate the maximum amount of fees that the user is willing to pay, and abort otherwise.

### UTXO Management Account

The UMA serves as a fee buffer to smooth peg-out fees and also to pay for Bitcoin fees in exceptional events, such as forced UTXO consolidations or Powpeg migrations. In the future, it can also serve to provide an automatic fee bumping mechanism. The fees would be bumped automatically after 20 hours if a peg-out transaction containing a change output has not been informed to the Bridge. 

### Simulations

<u>The algorithms and thresholds presented here need to be validated by simulations before the RSK community can approve this RSKIP. This section will show the simulation results. This RSKIP will be updated with improved algorithms and thresholds after simulations are completed.</u>

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Security Considerations

This RSKIP resolves an existing, but not critical, platform vulnerability.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

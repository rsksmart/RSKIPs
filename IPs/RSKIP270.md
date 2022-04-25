---
rskip: 270
title: Bridge UTXO set size management
description: 
status: Draft
purpose: Sec, Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-08
---
# Bridge UTXO set size management


|RSKIP          | xxx |
| :------------ |:-------------|
|**Title**      |Bridge UTXO set size management|
|**Created**    |AUG-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec, Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

One of the problems of the RSK bridge is that it doesn't have any means to prevent the proliferation and fragmentation of UTXOs. This can lead to a number of cost, usability, efficiency and security problems defined in [RSKIP265](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md). This RSKIP is part of a series of RSKIPs that address these problems (RSKIP265, and RSKIP270 to RSKIP273). In particular, this RSKIP proposes a new protocol for the Bridge to shrink or expand the UTXOs to keep the size of the UTXO set within a predetermined range. This is done by splitting amounts (expansion) or joining amounts (consolidation). Lower bound and upper bound thresholds are used to avoid frequent UTXO management operations. 

## Motivation

The current RSK bridge can suffer from a number of problems described in RSKIP265. In this RSKIP we're mostly concerned with UTXO proliferation,  UTXO fragmentation and  UTXO uneven amount distributions.
The solution to UTXO proliferation/fragmentation is periodic, user-triggered or peg-in/peg-out triggered consolidations.
The solution to uneven amount distribution is to rebalance amounts during consolidations.

## Specification

### UTXO Consolidation/Expansion

Every time after a peg-in transaction is registered by the Bridge, the Bridge will count the number of UTXOs and perform the following actions:

* **Consolidation**: If the number of UTXOs is higher than A, then (M+1) UTXOs will be immediately consolidated in a peg-out/peg-in transaction with a single Powpeg output. The additional inputs will be chosen greedily to consume low amounts.

Before a peg-out is to be built, the Bridge will count the number of UTXOs and perform the following actions:

* **Consolidation**: If the number of UTXOs is higher than B, the peg-out transaction will consume at least M inputs. The inputs will be chosen with the algorithm described in RSKIP265, which should add a low-value input. 
* **Expansion**: If the number of UTXOs is lower or equal to C, and the change amount is higher than 1 BTC, then M change outputs will be added, split the change amount evenly between the (N+1) change outputs for the Powpeg. 

If UTXO Management Account is activated ([RSKIP272](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP272.md)), then the additional cost of creating a peg-in-peg-out transaction or adding adding an input or an output to a peg-out transaction will be paid by the UMA, otherwise it will be charged to the current peg-in or peg-out users.

If peg-out batching is activated ([RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md)) then the constants selected are A=80, B=60 and C=30. 

If peg-out batching is not activated, then A=800, B=600 and C=300.

If UMA is activated ([RSKIP272](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP272.md)) then M=3, N=3.

If UMA is not activated, them M=1, N=2.


## Rationale

We define an algorithm that tries to:

- Keep the number of UTXOs between two bounds. 
- Do not generate too many  UTXO consolidation or expansion transactions. 

### Magic Numbers

If peg-out batching is not activated, then having a low number of UTXOs allows a DoS attack to be performed on the Bridge, by consuming all UTXOs quickly. Therefore A, B and C, are 10 higher than with batching.

If UMA is activated, then the UMA can pay for the high cost of consolidation, and therefore M and N can be set to much higher values. If UMA is not activated, then the consolidation must be done is lower UTXOs because each input added almost doubles the cost of the peg-out transaction.

### Hysteresis

The correction procedures kick-in when the number of UTXOs is far below or far above the desired number. This reduces the risks of oscillations that will increase the transaction fees consumed on average. 

The correction method based on consolidation on peg-ins will only be triggered if there is a stream of peg-ins but no peg-outs. Because in this case consolidation requires creating a new peg-out transaction, the process consumes N+1 inputs to amortize the cost in fees. Since the other inputs consumed are chosen to be low valued, we don't expect peg-outs to be blocked after a stream of peg-ins.

### Simulations

<u>The algorithms and thresholds presented here need to be validated by simulations before the RSK community can approve this RSKIP. This section will show the simulation results. This RSKIP will be updated with improved algorithms and thresholds after simulations are completed.</u>



## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Security Considerations

This RSKIP resolves an existing, but not critical, platform vulnerability.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


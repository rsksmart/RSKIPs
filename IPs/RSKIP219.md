---
rskip: 219
title: New minimum values for peg-in and peg-outs
description: 
status: Adopted
purpose: Usa
author: MI <marcos@iovlabs.org>
layer: Core
complexity: 1
created: 2021-07-12
---

# New minimum values for peg-in and peg-outs

|RSKIP          |219           |
| :------------ |:-------------|
|**Title**      |New minimum values for peg-in and peg-outs |
|**Created**    |12-JUL-21 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes a reduction in the minimum amounts of BTC / RBTC accepted by RSK for peg-in and peg-out operations. In the case of peg-outs, the minimum value is computed for each peg-out operation taking into consideration the current fee per kb value configured in RSK. This proposal also ensures that after the fee is paid, the remainder is enough for the user to be able to operate in the network.

## Motivation

With the increasing value of BTC in USD, the current minimum values to peg-in and peg-out are too high for the majority of users. As an example, the current price of BTC is around U$S30.000 and the minimum peg-in value is 0.01 BTC. At this price, a user needs to be willing to transfer U$S300 just to start using the platform.

## Specification

This RSKIP proposes 4 different changes to improve the UX of the RSK platform:

1. Reduce the minimum peg-in amount to half its current value. From 0.01 BTC to 0.005 BTC.
2. Reduce the minimum peg-out amount to half its current value. From 0.008 RBTC to 0.004 RBTC.
3. Define the minimum value that the user should receive after paying release transaction fees.
4. Make minimum peg-out value inclusive (the amount to peg-out should greater than or **equal to** minimum value)

### Reduce the minimum peg-in amount to half its current value

After this RSKIP activation, the minimum value accepted to perform a peg-in is **0.005 BTC**. If less than this amount is sent to the federation address then those funds will be ignored by RSK, and will be lost.

### Reduce the minimum peg-out amount to half its current value

After this RSKIP activation, the minimum value accepted to perform a peg-out is **0.004 RBTC**. If less than this amount is sent to the bridge address then those funds will be refunded to the user.
Additionally, a calculation of the fees to be paid by the user is made. If after subtracting the fees, the amount to be received by the user is less than **%80** of the fee value then the peg-out will not be completed and funds will be returned to the user.


## Rationale

With the proposed changes the platform will become more accessible to users by reducing barriers to entry as well as barriers to exit.
Making the minimum peg-out value **inclusive** will prevent many user errors and loss of funds that have occurred in the past.
Finally, ensuring that the user will receive at least a percentage of the peg-out value after paying transfer fees is especially important when performing peg-outs for values close to the minimum. If at a given point in time BTC transaction fees are particularly high and the execution of the peg-out would imply that a big portion of the user funds goes into paying fees, it's best not to execute the peg-out and refund the user. Then the user can choose to either try to peg-out more funds at once or wait until BTC fees are lower to try again.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

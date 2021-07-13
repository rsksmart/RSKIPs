# New minimum values for peg-in and peg-outs

|RSKIP          |219           |
| :------------ |:-------------|
|**Title**      |New minimum values for peg-in and peg-outs |
|**Created**    |12-JUL-21 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a reduction in the minimum amounts of BTC / RBTC required to perform peg-in and peg-out operations in RSK. In the case of peg-outs, the minimum value is calculated for each operation taking into consideration the current fee per kb value in the network. The idea is to ensure the amount paid in fees by the user represents only a small percentage of the total value that's being released.

## Motivation

With the increasing value of BTC in USD, the current minimum values to start using RSK (and to retire) tend to be rather high for the majority of users. As an example, the current price of BTC is around U$S30.000 and the minimum peg-in value is 0.01 BTC. At this price, a user would need to be willing to lock U$S300 just to start using the platform.

## Specification

This RSKIP proposes 4 different changes to improve and facilitate users the use of the RSK platform:

1. Reduce the minimum peg-in amount to half its current value. From 0.01 BTC to 0.005 BTC.
2. Reduce the minimum peg-out amount to half its current value. From 0.008 RBTC to 0.004 RBTC.
3. Define the minimum percentage of the total release value that the user should receive after paying fees.
4. Make minimum peg-out value inclusive (the amount to peg-out should greater than or **equal** minimum value)

### Reduce the minimum peg-in amount to half its current value

After this RSKIP activation, the minimum value accepted to perform a peg-in is **0.05 BTC**. If less than this amount is sent to the federation address then those funds will be burnt.

### Reduce the minimum peg-out amount to half its current value

After this RSKIP activation, the minimum value accepted to perform a peg-out is **0.004 RBTC**. If less than this amount is sent to the bridge address then those funds will be refunded to the user.
Additionally, a calculation of the fees to be paid by the user is made. If after subtracting the fees, the amount to be received by the user is less than the configured percentage of the total value trying to peg-out then the peg-out will not be completed and funds will be returned to the user.
The percentage is established on each network independently. The suggested value is 80%.
 

## Rationale

With the proposed changes the platform will become more accessible to users by reducing barriers to entry as well as barriers to exit.
Making the minimum peg-out value **inclusive** will prevent many user errors and loss of funds that have occurred in the past.
Finally, ensuring that the user will receive at least a percentage of the peg-out value after paying transfer fees is especially important when performing peg-outs for values close to the minimum. If at a given point in time BTC transaction fees are particularly high and the execution of the peg-out would imply that a big portion of the user funds goes into paying fees, it's best not to execute the peg-out and refund the user. Then the user can choose to either try to peg-out more funds at once or wait until BTC fees are lower to try again.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

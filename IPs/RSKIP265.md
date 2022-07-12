---
rskip: 265
title: Bridge UTXOs Coin Selection
description: 
status: Draft
purpose: Sec, Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-08
---
# Bridge UTXOs Coin Selection


|RSKIP          | 265 |
| :------------ |:-------------|
|**Title**      |Bridge UTXOs Coin Selection|
|**Created**    |AUG-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec, Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

One of the problems of the RSK bridge is that it doesn't have any means to prevent the proliferation and fragmentation of UTXOs. This can lead to a number of cost, usability, efficiency and security problems. This RSKIP proposes new algorithm to improve the selection of UTXOs. This RSKIP also specifies specific changes when activated together with [RSKIP207](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP207.md) [RSKIP264](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP264.md), [RSKIP270](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP270.md), [RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md) and [RSKIP272](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP272.md). 

## Motivation

The current RSK bridge can suffer from a number of problems:

* **Denial-of-Service**: If too many peg-outs are commanded in a short timeframe, then the users may need to wait tens of hours for the funds to be available. This is because peg-out transactions require a confirmation period of approximately 37 hours and change UTXOs that were recently included in peg-outs need to be registered with the Bridge again, with requires 100 Bitcoin confirmations. Only afterward the returning change is again usable by the Bridge and new peg-outs can consume them. A more advanced attack can block the peg for longer periods by looping peg-in and peg-out transactions. This attack works because peg-outs requests are queued, so a sufficiently resourced attacker can queue enough small-valued peg-outs to block the peg. The cost of the attack is the financial cost of locked bitcoins and the bitcoin transaction fees involved.
* **UTXO proliferation**: Each peg-in creates a new UTXO. Each peg-out on average consumes more UTXOs than it creates (but only slightly less). However, currently the platform receives more peg-ins than peg-outs it creates, which leads to a slow but steady proliferation of UTXOs. The only force that counteracts this proliferation is the Powpeg migration process, which consolidates all UTXOs into a single output (or a few outputs if not all inputs can be added into a single transaction). Higher number of UTXOs means that the Bridge must scan a higher number of inputs. This is  scan would be lineal if the Bridge used a greedy algorithm, but currently is worse because the Bridge sorts the UTXOs by hash (which provides no benefit at all). The UTXO proliferation leads to higher CPU and storage resource consumption. First, all UTXOs are currently loaded into memory each time a peg-out transaction needs to be built. Second, the same list of UTXOs with some UXTOs removed is written back to storage after a peg-out transaction is built. Finally, during the processes to build a peg-out transaction, the Bridge must scan all UTXOs looking for valid inputs to cover the funds to release.
* **UTXO fragmentation**: Each time a peg-out occurs, an UTXO amount is sliced into two: the amount to pay to a user, and a change amount. Change amounts generally get smaller and smaller, until they are consumed or considered "dust" and they stay forever in the UTXO set. Dust outputs are UTXOs which costs more to spend them than the value they provide. Currently, if a peg-out transaction would create a dust output, this output is omitted from the transaction and the change amount is charged to the user that performs the peg-out, which leaves the dust UTXO forever in the Bridge UTXO set. Also fragmentation increases the cost of peg-out if the Bitcoin fees/kilobyte price is high at the time of peg-out. Historically, the Bitcoin transaction cost has followed Bitcoin price change (first derivative). In other words, it is high when Bitcoin price increases, but decreases when Bitcoin price does. This means that the best strategy to keep the UTXO set small is to defragment at specific times, but not necessarily to defragment it as soon as possible. The advantage of defragmenting at peg-in time (by consolidating the recently received UTXO with a preexisting one) is that the consolidation cost can be transferred to the user that chose to perform the peg-in and the cost is bounded because the number of inputs in the consolidation transaction can be fixed at 2. When consolidation is performed at peg-out time, the number of inputs can be high if the UTXOs amount is not uniformly distributed over the common peg-out amount range. In either case, what is important is that the user can make an informed decision of what fees she will be paying for peg-in and peg-out, and it is much easier to estimate input consolidation costs at peg-in than at peg-out.
* **UTXO uneven amount distributions**: if there is a peg-in transaction P of a very high amount of bitcoins, followed of a peg-out of an amount that is higher than all other UTXO amounts, yet much lower than the amount in P, then a high amount of bitcoins will be blocked for 54 hours until the change UTXO is registered again.  If more of similar peg-out operations repeat, then a high amount of bitcoins may be locked for a long period, even if decreasing in value, blocking other peg-outs. We call this the "ladder" attack. The underlying problem can also presents itself spontaneously if the normal functioning of the peg tends to produce an uneven distribution of amounts.
* **UTXO set shrink**: if there are many more peg-outs than peg-ins, then the number of available UTXOs can be depleted, leading to long waits for the change amounts to be confirmed into the peg again.
* **Varying peg-out costs:** peg-outs are queued, and peg-out transactions are built every 3 minutes. When that happens, peg-out fees are computed, and the user is charged accordingly by subtracting fees from their peg-out amounts. This brings two problems: 1) The user does not know the exact fee being charged in advance and 2) The exact amount to be received cannot be preestablished. 
* **UTXO refresh preference**: If RSKIP264 is activated, then the coin selection algorithm should prefer consuming older UTXO to avoid the need for UTXO refresh transactions.
* **Peg-out transactions fee bumping:** Bitcoin mempool congestion produce spikes in Bitcoin transaction fees. Peg-outs may be left waiting in the mempool for hours or days. This not only affect the user performing the peg-out but also other future peg-out users as the Bridge cannot spend the change UTXO. RSKIP207 proposed a method for fee-bumping that requires changes in the coin selection algorithm.

This RSKIP proposes a new UTXO selection algorithm that solves most of these problems with an heuristic and efficient algorithm.

## Specification

We describe the new coin selection algorithm. This algorithm varies if other RSKIPs are activated together with this RSKIP.

We define usersPegOutValue to the sum of all user peg-out amounts that must be serviced in a peg-out transaction. 

We define pegOutValue as usersPegOutValue plus k\*(Fi+(U+1)\*Fo). We set k=1 (this value is changed if RSKIP241 is activated). U is the number of user peg-outs being processed.

### Single-Pass Scoring Function

We define score(u) to be a scoring function of a UTXO u. The algorithm will try to find an UTXO u with the minimum score for a peg-out transaction with a single input u.

The scoring function is:

score(u) = u.amount*10/pegOutValue, which basically chooses the UTXO with lowest amount. 

The score is an integer value, and divisions are integer divisions. The function is changed if RSKIP264 is activated.

### Algorithm

To select inputs for a peg-out transaction, the Bridge performs at most three steps:

1. **Single-input greedy**: The Bridge will scan all UTXOs and try to find the UTXO with a value higher than the pegOutValue but with the lowest score. If the found UTXO has a score lower than 20, the corresponding UTXO is added as input to the peg-out transaction and the process stops here.
2. **Lower Amounts, Multi-input expensive**: The Bridge will sort UTXOs by amount, and starting from the UTXO with the lowest amount closest to pegOutValue, it will scan the UTXOs from higher to lower amounts, adding UTXOs to the transaction until the total value added is higher than pegOutValue+Fi\*W or the input count reaches Q, where W is the number of inputs added over 1. If the value is covered with Q or less inputs, the process stops here. Q is set to 4 (this value changes if RSKIP270 is activated)   
3. **Higher Amounts, Multi-input expensive**: The Bridge continues using the sorted UTXOs, and going from the highest amount to the lowest amount, it add inputs until the value added is higher than pegOutValue+Fi\*W.



### Changes if co-activated with RSKIP241

We set k=10. 

### Changes if co-activated with RSKIP264

The scoring function is: 

score(u) = (u.amount*10/usersPegOutValue + 3650/daysToTimeLockExpiration(u))/2

daysToTimeLockExpiration(u) is always rounded up, and the minimum value it returns is 1, even if the time-lock is already expired.

### Changes if co-activated with RSKIP270

if no input consolidation is required Q is set to 4. If input consolidation is required, then Q is set to 8.

If the peg-out must expand outputs , and before any of the coin selection attempts is made, the Bridge will scan all UTXOs and add the one with the lowest amount to the peg-out transaction. Then the value of the added UTXO will be subtracted from the pegOutValue. If pegOutValue is still positive, the algorithm will proceed finding additional inputs. 

If the peg-out must consolidate outputs, then the process will perform the steps in a different order: 2, 1, and 3. Step 2 is performed using Q=8. 

### Changes if co-activated with RSKIP272

The value usersPegOutValue  is computed after `getFeesForNextPegOutEvent()` has been substracted from each amount. All fees collected will be added to the UTXO Maintenance Account.

Fees for the transaction will be paid from the UMA. If the UMA does not have enough funds, then the remaining funds will be paid from the peg balance. If RSKIP272 is not activated, fees will be substracted evenly from every user peg-out output.


## Rationale

We define a algorithms that implement heuristics that try to:

- Keep the number of UTXOs between two bounds. 
- Keep a Pareto distribution of amounts over the UTXOs.
- Keep the amount of bitcoins on UTXOs being consumed low, to prevent peg-outs to be blocked waiting for change amounts to return to the peg balance.
- If RSKIP241 is activated, then try to consume extra bitcoins in inputs to be able to bump fees.
- If RSKIP264 is activated, then try to consume inputs that need time-lock refresh.

### Minimization of Value in Transit

Moving additional value apart from the user selected peg-out amounts always carry a certain risk that the change amounts cannot be quickly pegged-in due to the lack of transaction inclusion. This can happen during sudden mempool congestions. 
To reduce this risks, the coin selection algorithm tries to minimize change, yet it behaves mostly greedy. If UTXO amounts are uniformly distributed, it should not consume more than the double of the peg-out amount. 
The proposed coin selection algorithm does not try to find the optimum solution. Finding the best combination of UTXOs is probably as hard as solving the [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem).

### Simulations

<u>The algorithms and thresholds presented here need to be validated by simulations before the RSK community can approve this RSKIP. This section will show the simulation results. This RSKIP will be updated with improved algorithms and thresholds after simulations are completed.</u>

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 



## Security Considerations

This RSKIP resolves potential DoS attacks on the current coin selection algorithm if the number of UTXOs increases several orders of magnitude.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
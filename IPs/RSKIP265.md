# Bridge UTXOs Selection, Consolidation and Batching


|RSKIP          | 265 |
| :------------ |:-------------|
|**Title**      |Bridge UTXOs Selection, Consolidation and Batching|
|**Created**    |AUG-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec, Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

One of the problems of the RSK bridge is that it doesn't have any means to prevent the proliferation and fragmentation of UTXOs. This can lead to a number of cost, usability, efficiency and security problems. This RSKIP proposes new protocols for the Bridge to consolidate peg-ins into a small set when the number of UTXOs grows over a threshold. It also proposes that the Bridge should expand the UXTO set if the number of UXTOs drops below another threshold. Since a low number of UTXOs can be consumed easily, this RSKIP can only achieve security against denial of service attacks if activated concurrently with a method of peg-out output batching, which is also proposed in this RSKIP. We also solve the problem of variable peg-out fees, which degrades the user experience.

## Motivation

The current RSK bridge can suffer from a number of problems:

* **Denial-of-Service**: If too many peg-outs are commanded in a short timeframe, then the users may need to wait tens of hours for the funds to be available. This is because peg-out transactions require a confirmation period of approximately 37 hours and change UTXOs that were recently included in peg-outs need to be registered with the Bridge again, with requires 100 Bitcoin confirmations. Only afterward the returning change is again usable by the Bridge and new peg-outs can consume them. A more advanced attack can block the peg for longer periods by looping peg-in and peg-out transactions. This attack works because peg-outs requests are queued, so a sufficiently resourced attacker can queue enough small-valued peg-outs to block the peg. The cost of the attack is the financial cost of locked bitcoins and the bitcoin transaction fees involved.

* **UTXO proliferation**: Each peg-in creates a new UTXO. Each peg-out on average consumes more UTXOs than it creates (but only slightly less). However, currently the platform receives more peg-ins than peg-outs it creates, which leads to a slow but steady proliferation of UTXOs. The only force that counteracts this proliferation is the Powpeg migration process, which consolidates all UTXOs into a single output (or a few outputs if not all inputs can be added into a single transaction). Higher number of UTXOs means that the Bridge must scan a higher number of inputs. This is  scan would be lineal if the Bridge used a greedy algorithm, but currently is worse because the Bridge sorts the UTXOs by hash (which provides no benefit at all). The UTXO proliferation leads to higher CPU and storage resource consumption. First, all UTXOs are currently loaded into memory each time a peg-out transaction needs to be built. Second, the same list of UTXOs with some UXTOs removed is written back to storage after a peg-out transaction is built. Finally, during the processes to build a peg-out transaction, the Bridge must scan all UTXOs looking for valid inputs to cover the funds to release.

* **UTXO fragmentation**: Each time a peg-out occurs, an UTXO amount is sliced into two: the amount to pay to a user, and a change amount. Change amounts generally get smaller and smaller, until they are consumed or considered "dust" and they stay forever in the UTXO set. Dust outputs are UTXOs which costs more to spend them than the value they provide. Currently, if a peg-out transaction would create a dust output, this output is omitted from the transaction and the change amount is charged to the user that performs the peg-out, which leaves the dust UTXO forever in the Bridge UTXO set. Also fragmentation increases the cost of peg-out if the Bitcoin fees/kilobyte price is high at the time of peg-out. Historically, the Bitcoin transaction cost has followed Bitcoin price change (first derivative). In other words, it is high when Bitcoin price increases, but decreases when Bitcoin price does. This means that the best strategy to keep the UTXO set small is to defragment at specific times, but not necessarily to defragment it as soon as possible. The advantage of defragmenting at peg-in time (by consolidating the recently received UTXO with a preexisting one) is that the consolidation cost can be transferred to the user that chose to perform the peg-in and the cost is bounded because the number of inputs in the consolidation transaction can be fixed at 2. When consolidation is performed at peg-out time, the number of inputs can be high if the UTXOs amount is not uniformly distributed over the common peg-out amount range. In either case, what is important is that the user can make an informed decision of what fees she will be paying for peg-in and peg-out, and it is much easier to estimate input consolidation costs at peg-in than at peg-out.

* **UTXO uneven amount distributions**: if there is a peg-in transaction P of a very high amount of bitcoins, followed of a peg-out of an amount that is higher than all other UTXO amounts, yet much lower than the amount in P, then a high amount of bitcoins will be blocked for 54 hours until the change UTXO is registered again.  If more of similar peg-out operations repeat, then a high amount of bitcoins may be locked for a long period, even if decreasing in value, blocking other peg-outs. We call this the "ladder" attack. The underlying problem can also presents itself spontaneously if the normal functioning of the peg tends to produce an uneven distribution of amounts.

The solution to the denial-of-service attack is peg-out queuing batching. 
The solution to UTXO proliferation/fragmentation is periodic, user-triggered or peg-in/peg-out triggered consolidations.
The solution to uneven amount distribution is to rebalance amounts during consolidations.

## Specification

We define a algorithms that implement heuristics that try to:

* Keep the number of UTXOs between two bounds. 
* Keep a Pareto distribution of amounts over the UTXOs.
* Do not generate too many  UTXO consolidation or expansion transactions. 
* Reduce the fees consumed by these transactions,  by increasing or decreasing the UTXO set size by at least 3 units simuntanously. 
* Keep the amount of bitcoins on UTXOs being consumed low, to prevent peg-outs to be blocked waiting for change amounts to return to the peg balance.
* If RSKIP241 is activated, then try to consume extra bitcoins in inputs to be able to bump fees.
* If RSKIP264 is activated, then try to consume inputs that need time-lock refresh.

### Batching

The Bridge contract will wait T=3 hours to create a peg-out transaction. During the waiting time, if more peg-out requests are received, they will be queued so that all of them can be batched in a single peg-out transaction. When the deadline is reached, a peg-out event will be triggered. Under normal usage, the Bridge would create no more than 8 peg-outs events a day. Under a rush hour of peg-outs, the Bridge may need to create more than one peg-out transaction simultaneously per peg-out event to reduce the transaction size. This peg-out splitting does not differ from the current protocol.

### UTXO Consolidation/Expansion

Every time after a peg-in transaction is registered by the Bridge, the Bridge will count the number of UTXOs and perform the following actions:

* **Expansion**: If the number of UTXOs is lower than 30 and then peg-in amount is higher than 0.1 BTC, then the UTXO registered will be immediately consumed in a peg-out transaction that splits the input amount into two equally sized Powpeg outputs.

* **Consolidation**: If the number of UTXOs is higher than 80, then 4 UTXOs will be immediately consolidated in a peg-out-peg-in transaction with a single Powpeg output. The additional inputs will be chosen greedily to consume low amounts. The fee for this consolidation UTXO will be consumed from the consolidation account (defined in next section).

Before a peg-out is to be built, the Bridge will count the number of UTXOs and perform the following actions:

* **Consolidation**: If the number of UTXOs is higher than 60, the peg-out transaction will consume at least 2 inputs. The inputs will be chosen with the algorithm described in the next section, to add enough funds to pay the peg-out amount and fees, but not higher than that. The additional cost will be charged to the current peg-out users.

* **Expansion**: If the number of UTXOs is lower or equal to 30, and the change amount is higher than 1 BTC, the change amount will be split evenly between two outputs for the Powpeg. The additional cost will be charged to the current peg-out users.


### UTXO Maintenance Account 

The Bridge contract will try to maintain a **UTXO Maintenance Account (UMA)** with a balance high enough to command some UTXO consolidations and other maintenance operations without the need to charge peg-in or peg-out users. This account will first be used to consolidate UTXOs after peg-ins, when consolidation could not occur in user-paid peg-outs. We define a threshold T(n) as the fee cost of a transactions consuming n Powpeg Bitcoin inputs. We compute T(n) as (n\*Fi+2\*Fo) (see definitions of Fi and Fo in the next section) 
If the UMA balance is lower than T(30), then peg-outs users will be charged 10% more, and the Bridge will move the collected extra fee to the UMA account. If the balance is higher than T(60), then the peg-outs will be charged 10% less. 
The UMA account amount will be stored in the bridge contract, and a new method `getUMABalance()` will be enabled to query its current balance. If RSKIP265 is merged together with this RSKIP, this account will also be used to pay for UTXO refresh required to avoid the emergency time-lock from expiring.

### Peg-out Fees

The peg-out fees will be fixed to let user known in advance how much they may need to pay for each peg out.
The bridge computes two values: Fi and Fo. Fi represents the cost in fees of consuming a single Powpeg input, and Fo is the cost of one P2SH output.

A new bridge method `getPegOutFeesForNextEvent()` will be added.


This method will return UMA_adjustFactor*avgFee, where, avgFee is the average fee paid by each transaction in the the past 24 events, except for the last event which is excluded. To compute avgFee, when an peg-out event ends the Bridge will record the average fee per transaction that would have been required not to modify the UMA balance. The value for avgFee will be updated each time a peg-out event ends.

At the RSKIP activation block, avgFee will be set to (Fi\*1.5+2*+Fo). The value of avgFee will bee updated once the first 24 peg-out events after activation have been executed.

When the peg-out events is triggered, all users waiting to peg-out will be immediately charged `getPegOutFeesForNextEvent()`. 

UMA_adjustFactor is 0.9 if the UMA balance is higher than T(60), 1.1 if the UMA balance is lower than T(30) and 1 otherwise.


### Peg-out Coin selection Algorithm

We first describe the new coin selection algorithm. This algorithm varies depending on the need for UTXO consolidation or expansion.

We define usersPegOutValue to the sum of all user peg-out amounts that must be serviced in a peg-out transaction, after `getPegOutFeesForNextEvent()` has been substracted from each amount. All fees collected will be added to the UTXO Maintenance Account.

We define pegOutValue as usersPegOutValue plus k\*(Fi+(U+1)\*Fo). If RSKIP241 is implemented together with this proposal, then we set k=10. If RSKIP241 is not implemented, then we set k=1. U is the number of user pegouts being processed.

We define score(u) to be a scoring function of a UTXO u. The algorithm will try to find an UTXO u with the minimum score for a peg-out transaction with a single input u.

If RSKIP264 is activated, then the scoring function is: 

score(u) = (u.amount*10/usersPegOutValue + 3650/daysToTimeLockExpiration(u))/2

daysToTimeLockExpiration(u) is always rounded up, and the minimum value it returns is 1, even if the time-lock is already expired.

If RSKIP264 is not activated, the scoring function is:

score(u) = u.amount*10/pegOutValue, which basically chooses the UTXO with lowest amount. 

The score is an integer value, and divisions are integer divisions.

To select inputs for a peg-out transaction, the Bridge performs three steps:

1. **Single-input greedy**: The Bridge will scan all UTXOs and try to find the UTXO with a value higher than the pegOutValue but with the lowest score. If the found UTXO has a score lower than 20, the corresponding UTXO is added as input to the peg-out transaction and the process stops here.

2. **Lower Amounts, Multi-input expensive**: The Bridge will sort UTXOs by amount, and starting from the UTXO with the lowest amount closest to pegOutValue, it will scan the UTXOs from higher to lower amounts, adding UTXOs to the transaction until the total value added is higher than pegOutValue+Fi\*W or the input count reaches Q, where W is the number of inputs added over 1. If the value is covered with Q or less inputs, the process stops here. Q is set to 4 if no input consolidation is required. If input consolidation is required, then Q is 8.  

2. **Higher Amounts, Multi-input expensive**: The Bridge continues using the sorted UTXOs, and going from the highest amount to the lowest amount, it add inputs until the value added is higher than pegOutValue+Fi\*W.

When the peg-out must expand outputs, and before any of the coin selection attempts is made, the Bridge will scan all UTXOs and add the one with the lowest amount to the peg-out transaction. Then the value of the added UTXO will be subtracted from the pegOutValue. If pegOutValue is still positive, the algorithm will proceed finding additional inputs. 

When the peg-out must consolidate outputs, then the process will perform the steps in a different order: 2, 1, and 3. Step 2 is performed using Q=8. 

In all cases the fees for the transaction will be paid from the UMA. If the UMA does not have enough funds, then the remaining funds will be paid from the peg balance. The `getPegOutFeesForNextEvent()` fees are chosen to keep enough funds in the UMA to avoid consuming from the peg balance. 


## Rationale

### Fixed vs variable peg-out fees

Having variable peg-out fees, together with batching, degrades the user experience, as the user does not have an immediate good estimation of the fees that the peg-out will collect. This proposal allows peg-out fees to vary accorindg to a moving average that is updated on each peg-out event, but does not take into account the last event. Therefore the user should not expect fees to vary between querying the Bridge and performing the peg-out even if one peg-out event occurs in between. 


### Magic Numbers

The peg out process lasts approximately 37 hours. The time of a peg-in is approximately 17 hours. Therefore the round-trip time of a change amount is 54 hours on average. We want to have 40% of the UTXOs available at all times to be able to efficiently choose inputs. Assuming a 50% waste in value moved on each peg-out, then the 60% of UTXOs in transit will equate to 90% of the peg balance in transit, which means that any peg-out of an amount lower than 10% of the pegged balance will be processed immediately, while higher amounts may need to wait up to 54 additional hours. 
In 54 hours of round-trip time, we have a maximum of 18 slots to perform peg-outs. Assuming an average of 1.5 inputs are consumed per peg-out transaction, and one transaction per batched peg-out, we consume 27 UTXOs. Those 27 UTXOs must not represent more than 60% of the all UTXOs being rotated, so the minimum number of rotating peg-outs must be 45. 

The choice to collect UMA fees for 60 inputs allows the Bridge to perform 15 consolidations when the UTXO set reaches 80 UTXO, removing up to 45 UTXOs from UTXO set.  Assuming there are UTXO 60 in the Bridge, if the UMA is used to refresh emergency time-locks, then 100% of the UTXO will be able to be refreshed in the absence of peg-outs. Also if the UTXO expirations are uniformly distributed over the year, the current threshold allows all UTXOs for one year without peg-outs. The current threshold also allows to perform a full Powpeg migration. 

At current BTC/USD prices and with the current Powpeg multisig structure, the average balance of the UMA would be 800 USD. 

Note that if UTXO are very close to expiration, the Bridge commands the refresh of expiring UTXO even if that requires consuming BTCs from the peg balance, at the expense of "diluting" the value of the rBTC compared to BTC, although the dilution percentage would probably be negligible and won't affect the 1:1 parity. This mechanism is specified in RSKIP265.

### Hysteresis

The correction procedures kick-in when the number of UTXOs is far below or far above the desired number. This reduces the risks of oscillations that will increase the transaction fees consumed on average. On the other side, sorting 60 elements is only 4 times slower than sorting 30 elements, so there is no risks of a much higher CPU consumption. 
The correction method based on consolidation on peg-ins will only be triggered if there is a stream of peg-ins but no peg-outs. Because in this case consolidation requires creating a new peg-out transaction, the process consumes 4 inputs to amortize the cost in fees. Since the other inputs consumed are chosen to be low valued, we don't expect peg-outs to be blocked after a stream of peg-ins.

### Fee Estimation

We've solved the problem of how users can estimate the peg-out transaction costs before the peg-out transaction is built. Not Bridge computes the fees for wallets to show in their UIs. However, if two peg-out events occur after the query but before the peg-out command, the fees may have changed. 

It's also possible to provide a new version of the releaseBTC() method that supports an argument to indicate the maximum amount of fees that the user is willing to pay, and abort otherwise.

### UTXO Maintenance Account

The UMA serves as a fee buffer to smooth peg-out fees and also to pay for Bitcoin fees in exceptional events, such as forced UTXO consolidations or Powpeg migrations. In the future, it can also serve to provide an automatic fee bumping mechanism. The fees would be bumped automatically after 20 hours if a peg-out transaction containing a change output has not been informed to the Bridge. 

### Simulations

<u>The algorithms and thresholds presented here need to be validated by simulations before the RSK community can approve this RSKIP. This section will show the simulation results. This RSKIP will be updated with improved algorithms and thresholds after simulations are completed.</u>

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Minimization of Value in Transit

Moving additional value apart from the user selected peg-out amounts always carry a certain risk that the change amounts cannot be quickly pegged-in due to the lack of transaction inclusion. This can happen during sudden mempool congestions. 
To reduce this risks, the coin selection algorithm tries to minimize change, yet it behaves mostly greedy. If UTXO amounts are uniformly distributed, it should not consume more than the double of the peg-out amount. 
The proposed coin selection algorithm does not try to find the optimum solution. Finding the best combination of UTXOs is probably as hard as solving the [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem).

## Security Considerations

This RSKIP resolves an existing, but not critical, platform vulnerability.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
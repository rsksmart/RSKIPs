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

One of the problems of the RSK bridge is that it doesn't have any means to prevent the proliferation and fragmentation of UTXOs. This can lead to a number of cost, usability, efficiency and security problems. This RSKIP proposes new protocols for the Bridge to consolidate peg-ins into a small set when the number of UTXOs grows over a threshold. It also proposes that the Bridge should expand the UXTO set if the number of UXTOs drops below another threshold. Since a low number of UTXOs can be consumed easily, this RSKIP can only achieve security against denial of service attacks if activated concurrently with a method of peg-out output batching, which is also proposed in this RSKIP.

## Motivation

The current RSK bridge can suffer from a number of problems:

* **Denial-of-Service**: If too many peg-outs are commanded in a short timeframe, then the users may need to wait tens of hours for the funds to be available. This is because peg-out transactions require a confirmation period of approximately 37 hours and change UTXOs that were recently included in peg-outs need to be registered with the Bridge again, with requires 100 Bitcoin confirmations. Only afterward the returning change is again usable by the Bridge and new peg-outs can consume them. A more advanced attack can block the peg for longer periods by looping peg-in and peg-out transactions. This attack works because peg-outs requests are queued, so a sufficiently resourced attacker can queue enough small-valued peg-outs to block the peg. The cost of the attack is the financial cost of locked bitcoins and the bitcoin transaction fees involved.

* **UTXO proliferation**: Each peg-in creates a new UTXO. Each peg-out on average consumes more UTXOs than it creates (but only slightly less). However, currently the platform receives more peg-ins than peg-outs it creates, which leads to a slow but steady proliferation of UTXOs. The only force that counteracts this proliferation is the Powpeg migration process, which consolidates all UTXOs into a single output (or a few outputs if not all inputs can be added into a single transaction). Higher number of UTXOs means that the Bridge must scan a higher number of inputs. This is  scan would be lineal if the Bridge used a greedy algorithm, but currently is worse because the Bridge sorts the UTXOs by hash (which provides no benefit at all). The UTXO proliferation leads to higher CPU and storage resource consumption. First, all UTXOs are currently loaded into memory each time a peg-out transaction needs to be built. Second, the same list of UTXOs with some UXTOs removed is written back to storage after a peg-out transaction is built. Finally, during the processes to build a peg-out transaction, the Bridge must scan all UTXOs looking for valid inputs to cover the funds to release.

* **UTXO fragmentation**: Each time a peg-out occurs, an UTXO amount is sliced into two: the amount to pay to a user, and a change amount. Change amounts generally get smaller and smaller, until they are consumed or considered "dust" and they stay forever in the UTXO set. Dust outputs are UTXOs which costs more to spend them than the value they provide. Currently, if a peg-out transaction would create a dust output, this output is omitted from the transaction and the change amount is charged to the user that performs the peg-out, which leaves the dust UTXO forever in the Bridge UTXO set. Also fragmentation increases the cost of peg-out if the Bitcoin fees/kilobyte price is high at the time of peg-out. Historically, the Bitcoin transaction cost has followed Bitcoin price change (first derivative). In other words, it is high when Bitcoin price increases, but decreases when Bitcoin price does. This means that the best strategy to keep the UTXO set small is to defragment at specific times, but not necessarily to defragment it as soon as possible. The advantage of defragmenting at peg-in time (by consolidating the recently received UTXO with a preexisting one) is that the consolidation cost can be transferred to the user that chose to perform the peg-in and the cost is bounded because the number of inputs in the consolidation transaction can be fixed at 2. In any case consolidation is performed at peg-out time, the number of inputs can be high if the UTXOs amount is not uniformly distributed over the common peg-out amount range. In either case, what is important is that the user can make an informed decision of what fees she will be paying for peg-in and peg-out, and it is much easier to estimate input consolidation costs at peg-in than at peg-out.

* **UTXO uneven amount distributions**: if there is a peg-in transaction P of a very high amount of bitcoins, followed of a peg-out of an amount that is higher than all other UTXO amounts, yet much lower than the amount in P, then a high amount of bitcoins will be blocked for 54 hours until the change UTXO is registered again.  If more of similar peg-out operations repeat, then a high amount of bitcoins may be locked for a long period, even if decreasing in value, blocking other peg-outs. We call this the "ladder" attack. The underlying problem can also presents itself spontaneously if the normal functioning of the peg tends to produce an uneven distribution of amounts.

The solution to the denial-of-service attack is peg-out queuing batching. 
The solution to UTXO proliferation/fragmentation is periodic, user-triggered or peg-in/peg-out triggered consolidations.
The solution to uneven amount distribution is to rebalance amounts during consolidations.

## Specification

We define a series of heuristics that try to balance the number of UTXOs and distribution of amounts without generating events that consume a high percentage of the bitcoins locked, which can block the peg.

### Batching

The Bridge contract will wait T=3 hours to create a peg-out transaction. During the waiting time, if more peg-out requests are received, they will be queued so that all of them can be batched in a single peg-out transaction. Under normal usage, the Bridge would create no more than 8 peg-outs transactions a day. Under a rush hour of peg-outs, the Bridge may need to create more than one peg-out transaction simultaneously to reduce the transaction size. This peg-out splitting does not differ from the current protocol.

### UTXO Consolidation/Expansion

Every time after a peg-in transaction is registered by the Bridge, the Bridge will count the number of UTXOs and perform the following actions:

* **Expansion**: If the number of UTXOs is lower than 30 and the peg-in amount is higher than 0.1 BTC, then the UTXO registered will be immediately consumed in a peg-out transaction that splits the input amount into two equally sized Powpeg outputs.

* **Consolidation**: If the number of UTXOs is higher than 80, then then 4 UTXOs will be immediately consolidated in a peg-out-peg-in transaction with a single Powpeg output. The additional inputs will be chosen greedily to consume low amounts.

Before a peg-out is to be built, the Bridge will count the number of UTXOs and perform the following actions:

* **Consolidation**: If the number of UTXOs is higher than 60, the peg-out transaction will consume at least 2 inputs. The inputs will be chosen with the algorithm described in the next section, to add enough funds to pay the peg-out amount and fees, but not higher than that.

* **Expansion**: If the number of UTXOs is lower or equal to 30, and the change amount is higher than 1 BTC, the change amount will be split evenly between two outputs for the Powpeg.

### Coin selection Algorithm

First, the bridge computes the amount Fi, which represents the cost in fees of  consuming a single Powpeg input, and Fo which is the cost of two outputs (user payment and change). To select inputs for a peg-out transaction, the Bridge performs three attempts:

1. **Single-input greedy attempt**: The Bridge will scan all UTXOs and try to find one whose amount is higher than the requested peg-out amount (plus Fi+Fo) but lower than 2 times the value. If found, the process stops here.

2. **Single-input expensive attempt**: The Bridge will sort UTXOs by amount, and starting from the lowest amount it will try to find the first item whose amount is higher than the peg-out amount plus Fi+Fo. If found, the process stops here.

2. **Multi-input expensive attempt**: The Bridge continues using the sorted UTXOs, and starting from the highest amount it add inputs until the value is higher than the peg-out amount plus Fi*W+fo, where W=(1+the number of inputs already added).

The single-input attempts will only be tried if peg-out is not in UTXO expansion mode.

### Important Note

This RSKIP lacks the specification of the changes to these coin selection algorithms that should be made if RSKIP265 is implemented together with this RSKIP. RSKIP265 requires that the coin selection algorithm does some minimum effort to consume UTXOs whose emergency time-lock is close to expire, to avoid creating dedicated time-lock refresh transactions.


## Rationale

### Magic Numbers

The peg out process lasts approximately 37 hours. The time of a peg-in is approximately 17 hours. Therefore the round-trip time of a change amount is 54 hours on average. We want to have 40% of the UTXOs available at all times to be able to efficiently choose inputs. Assuming a 50% waste in value moved on each peg-out, then the 60% of UTXOs in transit will equate to 90% of the peg balance in transit, which means that any peg-out of an amount lower than 10% of the pegged balance will be processed immediately, while higher amounts may need to wait up to 54 additional hours. 
In 54 hours of round-trip time, we have a maximum of 18 slots to perform peg-outs. Assuming an average of 1.5 inputs are consumed per peg-out transaction, and one transaction per batched peg-out, we consume 27 UTXOs. Those 27 UTXOs must not represent more than 60% of the all UTXOs beeing rotated, so the minimum number of rotating peg-outs must be 45. 

### Hysteresis

The correction procedures kick-in when the number of UTXOs is far below or far above the desired number. This reduces the risks of oscillations that will increase the transaction fees consumed on average. On the other side, sorting 60 elements is only 4 times slower than sorting 30 elements, so there is no risks of a much higher CPU consumption. 
The correction method based on consolidation on peg-ins will only be trigger if there is a stream of peg-ins but no peg-outs. Because in this case consolidation requires creating a new peg-out transaction, the process consumes 4 inputs to amortize the cost in fees. Since the other inputs consumed are chosen to be low valued, we don't expect peg-outs to be blocked after a stream of peg-ins.

### Fee Estimation

We've not solved the problem of how users can estimate the peg-out transaction costs before the peg-out transaction is built. We assume that another RSKIP should describe how the Bridge can compute such an estimation for wallets to show in their UIs. It's also possible to provide a new version of the releaseBTC() method that supports an argument to indicate the maximum amount of fees that the user is willing to pay.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Implementation

Moving additional value apart from the user selected peg-out amounts aways carry a certain risk that the change amounts cannot be quickly pegged-in due to the lack of transaction inclusion. This can happen during suddent mempool congestions. 
To reduce this risks, the coin selection algorithm tries to minimize change, yet it behaves mostly greedy. If UTXO amounts are uniformly distributed, it should not consume more than the double of the peg-out amount. 
The proposed coin selection algorithm does not try to find the optimum solution. Finding the best combination of UTXOs is probably as hard as solving the [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem).

## Security Considerations

TBD


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
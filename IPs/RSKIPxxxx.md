# User-triggered peg-out tx fee-bumping


|RSKIP          | xxx |
| :------------ |:-------------|
|**Title**      |User-triggered peg-out tx fee-bumping|
|**Created**    |MAY-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     | |


# **Abstract**

This RSKIP proposes the addition of a several methods to the bridge contract to force previously signed transaction to be rebuilt with higher Bitcoin transaction fees, and be re-signed by pegnatories.


## Motivation

Currently the bitcoin fees (satoshis x byte) used to create Bitcoin peg-out transactions are established by a threshold of 2 out of 3 keys. The use of these keys requires high security procedures. Bitcoin fees can vary rapidly over hours,  and we can't expect the key holders to sign and send messages to the bridge to update fees several times a day. We would like, if possible, to remove the need to vote. If this is not possible, then to increase the number of voters, to, for example, all pegnatories. However, the first idea is technically challenging, and the second, requires frequent fee update messages from all pegnatories. 

This proposals does not tackle the core problem, but instead proposes a mechanism for users to bump the fees of a peg-out transaction that previously signed, in case it has stuck in the memory pool by fast rising transaction fee prices. To achieve this goal, several methods are added to the bridge. In two of these methods, the user must transfer RBTC to the bridge to be used as additional Bitcoin transaction fees. 




## Specification

A collection of the last 128 transaction ids of previously released transactions is stored in a new collection *releasedTransactionIds* in the Bridge contract storage. When a new hash needs to be added, the oldest hash is removed.

Three methods are added to the bridge contract. These are:

```Solidity
function feeBump(byte[] tx) public payable; 
function feeBumpWithInput(byte[] tx,byte[] newInput) public payable;
function feeBumpWithCPFP(byte[] tx,uint pot) public payable;
```

Compute *txHash* as the standard Bitcoin hash of *tx*.

The bridge will check:

* if the transaction identified by *txHash* is not present in *releasedTransactionIds* , REVERT
* if the transaction identified by *txHash* is present in , but also present in BTC_TX_HASHES_ALREADY_PROCESSED_KEY (it was already registered in the blockchain), REVERT.

The *feeToAdd* be the value received in the method call. 

The gas consumed by any of these methods is SIG_BASE_COST+(length(tx)/32)/12. SIG_BASE_COST is set to (*pegnatoryCount*+1)/2\*40k (TBD). The base cost covers the cost of pegnatories to send new signatures. The variable cost coverts the SHA256 hashing of the transaction. For example, if there are 13 pegnatories the base cost is 280k.

"/" is the integer division. 

Let's *T* be the prior transaction. Let *T.fees* be the prior transaction fees. Let *T.output* be the amounts paid to users (peg outs). Let *T.change* be the amount returned to the Bridge contract as change. Let *T.input* the amount collected by *T* inputs. 

The input selector algorithm is modified so that a reasonable attempt is made to choose inputs that can cover potentially at least four times the computed transaction fees. 

These methods create new release transactions, but the Bridge does not store store them in its storage. The new transaction hashes are not stored in *releasedTransactionIds*.

### Method feeBump()

* If *feeToAdd* >= *T.change*, then then a new transaction *T'* is built with the same inputs, the same outputs, but with *T'.change = T.change-feeToAdd*. This moves fees from the change output to the transaction fee. If *T'.change* is low the dust limit, the output is removed.
* Otherwise, REVERT

Note that because of the change in the input selector algorithm, the case that  *feeToAdd* >= *T.change* will be the most common. This method allows to bump a transaction in most cases, but can only provide high assurance up to a 2X bump of the fees.

### Method feeBumpWithInput()

A new transaction *T'* is built having the inputs and outputs of *T*, but with an additional input appended, specified by newInput. The user is responsible to check that the input with amount is enough to bump the fees. This is not checked by the Bridge. The input given must already be signed.

The full amount of the input will be consumed to pay fees.

 The user is responsible to check that the input with amount is enough to bump the fees. This is not checked by the Bridge.

### Method feeBumpWithCPFP()

This method performs all checks and transaction changes similar to feeBump(). Then the new transaction *T'* is appended an additional output containing a transfer of *pot*  bitcoins paying to the address 1CPFPxxxSv1jWf6wCyihRWhKyYFkKX3bnf. The private key for this address is 5KFhJj5HeGh81EuzuRWo6SigTN5cQnFSTmeSHsWuwG7AN5pze9C. Since the inputs have not changed, the fee per byte of *T'* will be lower than the fee per byte of *T* unless the caller also adds fees by transferring bitcoins in the call. Because *T'* must replace *T* in the memory pool, an additional fee will be required. The may need use a Child-Pay-For-Parent later, consuming the CPFPxxx output to push this transaction into a block. This method allows to confirm a transaction if the user suspect there is a risk that the transaction fees increase an order of magnitude in a short time.

## Rationale

The following topics has been considered in this proposal.

### More protections against DoS

The high gas cost of these methods is to prevent users abusing of the powpeg resources to sign transaction. To prevent abuse even more Bridge could allow only fee bumps that add at least 30% more fees that the previous fees used. Therefore each attempt becomes more and more expensive. This can be done by storing the last fee used in *releasedTransactionIds*.

### Sending txHash instead of tx

Instead of passing the transaction as an argument, transactions that have been signed by the majority of powpeg members may be stored for 7 days in a new dictionary *signedTransactions* indexed by transaction hash. However, is seems that the complexity and storage cost of handling this additional collection is unnecessary and the used performing the fee bumping can re-send the transaction.

### Mixing Dirty UTXOs

The feeBumpWithInput() could be used to mix "dirty" UTXOs (for example, considered to come from illicit activity) into peg-out transactions. However, this feature cannot be reliably used to launder bitcoins, as the input added is fully paid to miners, and its value is supposed to be vey low. However, the Powpeg doesn't have (and since peg-ins are decentralized, it cannot have) any mean of avoiding the peg-ins with of illicit funds. Therefore feeBumpWithInput() cannot have a negative effect on the platform.

### Input Selection Algorithm

The input selection algorithm must do a reasonable effort to select inputs that allow 4X fee bumping. By reasonable we mean that the algorithm should still be linear, or the number of powpeg UTXOs, bounded. The algorithm should not add more inputs in order to satisfy this condition, as the cost in virtual bytes of a non-segwit input (~137 bytes) is higher than the cost of CPFP output (30 bytes). Adding more inputs only makes sense if they are all low valued, and they are using the peg-out transaction for input consolidation.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Test Cases

TBD

## Security Considerations

* The method used to create the new transaction forces it to be always a double-spend of the prior one, therefore this method cannot be used to perform two peg-outs at the price of one.

* This proposal avoids merging new Bridge inputs when creating a fee bumping transaction. The reason is that the bridge does not know which one of the transactions (*T* or *T'*) will be finally included in the Bitcoin blockchain, and therefore all inputs must be marked as consumed. To unmark the unused inputs the Bridge would need to receive notification of the transaction *T* or *T'* when it is included in the Bitcoin blockchain, and this functionality is not part of the Bridge. But more importantly, an attacker may be able to force the bridge to create hundreds of variations  of the fee bumping transaction, locking all inputs and rendering the Bridge useless for many blocks.
* FeeBumpWithInput cannot consume the input partially for fees because the Bridge doesn't know the input amount of the input given. To be sure of the input amount, the transaction would need to be segwit, because in segwit transactions the input amount is signed.
* Is could possible for an attacker to block a a honest user from consuming a CPFPxxx output in a peg-out transaction by broadcasting a transaction that also consumes the CPFPxxx output, but for other reasons it is not mined. The rules that govern CPFP in the Bitcoin node must be carefully analyzed to disregard this attack.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

---
rskip: 241
title: User-triggered peg-out tx fee-bumping
description: 
status: Draft
purpose: Usa, Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-05
---
# User-triggered peg-out tx fee-bumping


|RSKIP          | 241 |
| :------------ |:-------------|
|**Title**      |User-triggered peg-out tx fee-bumping|
|**Created**    |MAY-2021 |
|**Author**     |SDL |
|**Purpose**    |Usa, Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     | |


# **Abstract**

This RSKIP proposes the addition of a several methods to the bridge contract to force previously signed transaction to be rebuilt with higher Bitcoin transaction fees, and be re-signed by pegnatories.


## Motivation

Currently the bitcoin fees (satoshis x byte) used to create Bitcoin peg-out transactions are established by a threshold of 2 out of 3 keys. The use of these keys requires high security procedures. Bitcoin fees can vary rapidly over hours,  and we can't expect the key holders to sign and send messages to the bridge to update fees several times a day. We would like, if possible, to remove the need to vote. If this is not possible, then to increase the number of voters, to, for example, all pegnatories. However, the first idea is technically challenging, and the second, requires frequent fee update messages from all pegnatories. 

This proposals does not tackle the core problem, but instead proposes a mechanism for users to bump the fees of a previously signed peg-out transaction that is stuck in the memory pool by the fast rising transaction fee prices. To achieve this goal, several methods are added to the bridge. In two of these methods, the user must transfer RBTC to the bridge to be used as additional Bitcoin transaction fees. 




## Specification

A collection of the last N transaction ids of previously released transactions is stored in a new collection *releasedTransactionIds* in the Bridge contract storage. The list is managed as a FIFO queue. When a new hash needs to be added, the oldest hash is removed. This proposal assumes peg-out transactions will be batched and will not be issued faster than one every two hours. However, this proposal does not specify how batching is performed. With this rate-limiting assumption, we set a conservative value of N=128.

Three methods are added to the bridge contract. These are:

```Solidity
function feeBump(byte[] tx) public payable; 
function feeBumpWithInput(byte[] tx,byte[] newInput) public payable;
function feeBumpWithCPFP(byte[] tx,uint pot) public payable;
```

The value  *txHash* is computed as the standard Bitcoin hash of a transaction *tx*.

The bridge contract performs one check for all methods: If the transaction identified by *txHash* is not present in *releasedTransactionIds*, REVERT.

The gas consumed by any of these methods is SIG_BASE_COST+(length(tx)/32)/12. SIG_BASE_COST is set to (*pegnatoryCount*+1)/2\*40k (TBD). The base cost covers the cost of pegnatories to send new signatures. The variable cost coverts the SHA256 hashing of the transaction. For example, if there are 13 pegnatories the base cost is 280k.

"/" is the integer division. 

Let *T* be the prior transaction. Let *T.fees* be the prior transaction fees. Let *T.output* be the amounts paid to users (peg outs). Let *T.change* be the amount returned to the Bridge contract as change. Let *T.input* the amount collected by *T* inputs. 

These methods create new peg-out transactions and will require pegnatories to sign them as usual. However, the new transaction hashes are not stored in *releasedTransactionIds*. The fee bumping methods will still identify transactions to fee-bump with their original ids.

### Changes in Creation of Peg-outs

When creating a new peg-out transaction, a reasonable attempt will be made to choose inputs that can cover potentially at least 10 times the computed transaction fees. This will allow a 10x fee increase without adding inputs. This change is not fully specified in this proposal, and requires a change in the the transaction input selector algorithm.

### Method feeBump()

Let *callValue* the RBTC value received in the method call .

The *feeToAdd* is defined to be equal to *callValue*. 

* If *feeToAdd* <= *T.change*, then then a new transaction *T'* is built with the same inputs, the same outputs, but with *T'.change = T.change-feeToAdd*. This moves fees from the change output to the transaction fee. If *T'.change* is below the dust limit, the output is removed.
* Otherwise, REVERT

Note that because of the change in the input selector algorithm, the case that  *feeToAdd* <= *T.change* will be the most common. This method allows to bump a transaction in most cases, but not all.

### Method feeBumpWithInput()

If the input size is higher than 1000 bytes, then the call REVERTs.

A new transaction *T'* is built having the inputs and outputs of *T*, but with an additional input appended, specified by newInput. The input given must already be signed.

The bridge will validate that the input conforms to its expected format (the correct parsing of each field and that there is not additional data past the last field). However, the bridge will not inspect the content of each field, except for txin-script length, which must be inspected for parsing the script field.

The full amount of the added input will be consumed to pay fees.

 The user is responsible to check that the input with amount is enough to bump the fees. This is not checked by the Bridge.

### Method feeBumpWithCPFP()

If *pot* > *callValue*, and REVERT. The call value is split between a pot for CPFP and an additional fee for the miner.

The *feeToAdd* is defined as *callValue* - *pot*. 

If *feeToAdd* > 0 then the following check is performed (similar to method feeBump(), but with a different fee value):

- If *feeToAdd* <= *T.change*, then then a new transaction *T'* is built with the same inputs, the same outputs, but with *T'.change = T.change-feeToAdd*. This moves fees from the change output to the transaction fee. If *T'.change* is below the dust limit, the output is removed.
- Otherwise, REVERT

Then the new transaction *T'* is built as T with an additional additional appended output containing a transfer of *pot*  bitcoins paying to the address 1CPFPxxxSv1jWf6wCyihRWhKyYFkKX3bnf. The private key for this address is 5KFhJj5HeGh81EuzuRWo6SigTN5cQnFSTmeSHsWuwG7AN5pze9C. If *feeToAdd* == 0, the fee per byte of *T'* will be lower than the fee per byte of *T*. 

## Storage

The collection *releasedTransactionIds*  is stored in the Bridge contract storage serialized in a single cell with key `DataWord.fromString("releasedTransactionIds")` (10 zero bytes followed by the ASCII string "releasedTransactionIds"). The maximum space consumed by this cell is N*32. 

## Rationale

The following topics have been considered in this proposal.

### More protections against DoS

The high gas cost of these methods is to prevent users abusing of the powpeg resources to sign transactions. To prevent abuse even more, the Bridge could allow only fee bumps adding at least 30% more fees compared to the previous fees used. Therefore, each attempt to create a new peg-out transaction becomes exponentially more expensive. This can be done by storing the last fee used in *releasedTransactionIds*.

### Sending txHash instead of tx

An alternative design is to pass as argument the transaction hash instead of the transaction itself, reducing the call data cost. The bridge would need to store every signed transactions for M days in a new dictionary (i.e. *signedTransactions*) indexed by transaction hash. However, it seems that this design is more complex and it involves a higher state storage cost related to handling this additional collection. We believe this is unnecessary as fee-bumping would be infrequent, and the user performing the fee bumping can re-send the transaction.

### Mixing Dirty UTXOs

The feeBumpWithInput() could be used to mix "dirty" UTXOs (for example, considered to come from illicit activity) into peg-out transactions. However, this feature cannot be reliably used to launder bitcoins, as the input added is fully paid to miners, and its value is supposed to be vey low. However, the Powpeg doesn't have (and since peg-ins are decentralized, it cannot have) any means of avoiding the peg-ins with of illicit funds. Therefore feeBumpWithInput() cannot have a negative effect on the platform.

### Input Selection Algorithm

The input selection algorithm must do a reasonable effort to select inputs that allow 10x fee bumping. By reasonable we mean that the algorithm should still be linear, or the number of powpeg UTXOs, bounded. The algorithm should not add more inputs in order to satisfy this condition, as the cost in virtual bytes of a non-segwit input (~137 bytes) is higher than the cost of CPFP output (30 bytes). Adding more inputs only makes sense if they are all low valued, and they are using the peg-out transaction for input consolidation.

### Parameter Choices

This proposal is intended to be adopted concurrently with a proposal to perform peg-out batching at a maximum rate of one peg-out every 2 hours. By setting N=128, we enable a period of 10 days for transactions to be fee-bumped. 

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Test Cases

TBD

## Security Considerations

* The method used to create the new transaction forces it to be always a double-spend of the prior one, therefore this method cannot be used to perform two peg-outs at the price of one.

* This proposal avoids merging new Bridge inputs when creating a fee bumping transaction. The reason is that the bridge does not know which one of the transactions (*T* or *T'*) will be finally included in the Bitcoin blockchain, and therefore all inputs must be marked as consumed. To unmark the unused inputs the Bridge would need to receive notification of the transaction *T* or *T'* when it is included in the Bitcoin blockchain, and this functionality is not part of the Bridge. But more importantly, an attacker may be able to force the bridge to create hundreds of variations  of the fee bumping transaction, locking all inputs and rendering the Bridge useless for many blocks.
* FeeBumpWithInput cannot consume the input partially for fees because the Bridge doesn't know the input amount of the input given. To be sure of the input amount, the transaction would need to be segwit, because in segwit transactions the input amount is signed.
* It could possible for an attacker to block a honest user from consuming a CPFPxxx output in a peg-out transaction by broadcasting a transaction that also consumes the CPFPxxx output, but contains other features or orphan inputs that makes it non-mineable. The rules that govern CPFP in the Bitcoin node must be carefully analyzed to disregard this potential attack.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

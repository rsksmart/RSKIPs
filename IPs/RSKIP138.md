---
rskip: 138
title: Multi-signed transactions supporting enveloping and multi-key accounts
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2019
---
# Multi-signed transactions supporting enveloping and multi-key accounts

|RSKIP          | 138 |
| :------------ |:-------------|
|**Title**      |Multi-signed transactions supporting enveloping and multi-key accounts|
|**Created**    |2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


# **Abstract**

One of the features that can help the blockchain technology reach mass adoption is transaction enveloping. Enveloping allows a third party to pay for the gas of a transaction, while the transaction still originates from the same account that does not possess RBTC. Implementing this feature efficiently in terms of gas requires modifications to the layer 1. Basically we can build a transaction we more that one signer, and specify which of the signers will pay for the fees. Multi-signed accounts also open the possibility to compress with LTCP settlement transactions sent to payment channels. Instead of n transactions (one for each of the participant), a single transactions with n signatures can be built, reducing the network resource consumption. 



# **Specification**

This RSKIP describes a new transaction format to support multi-signed and multi-key transactions. 

Currently the transaction contains the following fields, stored using RLP:

1. nonce
2. gasPrice
3. gasLimit
4. eeceiveAddress
5. value
6. data
7. v
8. r
9. s

Each field begins with an RLP identifier that indicates if the item is an element or a list of elements.

### Transaction Format 1

We propose a new format (1), which both add new fields and change the meaning of existing fields:

1. multiPurpose = (version, ownerCount, nonceList)
2. gasPrice
3. gasLimit
4. receiveAddress
5. value
6. data
7. signatureList
8. empty
9. empty
  

The objetive to use this format is to keep it compatible with most existent tools, and hardware wallets in particular. Hardware wallets will show the destination, fees (derived from gasPrice and gasCount) and possibly a hash of the data, but not the nonce, so we can use the nonce to pack additional fields.  

Version is encoded as a list of a single element. For the new transaction format, the element is 1. Therefore the parser can detect if it is a transaction version 0 or a new kind by inspecting only the first element. If it's a list, it's the new type. If not, then it is the nonce of a transaction version 0.

If version is 1, then Nonce is a list of elements. 

The ownerCount field specifies how many signatures are needed to build a multi-key account. If ownerCount is 1, then it's a single key account. This has the same intent as RSKIP39, but here we propose a more restricted version that does not support multi-key smart-contracts. 

To derive the account address of a multi-key account, the first "ownerCount" public keys are concatenated and the first 20 bytes of the keccack hash of those public keys is the account address.  Multi-key accounts, when combined with enveloping, can help a group of users manage a payment channel without using RBTC, and purely using ERC-20 tokens, relying on an external network transaction relayer.

The field signatureList is a list of signatures. If there are more than one signature, this is a multi-signed transaction.  The format for each element of the signatures list is an RLP list containing v, r and s for each signer. Each element in Signatures must have exactly 3 elements. 

**Multiple Nonce Queues**

The field nonceList is a list of nonces. The number of element in nonceList can be equal to or one unit higher than the number specified in ownerCount. If the number of nonces is equal, then there is no enveloping. If the number of nonces is one higher, then the last party to sign is the one paying the fees. We adopt a similar approach to CallWithSigner proposal and we'll enable a multiple queues for nonces. This let a user dispatch several meta-transactions in parallel. 

A nonce can be a single element, which means that the standard queue is used, or it can be a pair (queueID, nonce) which opens a new queue. Queues are stored in a mapping (queueID --> nonce) in a specific tree that hangs from the Account node on the Unitrie.



**MemPool management with Multiple Nonce Queues**

When a large number of transactions from the same sender are issued in parallel using different queues, different nodes may store a different subset of transactions in their memory pool, depending on the order of arrival. It's important that all nodes share a policy of transaction replacement and eviction to avoid deadlocks. For example, let's assume that a node stores in the mempool up to 15 transactions originated from the same account. Alice sends 32 transactions, 16 from the main queue, and the other 16 in separate queues. A  peer A may store the 16 transactions for the main queue, while a peer B may store the 16 transactions using the other queues. Since both have reached the transaction limit, non of them will accept more transactions from Alice, until one of them is confirmed or one of them is replaced. Now Alice wants to speed up the inclusion of a transaction T in the last queue, so she tries to bump its fees, and a prepares a new version T'. She sends it to peers A and B. Since peer B is not storing T, it discards T' too. If peer A si surrounded by peers having a mempool similar to B, then the transaction T' won't propagate over that path also. If node B is surrounded by nodes having a mempool similar to node A, then Alice won't be able to replace any of her transactions.

Therefore nodes should set a fee bumping policy where a new transaction T' can replace any other transaction Q that is a tail in a any queue provided that the gasPrice in T is higher than the gasPrice in Q. If the transaction limit has been reached, then Q is evicted and T stays. This ensures that the overall fees that Alice is willing to pay always increases. 

### Signing the Transaction

Signers do not sign exactly the same payload. Each signer signs the RLP of all the fields with changes in the multiPurpose, gasPrice and singatureList fields. If there is an enveloper, then the gasPrice field is signed only by the enveloper, and not by the rest. This allows for the enveloper to bump the transaction fees to increase the chances it is included in a block. If there is no enveloper, than all signatories sign the gasPrice field as normal. The signatureList field is replaced by the chainId, to maintain compatibility when signing. The signerIndex is added as the last element of the multiPurpose field.

The following list shows how elements are packed for signing (changes in bold):

1. multiPurpose = (version, ownerCount, nonceList, **signerIndex**)
2. gasPrice or empty
3. gasLimit
4. receiveAddress
5. value
6. data
7. **chainId**
8. empty
9. empty

The signerIndex is the index of this signer in the list of all signers, starting from zero. Note that all account owner signatures must be listed in a pre-defined order. The transaction will only be valid if each signature signs its correct position. This is to prevent that an attack where owner and enveloper signatures are exchanged by an attacker.

### Transaction Execution

When the transaction is executed, all nonces must be checked. If a new nonce queue  is used, then the first valid nonce is zero. Any invalid nonce invalidates the transaction. All nonces associated with the transaction must be incremented. If the transaction specifies an enveloper (by containing one nonce more than the onwerCount), then the transaction origin (msg.origin) will still point to the account address, as defined by the other signers public keys. However fees will be deducted from the last signer account (the sponsor). Also the gas reimbursement will be performed to the sponsor account.

It would be beneficial to have an opcode or precompile contract to obtain the address of the sponsor.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

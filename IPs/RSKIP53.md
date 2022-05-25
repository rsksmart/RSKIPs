---
rskip: 53
title: LTCP 
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-02-
---

# LTCP

|RSKIP          |53           |
| :------------ |:-------------|
|**Title**      |LTCP |
|**Created**    |FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes a novel scaling proposal based on two concepts:

- reducing the size of transactions by delta compression
- removing signatures from the blockchain using signatures for chains of transaction 

This proposal, originally named LTCP, was motivated to reduce the cost of use of payment channels, and other multi-party protocols. This RSKIP is an adaptation of the original LTCP proposal with some key changes:  delta compression is not performed by using a previous transactions as template, but by using a user preset. The prior design was proven difficult for state-synched node to reconstruct  a transaction.  

The LTCP proposal also set the basis for a whole new scaling technology named Shrinking-chain Scaling, which is based on the reduction of the blockchain after blocks have been mined.

# **Motivation**

The shorter the transactions, the more scalable a blockchain is. The less CPU load required to process a transaction, the more scalable a blockchain is. 

### Privacy Considerations

It must be noted that compression means there is an identifiable pattern repetition and signature linking means there is a common owner. Both properties generally work against privacy. However, RSK plan, described in its foundational whitepaper, has been to achieve the cheapest layer for transaction verification, and on top of it build a robust layer for transaction privacy, which may arise from on-chain (zCash-like) or off-chain (BOLT-like) payments. The aim is that all users (not only the wealthy) will be able to protect their private transactions using the privacy layer, while being able to achieve the cheapest non-private transactions. On-chain transaction compression in fact reduces the cost of opening and settling payment channels, which in turn enables off-chain private payments at a lower cost.

# **Specification**

The LTCP protocol provides two benefits:

- delta compression of selected fields, taken from a previous preset.
- signature aggregation of previous transactions, so previous signatures can be disposed. 

Delta compression is done by allowing each transaction to refer to a preset created by the same sender, which is used as a template. Any field can be overridden, and unmodified fields are copied intact. 

A transaction that uses LTCP contains several fields, some of them optional and some of them are persistent. Persistent means that they will become part of the blockchain forever, while non-persistent fields may or may not become part, depending on future transactions. These are the user provided transaction fields: 

- [0] **nonce**: a nonce [persistent if repeated]
- [1] **seqNum**: a sequence number [optional,persistent]. For future use with multi-key accounts.
- [2] **amount**: amount of funds to transfer [optional,persistent]. If not specified the amount is zero.
- [3] **receiver**: the address of the receiver [optional,persistent]
- [4] **gasPrice** (or fee): an amount of fee to pay [optional, persistent]
- [5] **gasLimit**: max number of steps to execute [optional, persistent]. If omitted, the default value is 30000.
- [6] **readPreset**: this value specifies the preset that must be used as template for delta compression  [optional,persistent]. If missing, is specifies the previous nonce.  [optional, persistent]
- [7] **data**: arbitrary user data to be sent to the receiver (used mainly for smart contracts) [optional, persistent]
- [8] **compressedAmount**: amount of funds to transfer in float format [optional]
- [9] **signatures**: an RPL list of signature fields [RESERVED for multi-key accounts]. [not persistent]
- [10] **signature**: represent a single ECDSA signature, as a list of fields (r,s,v, linkSigRecHash). [not persistent]
- [11] **senderAccountPrefix**: specifies the sender account. Used internally, cannot be specified by the user [persistent]  
- [13] **writePreset**:  this value specifies the preset that must be overwritten with all transaction data fields reconstructed in this SigRec. [optional, persistent]
- [14] **deletePreset**: this is used to erase a preset. It can be combined with readPreset or writePreset, but the same preset cannot be written and deleted. [optional, persistent]
- [15] **properties**: holds a bitmap of additional flags (in the 3 upper bits, as explained later). Currently a single bit is specified (bit 5). Bit 5 corresponds to the **linkBit** field. This field specifies if the transaction is linked to the previous one or not. The default value of this bit is 1 (linked). [optional,persistent]
- [16] **receiverPrefix**: a prefix of the address of the receiver [optional,persistent]

The set of persistent fields in a transaction is called the Persistent Transaction Information (PTI). The senderAccounPrefix indicates how to locate the sender in the state trie. It's required once the signature is removed. The PTI of a transaction is what is actually stored in the blockchain.

Either the "signature" field or the "signatures" fields can be specified (but not both). The signature field is used when there is a single signatory, while the signatures field is used when there are multiple signatories. This achieves a slight reduction in transaction size.

The compressedAmount is a big integer represented by a base-10 exponent (lowest 5 bits) and a mantissa (most significative variable number of bits), allowing greater compression of the amount.  The transaction cannot specify both compressedAmount and amount. 

Each signature contains the standard 3 fields (r,s,v).  The "linkBit" in the properties field of the transaction indicates if this signatures is linking to the previous one, or not. 

Transactions are serialized in a new format. Each field is identified by a single byte id, which is the value specified between brackets in the previous list. 

### Transaction Wire Format

The transaction starts with a version byte 0x01, which means the transaction is formatted according to this RSKIP, and not the standard formatting. Standard transactions always start with a byte in the range [0xC0, 0xFF]. The fields are serialized in RLP format after this version byte. The values, interpreted as byte arrays. The most significant byte of each value is the idData. The lower 5 bits of IdData are used to specified the fieldId. The upper 3 bits are used to store additional flags whose definition depends on the fieldId. After the idData is removed, the remaining bytes specify the value (as in the old format). For example, the following is a valid transaction calling a method in a contract.

0x01 | RLPList( RLP(0x030102030405060708090A0102030405060708090A), RLP(0x0701020304))

The first argument represents the field receiver (0x03). The second argument represents the "data" field (0x03).

### Signing

What is actually signed in a transaction is not the PTI, nor the user transaction, but a new compound record with additional information, called the **sigRec**. If linkTo is true, then the sigRec consist of two sub-parts, the fullRec and the hash of the previous SigRec (called **linkSigRecHash**). The fullRec contains all the fields in a fixed order. Empty fields have ids but no appended data.  Fields used only for compression are not signed. Currently this covers only the compressedAmount  field. If fullRec will always contain the amount specified as the amount field, not the compressedAmount field. The compressedAmount only appears in the wire format of the transaction, and in the PTI. 

To sign a transaction, its corresponding sigRec is signed (which in turns involves hashing the message). The sigRec is signed with the ECDSA private key corresponding to the signer. 

The following example shows a sequence of 2 transactions. The first, created by Alice, is not linked to any prior one. The second originates in the same Alice account and is linked to her prior one. The transactions are described as (field,value) pairs (not in its wire format). 

- **T0-wire** = (amount: 10, receiver: 0xFF, gasPrice: 10, writePreset: 0, properties: 0, signature:  s0=(r0,s0,v0) ). v0 doesn't have the linkTo bit set.
- **T0-PTI**  = (amount: 10, receiver: 0xFF, gasPrice: 10, writePreset: 0,properties: 0 )
- **T0-fullRec** = (nonce:0, amount:10, receiver: 0xFF, gasPrice: 10, gasLimit: 30000, data: [], senderAccountndex: 99, writePreset: 0, properties: 0) 
- **T0-sigRec** = T0-fullRec
- **(r0,s0,v0)** = Sign( privKey, T0-sigRec )
- **T1-wire** = (nonce: 1, amount: 100, readPreset: 0, signature: s (r1,s1,v1), properties: 32 ). Properties has the linkBit set.
- **T1-PTI** = ( amount: 100, readPreset: 0)
- **T1-fullRec** = (nonce:0, amount:100, receiver: 0xFF, gasPrice: 10, gasLimit: 30000, data: [], senderAccountndex: 99, readPreset: 0, properties: 32)
- **T1-sigRec** = ( Hash(T1-fullRec) | Hash(T0-sigRec ) ]
- **(r1,s1,v1)**= Sign( privKeyAlice,T1-SigRec ). 

To locate the last sigRec, the hash of the last sigRec is stored along the the signatory account, in a new field **lastSigRecHash**. This adds 32 bytes of persistent storage but reduces the need to additional caches. Therefore, each time a transaction is executed, the sender's nonce will increase and the sender's lastSigRecHash will be updated.  This also enables the blockchain processor, when processing a block at height H,  to process in advance and backwards blocks in future blocks, marking transactions as signed, until it reaches block H, and finds out if all transactions have or not been signed. If a transaction is built for a different linkSigRecHash, full nodes won't be able to verify the transaction.  

A transaction with nonce zero cannot have the linkBit set.

### Storage of lastSigRecHash

The lastSigRecHash of each account is stored in a node in the account trie, under the key 0x02 (similar to code and storage in RSKIP16). It comprises a 32-byte data field. If a transaction is included for an account at a certain block, then the lastSigRecHash will be modified, reflecting the signature included in that block.

### Gas Cost

LTCP transaction format adds a cost per byte not only in the "data" part, but in the transaction itself. Each byte, either part of the RLP encoding, version byte, field id or remaining payload, costs 80 gas for non-zero bytes and 4 gas for zero bytes. 

The base gas cost of a version 1 transaction is 8K gas, instead of 21K.

Additionally, the value of 10000 is added if it's a version 1 transaction and:

1. nonce is zero OR
2. linkBit is zero

This additional amount covers the initial cost of a single signature storage and the lastSigRecHash storage.  

For example, here is a break-down cost of a transaction: 

0x01 | RLPList(  RLP(0x2F) | RLP(0x030102030405060708090A0102030405060708090A), RLP(0x0201))

* Base cost: 8000
* Wire bytes cost:  29 * 68 = 1972

Total cost: 9972

### Rationale for Signing All Fields

All values are signed (not only the "deltas") to enable hardware wallets to known exactly what they are signing.

The delta compression can be provided by the wallet within an insecure computer. This could have a drawback: there could be transaction malleability attacks where peers expand the transaction with additional values. While this is true, normal peer will automatically compress the transaction again back to its minimal size because they need to access the specified preset. 

### Preset Set and Recall

Every transaction can overwrite a preset or use a preset. To overwrite a preset, the field "writePreset" must be used. The argument of this field is the preset index. To use a preset, the "readPreset" field must be used. To clear a preset, the field "clearPreset" is used. 

More thought has to be dedicated to understanding how presets interact with storage rent, but it doesn't seem to be problematic.  

### Preset Presistence

Full nodes need to store each preset. Presets are stored in the blockchain state. This is done by creating a subtree down the account node, with the fieldSelector equal to 0x01, according to RSKIP16. Each preset will be stored in its RLP format, with ordered fields, in a separate node of the trie.  Therefore the key will be the preset number (without leading zeros, except the 0x00 preset, which is ecoded as 0x00), and the value the RLP of the fields. 

### Transaction Propagation Policy

A preset can be updated or cleared by a transaction. If following transactions from the same source are allowed to be propagated before the parent is included in a block, then nodes must be forced to maintain dynamic preset states. To avoid this, if a transaction with nonce N that writes or clears a preset is received by a node, then the node will not process or forward a transaction with greater nonce, before this transaction is included in a block.  

### Transaction Acceptance Criteria

 A transaction is accepted if it satisfies standard conditions (enough funds, well formatted, etc.) plus additional conditions on the sequence number, which allows transaction replacement. Transaction replacement is a feature that is necessary for multi-key contracts used in payment networks. In those cases, the involved parties need to be able to create transactions that replace each other, but where an old transaction cannot block the newer one.  The involved parties will keep the nonce equal, but increase the sequence number. 

The transaction fields that are missing from the user provided fields are copied with the fields of the preset transaction. The additional conditions to accept a transaction are the following:

- the nonce must be one unit higher or equal than the nonce of the previous transaction from this account (previously it could only be the following nonce).

- If the nonce is equal, the seqNum value must be higher than the predecessor (any number of units). 
- If the nonce is equal, it is included in the PTI to signal it. 
- the source account must have enough funds for the payment.
- If the linkBit is set and the nonce is non-zero, then the senderAccountIndex must be present, and it must match the sender recovered by the standard ECDSA public key recovery method.

### Peer Scoring

If a peer receives a transaction whose signature cannot be verified (ECDSA signature public key recovery generates an unknown address), then the node can score negatively the peer that forwarded the transaction. Therefore nodes must be careful not to forward a transaction that depends on a preset that is too recent. We recommend a 100 block gap to avoid this situation. As an example, because of the asynchronous nature of the communication link, if the preset is recent it may affect the template which the transaction is based on, between the time it is announced and the time it is forwarded. 

### Mining a Transaction Version 1

To include a transaction version 1in a block, if the nonce is non-zero and the linkBit is set, then miner must add the senderAccountIndex value to the PTI, to make it valid.  

### Block Format

The LTCP proposal requires a modification to the block format. Currently, the wire-block format is as follows:

- wire-block = (header, tx_list, uncle_list ,transaction_list, [, signature_list] ).

For LTCP the block header semantic must change. In particular the Merkle trie of the transaction list is modified. The new trie can contain both old-style (version 0) transactions and the transaction PTIs of transactions version 1.  Therefore the block PoW authenticates the actual compressed information contained, not the "meaning" for transactions version 1. 

Signatures are not directly referenced in the block, but the block is invalid if the sender does not provide a set of valid signatures, or the future blocks do not provide valid linked signatures. It's clear that if signatures are removed, then the block must still be able to be validated, so the transaction id must not reference them. 

The Uncle_list fields corresponds a the previous definition of block. The transaction_list contains can contain both version 0 transactions and version 1 PTIs. PTIs are always prefixed by the version byte 0x01.

The signature_list is a list of tuples in the wireSig format, as specified:

- wire-Sig = ( txIndex, (r,s,v)) .

txIndex is the index in the tx_list which this signature is associated with. Indexes should be specified in ascending order. The signature_list may be empty if all signatures are removed. 

### Blockchain Synchronization

To fully validate a block, many signatures for the involved accounts may need to be collected, both in this wire-block or in following wire-blocks, until all transactions in the block can be recovered.  In practice the blockchain synchronization protocol needs to be modified. Although this RSKIP does not specify the final protocol, we show an example protocol as a starting point. Four new wire-protocol commands are added:

- **getSignatures**(addressList,blockHash ) : requests the signatures of the list of provided addresses at block with the specified hash. 
- **getSignaturePackage**(startingAddress, sigCount,blockHash ): requests a list of signatures, starting from a specified address, and continuing with addresses in the trie lexicographical order (hash prefix included).  The peer should return a sendSignaturePackage message
- **sendSignaturePackage**(startingAddress, signature_list,blockHash ): sends a list of signatures, starting from a specified startingAddress
- **sendSignature**(address, signature,blockHash ): sends a signature for a specified address

In the following protocol, Alice wants to synchronize the blockchain from zero.

1. Alice discovers the best chain tip.
2. Alice requests all headers from genesis to the chain tip from one or more peers.
3. All block headers are collected and validated while being received
4. Every block where (blockHeight % 10000==0) is defined as a checkpoint. 
5. Alice finds the last checkpoint with at least 1000 confirmations. 
6. Alice request the state at the checkpoint from one or more peers.
7. Alice validates with inclusion proofs of the state chunks as received.
8. Alice requests all signatures for that checkpoint using getSignaturePackage() from one or more peers.
9. Alice validates the signatures as received, using the last hash stored in the account trie.
10. Alice downloads blocks from genesis to the blockchain tip, and verify each block as received.

It's important to note that all signatures are verified first, with the exception of accounts that have been deleted. Currently RSK does not support the deletion of accounts. If in the future is supports deletion, then the last signature for each deleted account should be sent in the block where it is deleted. Also the same could happen with other accounts if hibernation is implemented.

If all accounts have valid signatures, the checkpoint block is considered sig-valid. However this does not imply the checkpoint block is valid according to all consensus rules.

During full block download (step 10) it may be the case that a block received is invalid. If the invalid block is prior to the checkpoint or the checkpoint itself, then that block and all child blocks must be discarded: a new checkpoint should be requested prior the faulty block, or from other peer. The peer that informed the invalid checkpoint must be penalized. 

### Lifetime of Signatures

Old transaction signatures can be removed from the blockchain if:

* There exist a new signature that links back to the old one and this signature is included in a block.
* The block containing the new signature has at least two checkpoints past it.

Because a checkpoint is built every 10k blocks, it's possible to prevent the blockchain from advancing if two checkpoints are reverted (at least 20k blocks, or about 7 days). Archive nodes or probabilistic archive nodes can be used to recover this data in case of a such devastating 51% attack. Probabilistic archive nodes are full nodes that will keep one more past checkpoint with probability 1/2, and the previous one with probability 1/4 and so on. 

### Identifying Transactions

To maintain compatibility, the transaction id is redefined for transactions version 1, but not for version 0. For version 1, the id is the hash of the fullRec. Signatures are not included.

To obtain a transaction id of a transaction in certain block with a certain index (I), a node must:

1. Locate the PTI using I as key into the PTI trie
2. Find the preset (if any) in the account trie. Apply the preset to the PTI
3. Expand the PTI into a fullRec. 
4. Hash the fullRec.

### Security Considerations

Let's suppose that a transaction R replacing a pre-existent preset with new fields is included in a block, and a transaction T refering to that preset is included afterward. It's possible that a chain reorganization removes R and therefore makes T refer to the first (deleted) preset instead of new one defined in R. This does not pose a security risk because all fields are signed, including those in the preset. Therefore the signature of T would become invalid if the transaction R is removed.

### Variations 

Also it’s possible to replace the senderAccountPrefix a shorter field, representing the block number delta and transaction index in the block of the previous transaction from the same account in the blockchain, using compact variable-length integers. For instance, if the prior transaction is the second of the previous block, the value (-1,2) would encode the reference, consuming no more than 2 bytes.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

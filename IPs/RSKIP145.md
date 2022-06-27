---
rskip: 145
title: Struct Transaction Format
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-02-
---
# Struct  Transaction Format

|RSKIP          |145           |
| :------------ |:-------------|
|**Title**      |Struct Transaction Format |
|**Created**    |FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP (extracted from the original RSKIP53) proposes a new more flexible transaction format. This new format enables new optional fields to be added in the future. Each field is identified by an identifier instead of by its position in a list. The LTCP protocol (RSKIP53) makes use of this new format.

# **Motivation**

A more flexible transaction format enables adding new fields in the future. This will be the version 1 of the transaction format.

# **Specification**

A transaction version 1 contains a list of fields, some of them are optional and some are required. 

These are the transaction fields: 

- [0] **nonce**: a nonce, if not specified, the value is one.
- [1] **amount**: amount of funds to transfer [optional]. If not specified, the amount is zero.
- [2] **receiver**: the address of the receiver [optional], If not specified, the receiver is the null address, which creates a new contract.
- [3] **gasPrice**: an amount of fee to pay [mandatory]. 
- [4] **gasLimit**: max number of steps to execute [optional]. If omitted, the default value is 30000.
- [5] **data**: arbitrary user data to be sent to the receiver ( used mainly for smart contracts) [optional]
- [6] **signature**: represent a single ECDSA signature, as a list of fields (r,s,v). [mandatory]

All the fields except the signature field are called "persistent". This distinction will be important for the LTCP protocol defined in another RSKIP.  The set of persistent fields whose values are different from their default values is called the Persistent Transaction Information (PTI).  The PTI of a transaction is what is actually stored in the block.

Transactions version 1 are serialized in a new format. Each field is identified by a single byte id, which is the value specified between brackets in the previous list. 

### Transaction 1 Wire Format

The transaction starts with a version byte 0x01, which means the transaction is formatted according to this RSKIP, and not the formatting version 0. Version 0 transactions always start with a byte in the range [0xC0, 0xFF] therefore there is no type collision. The fields are serialized in RLP format after this version byte. The values are interpreted as byte arrays. The most significant byte of each value is the idData. The lower 5 bits of IdData are used to specified the fieldId. The upper 3 bits (flags) are used to store additional flags whose definition depends on the fieldId. After the idData is removed, the remaining bytes specify the value (as in the old format). For example, the following is a valid transaction calling a method in a contract.

0x01 | RLPList( RLP(0x**02**0102030405060708090A0102030405060708090A), RLP(0x**05**01020304))

The first argument represents the field receiver (0x02). The second argument represents the "data" field (0x05).

Currently the flags fields is not used, but may be used by future RSKIPs.

### Wire Format Transaction Validation

The wire-format, which is also used in the block, must not have any additional field. All fields whose values equal their default values must be omitted. If not, then the block will be invalid. All fields 



### Signing

What is actually signed in a transaction is not the PTI, nor the user transaction, but a new compound record with additional information, called the **fullRec**.  The fullRec contains all the fields in a RLP list in a fixed order (the order is given by the fieldIds). Empty fields have ids but no appended data.  

### Gas Cost

The Struct transaction format adds a cost per byte not only in the "data" part, but for the remaining fields also. Each byte, either part of the RLP encoding, version byte, field id or remaining payload, costs 68 gas for non-zero bytes and 4 gas for zero bytes. 

The base gas cost of a version 1 transaction is reduced from 21K to 18K gas.

For example, here is a break-down of the cost of a transaction: 

0x01 | RLPList(  RLP(0x2F) | RLP(0x030102030405060708090A0102030405060708090A), RLP(0x0201))

* Base cost: 18000
* Wire bytes cost:  29 * 68 = 1972

Total cost: 19972

All persistent fields are signed to enable hardware wallets to known exactly what they are signing.

### Block Format

This proposal segregates the signatures (similar to Bitcoin's segwit). This requires a modification to the block format. The new wire-block format is the similar as the previous one but with an additional last optional field:

- wire-block = (header, tx_list, uncle_list ,transaction_list, [, signature_list] ).

The block header semantic is unchanged. The trie of the transaction list is modified. The new transaction list trie can contain both old-style (version 0) transactions and new-style (version 1). All transactions are stored in their corresponding RLP formats. Transactions version 0 are stored with their signatures embedded in the RLP encoding as before, but transactions version 1 are stored without signatures. 

Although the LTCP RSKIP changes this, this RSKIP specifies that all transactions version 1 must have their corresponding signature stored in the signature_list field. 

The signature_list is a RLP list of tuples in the wire-Sig format, defined as:

- wire-Sig = ( txIndex, (r,s,v)) .

txIndex is the index in the tx_list which this signature is associated with. 0 represents the first transaction. Indexes should be specified in ascending order. The signature_list may be empty if there is no transaction version 1 in the block. The RLP-list (r,s,v) is similar to version 0 signature fields. All these fields have maximum sizes and the RLP-lists cannot contain additional values.



### Transaction IDs

A new transaction id is created for transactions version 1, and this id excludes the signature. 

The transaction id is exactly the keccak hash digest of the fullRec. This means that nodes must built the fullRec in order to verify the signature and also to learn the transaction id.

Version 0 transactions will still be identified by their old ids.



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

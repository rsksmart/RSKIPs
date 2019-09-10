# Multi-signed transactions supporting enveloping and multi-key accounts

|RSKIP          |           |
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

Both transaction format 0 and LTCP transaction can be modified for compatibility with multi-signed and multi-key transactions. We'll describe here both formats, although in the future it would be preferable to use two RSKIPs to specify each modification.

First, we define a transaction format 1 which is a simple extension to transaction format 0.



### Transaction Format 1

 Currently the transaction contains the following fields, stored using RLP:

1. Nonce
2. GasPrice
3. GasLimit
4. ReceiveAddress
5. Value
6. Data
7. v
8. r
9. s



Each field begins with an RLP identifier that indicates if the item is an element or a list of elements.

We propose a new format (1), which both add new fields and change the meaning of existing fields:

1. Version

2. Nonce

3. GasPrice

4. GasLimit

5. ReceiveAddress

6. Value

7. Data

8. ownerCount

9. signatures

   

Version is encoded as a list of a single element. For the new transaction format, the element is 1. Therefore the parser can detect if it is a transaction version 0 or a new kind by inspecting only the first element. If it's a list, it's the new type. If not, then it is the nonce of a transaction version 0.

If version is 1, then Nonce can be a a list of elements, or a single element. 

The ownerCount field specifies how many signatures are needed to build a multi-key account. If ownerCount is 1, then it's a single key account. This has the same intent as RSKIP39, but here we propose a more restricted version that does not support multi-key smart-contracts. 

To derive the account address of a multi-key account, the first "ownerCount" public keys are concatenated and the first 20 bytes of the keccack hash of those public keys is the account address.  Multi-key accounts, when combined with enveloping, can help a group of users manage a payment channel without using RBTC, and purely using ERC-20 tokens, relying on an external network transaction relayer.

Signatures contain a list of signatures or a single signature. If there are more than one signature, this is a multi-signed transaction.  The format for each element of the signatures list is an RLP list containing v, r and s for each signer. Each element in Signatures must have exactly 3 elements. 

Nonce can also be a list. The number of element in Nonces can be equal or one higher than the number specified in ownerCount. If the number of nonces is equal,  then there is no enveloping. If the number of nonces is one higher, then the last party to sign is the one paying the fees. 

### LTCP Transaction Format

The format, as specified by RSKIP53 already defines a **signatures** field. We add a new field **nonces** with id 17, which must contain a list of nonces. We also add the field ownerCount with id 18.

Only one field may be included (nonce or nonces) but not both. If ownerCount is not included, the default value is 1. Also additional field chainId (19) is added. The default value is the chainId of the RSK platform. Also the implicit signerIndex (20) field is added.

### Signing the Transaction

Signers do not sign exactly the same payload. Each signer signs the RLP of all the fields with the "signatures" field replaced by a tuple (chainId, signerIndex). The signerIndex is the index of this signer in the list of all signers. Note that all account owner signatures must be listed in a pre-defined order. The transaction will only be valid if each signature signs its correct position. This is to prevent that an attack where owner and enveloper signatures are exchanged by an attacker.

### Transaction Execution

When the transaction is executed, all nonces must be checked. Any invalid nonce invalidates the transaction. Also all nonces must be incremented. If the transaction specifies an enveloper (by containing an additional nonce), then the transaction origin (msg.origin) will still point to the account address, as defined by the other signers public keys. However fees will be deducted from the last signer account. Also the gas reimbursement will be performed to this account.

It would be beneficial to have an opcode or precompile contract to obtain the address of the enveloper, if there is one.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 
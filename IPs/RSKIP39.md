---
rskip: 39
title: Multi-key Accounts
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-02-25
---

# Multi-key Accounts

|RSKIP          |39           |
| :------------ |:-------------|
|**Title**      |Multi-key Accounts|
|**Created**    |25-FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft | 


# **Abstract**

Currently smart wallets need to maintain two accounts. One is a simple account (no code) what is used to consume the gas required to perform operations on the smart wallet. The second is the smart wallet (a contract) that holds of most user funds.

This complicates smart wallet design.

Also this prevents Lightning/Lumino network off-chain channels to be settled over the LTCP protocol, as the Lumino network is unable to create off-chain payment channels from a contract.

To achieve true financial inclusion within the next 10 years we need to allow high transaction volumes, and having the LTCP network be the settlement layer for the Lighning/Lumino network seems the optimal way to achieve it. 

[RSKIP37]  (Single Address Smart Wallets) also addresses the same problem. 

This RSKIP proposes a compromise solution that is simple, can be adapted to the current RSK design, allows multi-key accounts for lightning channel smart-contracts, but not cryptographic algorithm agility.

## Discussion

Each account can be protected by one or more ECDSA private keys. Also accounts can now have code, as contracts. In RSK it is possible that an attacker extracts all funds from a smart-contract if he manages to create private key whose public key address matches the address of the smart-contract. However this is practically unfeasible, sin smart-contract addresses are created as hashes of other information. But the platform itself already allows it.

# **Specification**

We replace the signature part of the transaction (r,s,x) by a list of signatures, each being sig(i). The address of a multi-key account is defined as the hash of the concatenation of the public keys obtained by ECDSA public key recovery of the provided signatures.

To fund a multi-key account, a normal transfer to the multi-key address is used.

To create a multi-key controlled contract requires new features.

We propose several ways to create multi-key controlled contracts and discuss them:

1. CREATEEXT

We add a new opcode CREATEEXT with the following arguments:

- InPubKeysOfs

- InPubKeysSize (or addressPreimage)

- Value

- InOffset

- InSize

This creates a multi-key controlled contract. To prevent the creation of a contract with an address that maps a contract that would be created by another account (standard source+nonce method), the CREATEEXT opcode receives the list of pubkeys, instead of the address itself.

Other methods to prevent the same address aquatering problem would be:

- Prefix each type of address with an address type, so multi-key addresses have a distinct type. (See [RSKIP19] and [RSKIP16])

- Use double-hashes instead of single-hashes to create multi-key contracts. Therefore the CREATEEXT opcode can show a pre-image, without showing all the pubkeys.

2. SETCODE

This opcode takes a multi-key account and converts it to a multi-key contract by adding the code. the SETCODE has the following arguments:

- codeOffset

- codeSize

- Code Signatures

The opcode verifies the signatures, and if correct, adds the code to the multi-key address  derived from the signatures.

3.  The setCode flag

We add a boolean flag **setCode** to transactions. When this flag is true and the the receiver of the transaction is null, instead of the creation of a new smart contract address controlled by no key, the code in the "data" field is executed, and the returned data is attached to the as code to the account. 

The drawback of this method is that attaching code can be easily censored.

4. The SETKEYED opcode.

This opcode changes the behaviour of the initialization code of a contract. When the initialization code of a contract executes, the returned data is the code that is attached to the contract address derived from source+nonce. If the SETKEYED opcode is executed, the address of the newly created code is changed, and it set to the address derived from the signatures. Therefore to create a keyed contract, the following steps must be performed:

- Send some funds to the multi-key address. (an account is created)

- Send a transaction from the multi-key address to null with the code stored in the data field. The code should execute SETKEYED before returning.

Note that since EVM code can be disassembled without ambiguity, and self-modifying code is not allowed, it’s also possible to detect and censor the SETKEYED opcode.

5. RETURN a size with 255th bit set.

To prevent censorship, we can modify the semantics of the RETURN opcode so if the size given has the 255th bit set, then this is interpreted similar to the SETKEYED opcode. To censor this operation, the miner has no other way than to execute the transaction fully.

This last option (5) is therefore selected.


[RSKIP16]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP16.md
[RSKIP19]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP19.md
[RSKIP37]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP37.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
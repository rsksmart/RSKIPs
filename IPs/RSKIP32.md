---
rskip: 32
title: Double-Hashed Addresses
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-01-10
---

# Double-Hashed Addresses

|RSKIP          |32           |
| :------------ |:-------------|
|**Title**      |Double-Hashed Addresses|
|**Created**    |10-JAN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


# **Abstract**

Currently a user can send funds to any 160-bit address. If the address does not exists, it is created as an account. Accounts are created by inserting a node into a state trie. Because the state trie does not re-balance itself, to prevent degeneration of the trie, addresses are hashed before being inserted in the state trie. This forces that for every transaction, the destination address is hashed before a lookup takes place. This consumes CPU resources on each transaction. This RSKIP proposes another method to protect trie degeneration: whenever a new account is created, a pre-image of the address hash must be given. Addresses are built as double-hashes to allow pre-images to be shown. Therefore the cost of hashing is shifted to the moment a new address is created, instead of being necessary for every transaction. Also this method protects from users making typos in addresses.

# **Motivation**

Increased scalability. Also prevents users from sending SBTC to incorrect addresses because the address must exists, and typos in the address won’t end up in an existing address.

# **Specification**

Procedure to create a Double-Hashed address: 

* Create a keypair (Kpriv, Kpub). 

* Let **Ht** = H(Kpub), 

* Let **Ha** = H(Ht). 

Transactions to pre-existent addresses specify **Ha**, while transactions that create new addresses must specify **Ht**. If a transaction specifies both Ha and Ht, then is can act in the two scenarios: if the destination account exists, Ha is used, if not, it is created and Ht is used.

If a contract calls an inexistent contract or sends SBTC to an inexistent contract whose address is a Ha, then the CALL fais and the destination address is not created.

**Security**

Therefore to find a prefix collision an attacker needs to test many addresses until it finds one.

## Implementation in the VM

Many opcodes in the VM use addresses. The following do:

* ADDRESS

* BALANCE

* ORIGIN

* CALLER

* EXTCODECOPY

* COINBASE

* CALL

* CALLCODE

* CREATE

* DELEGATECALL

To enable double-hashed addresses, addresses must be extended by a single byte (the address type). Therefore addresses can be longer than 20 bytes. A 20-byte Double-hashed address will require 21 bytes. Solidity compiler will probably not correctly handle these types of addresses. Also the address extension means that if a 32-byte address is extended with another byte, it will overflow the EVM register size (32-bytes). We therefore should reduce the maximum address size to 31 bytes. This has the drawback that reduced has digests may not be acceptable under certain financial regulations. 

To allow Ethereum contracts compiled with an old Solidity compile to handle correctly 20-byte addresses, when the smart contract is running in an environment version 0 (compatible with Ethereum), we truncate the msg.sender and msg.origin to 20 bytes. The same for the opcodes listed above: we only accept and process 20-byte addresses. 

In environment version 1, we do not truncate. 

To allow 20-byte addresses to work for 32-byte addresses, we must trim 32-byte addresses. To allow this trimming to work, we revert the order of bits in the word-state trie structure. Therefore we can, given a 20-byte prefix, extend it to the closest 32-byte address that matches that prefix. This means that the address type is not part of the 20-byte prefix, which means that version 0 contracts cannot sends funds to nonexistent addresses of a new type. They can, however, send funds or call pre-existent contracts with addresses of any type.

## Pros and Cons

This system requires a new address format specifying Ht, and let wallets decide which to use. If wallets do not know which address to use, they can include both.

This new type of addresses is defined partially in [RSKIP16]  (Combined State Tree) and [RSKIP19]  (RSK Address formats).

[RSKIP16]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP16.md
[RSKIP19]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP19.md


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
---
rskip: 31
title: Hibernation Compression
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-01-01
---

# Hibernation Compression

|RSKIP          |31           |
| :------------ |:-------------|
|**Title**      |Hibernation Compression|
|**Created**    |01-JAN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

Hibernation compression is a method where two hibernated siblings in the state trie are merged into a single hibernated parent. Trie siblings differing in the last bit can be easily compressed because no other contract address is involved. However two siblings that only share a short prefix, but not a long tail allow many addresses to be created in between. Therefore an hibernated mid-node cannot prevent new childs to be created. This RSKIP proposes using a new Trie to allow such compressions, bases on creation date (temporal association) and not based on address association. 

# **Motivation**

The following figure shows how two child contracts are hibernated, and how compression can be carried on.

In (1) there are two child contracts (A and B) whose addresses differ in a single last bit. C is the parent of the child nodes in the trie. In (2) B is hibernated. All the state of B (identified by SB) is hashed, and the hash is stored in the node. In (3) A is hibernated. Afterwards (and automatically) A and B nodes are removed from the trie, and their state hashes is compacted in C by a hash of both hashes. 

![image alt text](./RSKIP31/1MerkleTreeRSKIP31.png)

If later on a transaction sends value to the address of A (01000) the platform can block such transfer because, when traversing the trie, is finds the node C which hash an hibernated state, meaning that both of each children have hibernated.

However this does not happen in the following example:

![image alt text](./RSKIP31/2MerlkeTreeRSKIP31.png)

In this example, in state (4b) a payment sent to the address 0001 must be allowed, because it does not correspond to any of the hibernated nodes. Therefore any payment to an address ending with 0xyz must be allowed, including re-payments to the addresses 0000 and 01111, which will create new fresh accounts. These accounts can again hibernate, adding more hibernation info to node C. To keep constant size hibernation information, the new hibernation hash would join two "copies" of the state of a single contract. Eg:  H(H(SB*) | H( H(SB) | H(SA)). This nested structure is overly complex and does not guarantee O(logN) co-hashes required for waking up a contract.

Therefore compression does not work well.

One can achieve compression by associating accounts by co-temporal creation. A new tree **atree** is created. This is a standard Merkle tree. If depth of the tree is 30, then the tree can hold 2^30 account creations, and no more. When a transaction is processed, all new addresses created in such transaction are stored in T(i).alist. After all transactions have been processed, all T(i).lists are merged into an account-ordered map **amap**. Then all addresses are moved (in address order) from amap to atree, where the key in atree is the account creation number. This is an ever-increasing number. The atree as an immutability property: leaf nodes can be only added at the end, while intermediate nodes can be hibernated and de-hibenated, but no new nodes can be inserted.

The following figure shows how it is achieved. Figure  (1c) shows an initial state of the tree. There are thee accoutns stored: A, B and C, each having a completely different address.  In (2c) a new account D is added to the tree. It is the fouth account added, and the creation index determines the position on the tree. In (3c) tha account A is hibernated (missing information is replaced by a hash digest, marked as an asterisk. In figure (4c) the account B is hibernated, which causes nodes 00 and 01 to be removed, and their states are re-hashed into their parent node 0x. No new account can be created as child of this parent node.

![image alt text](3MTreesRSKIP31.png)

**Pros and Cons**

Having a data structure that maps every account created to an unique number has some benefits, such as allowing to specify the account index number instead of the account hash as a destination, saving bytes in the transaction. A similar approach the same can be accomplished by using a transaction hash prefix as destination, and by forcing the creation of a new account to provide a pre-image of the real address.

Procedure to create an address: Build a keypair. Take the public key PubKey. Let Ht = H(PubKey), let Ha = H(Ht). Transactions to pre-existent addresses specify Ha, while transactions that create new addresses must specify Ht.

Therefore to find a prefix collision an attacker needs to test many addresses until it finds one.

There are several drawbacks: an account with the same prefix can be created just before a user-transaction is included. In that case the transaction should be invalidated to prevent mistakes. Also it requires a new address format specifying Ht, and let wallets decide which to use. 

In case of the account tree solution, once an account is deeply buried in the tree, and there have been many block confirmations, there is no way the index-address can be mistaken, unless the blockchain is deeply reorganized. Also one can make the platform replace account-index in free transactions with their hash address counterparts, and maintain the standard hash addresses for use in contracts.

Maintaining this new data structure would require modifying the state tree at the end of block processing: this prevents bottlenecks from parallel transaction execution. A native smart-contract may access the list of newly created accounts and populate the binary tree with the new addresses. The same smart-contract could handle hibernations and wake-ups.

### New WAKEUP arguments

Arguments: contract_address spv_path_address spv_path_length code_address code_size trie_address trie_size

Returns: error_code

The spv_path consist of tuples (key-part, left-commitment, right-commitment). The key-part consist of a tuple (byte[], bit-length) that represents a part (either prefix or in-between) of the full key. The spv path may include nodes that have already been woken up. These nodes are automatically skipped. 

The left-commitment and right-commitment fields consist of (hash-of-disposed-node, block-height-of-disposal).


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
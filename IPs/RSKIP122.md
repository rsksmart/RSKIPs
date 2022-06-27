---
rskip: 122
title: New method GetBtcTransactionConfirmations for Bridge contract 
description: 
status: Draft
purpose: Usa
author: AM <amendelzon@iovlabs.org>
layer: Core
complexity: 2
created: 2019-05-03
---
# New method GetBtcTransactionConfirmations for Bridge contract

|RSKIP          |122           |
| :------------ |:-------------|
|**Title**      |New method GetBtcTransactionConfirmations for Bridge contract |
|**Created**    |03-May-19 |
|**Author**     |AM |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

A new method for the Bridge contract is introduced. This method provides a service to verify whether a given transaction belongs to a given block within the Bridge contract's view of the Bitcoin blockchain.

## Motivation

With the arising need of BTOs (Bitcoin Token Offerings) on top of the RSK network, the ability for RSK smart contracts to test whether a certain Bitcoin transaction has been mined (and how many blocks ago) becomes mandatory. Note that the possibility of ad-hoc implementations doesn't exist, since to this day there is no method (or group of methods) that enable a smart contract to inspect the Bridge contract's Bitcoin block headers. When designing the implementation, the decision favored a simple unique method tailored at both verifying the inclusion of a given transaction in a given block (via an SPV proof) as well as the inclusion of the latter in the Bridge's Bitcoin blockchain, thus proving or disproving the mined status of any given Bitcoin transaction. As a corollary, the method also informs the number of confirmations for the block, enhancing the calling contract's decision making ability with  a more fine-grained control.

## Specification

A new method for the Bridge precompiled contract (accessible at the `0x0000000000000000000000000000000001000006` address) is available. Its signature is as follows:

- `getBtcTransactionConfirmations(bytes32, bytes32, uint256, bytes32[]) returns (int256)`

### Parameters

The `getBtcTransactionConfirmations` method takes four parameters. In order, these are:

1. The Bitcoin transaction hash (`bytes32`): this is the hash of the transaction that is to be checked for inclusion in the Bitcoin blockchain.
2. The Bitcoin block hash (`bytes32`): this is the hash of the block that is to be checked for both including the given transaction and inclusion in the Bitcoin blockchain (thus then proving the transaction's own inclusion in the blockchain).
3. The merkle branch path (`uint256`): this is the "path" portion of the merkle branch (see the corresponding section for details).
4. The merkle branch hashes (`bytes32[]`): this is the "hashes" portion of the merkle branch (see the corresponding section for details).

### Return values

The `getBtcTransactionConfirmations` returns a single integer (`int256`) as a result of its execution. This can either represent _success_ (i.e., the transaction indeed is included in the Bitcoin blockchain) or _failure_ (i.e., the transaction inclusion in the blockchain couldn't be proved). In the case of _success_, the return value will be greater than zero, and it will indicate how many confirmations the given block - and transaction - have in the Bridge contract's Bitcoin blockchain (note that this could differ from other existing - non-blockchain-based, external - Bitcoin nodes, given that update intervals for the Bridge blockchain can be slower). In the case of _failure_, the return value will be a negative number, and it will show what went wrong. Possible values are:

- `-1`: The block hash given doesn't exist in the blockchain. This could well be due to the aforementioned delay.
- `-2`: The block hash corresponds to a block that is not the Bitcoin best chain.
- `-4`: The block hash given corresponds to a block that is too old to be processed (i.e., its depth with respect to the head of the chain is greater than `4320`).
- `-5`: The given merkle branch fails to prove the inclusion of the given transaction in the given block.
- `-3`: An inconsistency ocurred that prevented the block from being found. This would occur due to an internal, unexpected problem.

### Merkle branch

A merkle branch is a data structure that serves the purpose of proving the inclusion of a single given transaction in a given block. This data structure takes advantage of the block header merkle root and its underlying merkle tree. In fact, a merkle branch is exactly what its name suggests: a list of intermediate hashes from the leaf (corresponding to the desired transaction) to the merkle root, plus information on the "shape" of the branch itself. This information is precisely what the third and four parameters of the `getBtcTransactionConfirmations` method represent. The algorithm to build a merkle branch from a merkle tree is depicted next:

```text
1. Let tx be the hash of the transaction one wishes to prove inclusion on, and mt the corresponding block's merkle tree.
2. Let height(mt) be the height of a given merkle tree.
3. Let mt[i] be the ith level of the tree, with 0 being the leaf level and height(mt) being the root level (this last of size 1, such that mt[height(mt)][0] is the merkle root).
4. Let then mt[i][j] be the jth node of the ith level of the tree.

5. Let txi be the index such that mt[0][txi] equals tx.
6. Let mbhashes be an empty array; lix = txi; mbpath = 0.
7. With level from 0 to height(mt)-1 do:  
7a.    if lix is even:  
7a1.       mbpath = mbpath + (1 << (height(mt)-level-1))
7a2.       append mt[level][lix+1] to mbhashes (use mt[level][lix] if lix+1 is out of range)  
7b.    else:  
7b1.       append mt[level][lix-1] to mbhashes
7c.    lix = floor(lix/2)

8. Return (mbhashes, mbpath).
```

## References

[1] Simplified Payment Verification https://bitcoin.org/en/operating-modes-guide#simplified-payment-verification-spv

[2] Merkle trees https://en.bitcoin.it/wiki/Protocol_documentation#Merkle_Trees

[3] Bitcoin blockchain guide https://bitcoin.org/en/blockchain-guide#introduction

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

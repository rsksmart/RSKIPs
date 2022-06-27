---
rskip: 102
title: Efficient and Secure Fee Bumping
description: 
status: Draft
purpose: Usa, Sec
author: SDL (@sergiodemianlerner)
layer: 
complexity: 2
created: 2018
---
#  Efficient and Secure Fee Bumping  

| RSKIP          | 102                              |
| :------------- | :------------------------------- |
| **Title**      | Efficient and Secure Fee Bumping |
| **Created**    | 2018                             |
| **Author**     | SDL                              |
| **Purpose**    | Usa, Sec                         |
| **Layer**      |                                  |
| **Complexity** | 2                                |
| **Status**     | Draft                            |

# Abstract

This RSKIP proposes a method to bump the transaction gas price of an existing transaction, such as:

1) It can be done by a software component in a less secure environment than the original signer.

2) Does not impose in an additional cost for transactions that are not being fee bumped.

Other wallet software components can then bump the fees of a transaction that is taking too much time to confirm by sending a replacement with higher fees.

# Motivation

Secure wallets generally require user confirmation. Confirming a transaction involves verifying the destination, amount, nonce and data. One typical case are hardware wallets. Generally, after signing, hardware wallets are immediately disconnected and securely stored in safes. If later the transaction does not confirm, the whole ceremony has to be repeated, which increases the exposure time. One simple aproach is that that every time a transaction is confirmed to be signed by the user, N versions of almost the same transaction are created, varying the gas price.  Other wallet modules will then send one of them, and eventually send another if the previous one does not confirm due to low gas price. The drawback is that the network must reverify the signature every time a replacement is broadcast. The cost of signature verification is 3000 gas,  This is 14% of the transaction cost (21K). Since only nodes that are synched would forward and verify free transactions, the cost to the network can be estimated in half of 14%, or 7%, but let's assume full costs. The cost of a non-zero data byte in a transaction is 68 units. Assuming we're analyzing a simple SBTC transfer, then the transaction will consume about 100 bytes, consuming 6800 units of gas, which represents 32% of the transaction cost. Both signature verification and transmission storage together is 9800 gas, which represent 46% of the transaction cost. In this RSKIP we propose a better method that does not require re-verification of a signature, because it uses no-data hash-based signatures, or simple hash-chains, as a secondary means of authentication.  

### The Gas Price Spread and the Transaction Replacement Abuse

By analyzing Ethereum gas prices, we identify two patterns:

1. The average gas price can increase 3x in a day
2. One source shows the spread between fast confirmation (\<2 minutes) and slow confirmation (\>2 minutes) is sometimes as high as 6x. However, other sources shows it's generally not more than 2.5x

We checked this against the state of the gas price spread as of 10-5-2018, as reported by https://ethgasstation.info/.

RSK defines the minimum gas price as the cost of a transaction that is expected to confirm in less than 6 hours. 

The site https://gitcoin.co/gas/history?breakdown=daily shows that daily spread between fast and low confirmations is not as high, but just 2.5X maximum.

If the spread is 2.5X, we can imagine that under heavy load, all users with rush will switch to the highest price, or 2.5X more.

Current RSK nodes use the advertised minimum gas price to decide to forward or block a transaction. In the future, nodes may also estimate the minimum gas price required based on the average gas price in blocks. This can be done with low risk because miners cannot easily manipulate the gas price in blocks as fee revenue is smoothed over several blocks, so faking transaction have a cost. It also can be done by computing the time it takes for free transactions to get included in a block depending on its gas price. This method cannot be manipulated by miners because fake self-paying transactions are not broadcast, because in that case they could be picked by other miners to be inluded, and impose a cost.

Let's assume that a healthy network only broadcasts a replacement if gas price is 40% greater than the previous one.  We can think that under heavy load conditions, the gas price could rapidly increase 3X within  hours (a 3X increase within a day has been observed in Ethereum). Suppose that 80% of the miners vote to increase the minimum gas price, because the average gas price has increased, to match the 6-hour rule. Then approximately every block the minimum gas price increases 4/5 of 1%, in other words, it's multiplied by 1.008. After 138 blocks the minimum difficulty will have tripled.  Normally 138 blocks are mined in approximately 70 minutes. Therefore any attack based on sending replaceable transactions with low gas prices can not last much longer than 1 hour. So let's assume for a moment there is a price spike that triples the gas price without miners having enough time to adjust the block's minimum gas price. This would enable an attacker to spam the network with 3 transactions. The cost to the network 21K plus and additional 9.8K*3=29.4K gas, but only 21K gas is paid. The attacker is abusing the network, but not so much. We should assume that standard replaceable transactions should be more expensive to the user (i.e. 50% more expensive ). 

### Solution

We propose a cheaper approach to solve this problem. Each transaction has an additional 20-byte hash-digest of a privately generated pre-image hash-chain. Each hash in the chain used represents an increase the gas price by 40%. To bump the gas price of a transaction, the user will only need to broadcast a transaction id prefix (20 bytes called shortTxId) and the hash-preimage (20 bytes). This last hash is called the fee bumper pre-image. Let H(s)^i be the repeated hashing of s, i-times. The transaction is first broadcast with a null fee-bumper pre-image, which is h(s)^20. If gas price needs to be increased, a new fee bumper pre-image must be broadcast, so the pair (shortTxId,h(s)^19) is broadcast. If it needs another bump, then (shortTxId,h(s)^18) is broadcast, and so on. The miner will include the transaction with the highest gas price (lower i), together with the fee bumping pre-image that proves the sender accepted the fee bump.

A maximum of 20 pre-images are allowed. As the gas cost of SHA3 is 20 gas per 32 bytes hashed, each has of the chain consumes only 20 gas, which sets a maximum of 680 gas units consumed on hashing. As the transaction id consumes only 20 bytes, the pair consumes 40 bytes, and this represents only 2720 gas paid in bandwidth use. The drawback is that the transaction is 20 bytes longer, which adds a fixed cost of 1360 gas units, which represents 6.4% of the transaction cost.

Therefore a user willing to use this feature would pay 6.8%

### Reducing the Non-replaced Transaction Cost to Zero

It's possible to store a commitment of a 32-byte value in a ECDSA signatures. Basically r is chosen almost according to RFC6979, but with a slight modification.  In step 3.2.h.1 instead of setting T to the empty sequence, we set T to H(s)^20. Therefore the first transaction can be broadcast without hash digest value, and the hidden hash chain leaf could be revealed afterwards. Therefore the transaction without fee bump would not consume more space, and the cost will be minimal. In order to keep determinism, the value "s" is also chosen deterministically from the private key and the message.

## Specification

Wallets that require user confirmation should sign the transaction (as RFC6979 dictates) using H(s)^20 as the initial T value. For the Bitcoin curve, it will not fill the required qlen bits. The hashing function H with an exact domain/image of 160 bits. H should be SHA3, so the result must be truncated to the 160 most significant bits. By using SHA3 we don't need to care about length-extension attacks. The value "s" must be chosen deterministically from private key and message using the same hash function (s = First20Bytes(H(x || m) )). The signer should then export s and the signature itself. The user should be notified in the UI of the number of transactions being signed and the gas price range that is covered. The wallet can chose how many pre-images give to the other (less-secure) wallet component. We recommend at least 5 pre-images. We also recommend that the multiplier is 1.4, generating transactions with the following gas price increases: +0% (1X), +40% (1.4X), +96% (1.96%), +174% (2.74X), +284% (3.84X) ,+437% (5.37X). The multiplier could be part of the transaction data, but we think it shouldn't be necessary.

In a block, a transaction whose fees have been bumped will be published along the bumping pre-image. To enable this, the transaction trie (whose key is the transaction id) will contain for every transaction, and RLPList of the transaction data and the fee bump pre-image, whose size must be either 0 or 20 bytes. Any other item in the RLP list, or the presence of a shorter pre-image, will render the block invalid. 

To process a transaction having a fee bump-pre image, the results should be similar to that first the transaction is modified in memory altering the GASPRICE field, and then is processed as normal. This means that the GASPRICE opcode will not return the value in the transaction data, but the newly computed value. 

## Security

The length of the chain should be low (e.g. 20 slots maximum) to prevent a reduction in the entropy by repeated evaluations. Since the cardinality of the image of H is lower than the domain cardinality, repeated evaluations can reduce the security and alter the statistical uniformity of k. Side channel attacks in the computation of s must be taken into account. Since the Bitcoin curve Q value starts with 64 ones, the probability of the RFC6979  looping because of k >= q is atonishly low, so we can assume the verification of the correctness of a chain slot is still O(1).

Care must be taken that the public  knowledge of H(x || m ) doesn't impact other protocols relying on keeping this data private.

The new structure for storing the transactions in a RLPList enables to store the signature as a witness as a third item in the list, and change the transaction id not to include the witness.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



#  **Fee Bumping**  

| RSKIP          | nn          |
| :------------- | :---------- |
| **Title**      | Fee Bumping |
| **Created**    | 2018        |
| **Author**     | SDL         |
| **Purpose**    | Sec         |
| **Layer**      |             |
| **Complexity** | 2           |
| **Status**     | Draft       |

# Abstract

This RSKIP proposes that wallets that require confirmation (generally hardware wallets) should sign not only a single transaction, but 10 transactions, with ascending gas price. Other software components can then bump the fees of a transaction that is taking too much time to confirm by sending a replacement with higher fees.

# Motivation

Secure wallets generally require user confirmation. Confirming a transaction involves verifying the destination, amount, nonce and data. One typical case are hardware wallets. Generally, after signing, hardware wallets are immediately disconnected and securely stored in safes. If later the transaction does not confirm, the whole ceremony has to be repeated, which increases the exposure time. In this RSKIP we propose that every time a transaction is confirmed, N versions of almost the same transaction are created, varying the gas price.  Other wallet modules will then send one of them, and eventually send another if the previous one does not confirm due to low gas price. The drawback is that the network must reverify the signature every time a replacement is broadcast. The cost of signature verification is 3000 gas,  This is 14% of the transaction cost (21K). Since only nodes that are synched would forward and verify free transactions, the cost to the network can be estimated in half of 14%, or 7%, but let's full costs. The cost of a non-zero data byte in a transaction is 68 units. Assuming we're analyzing a simple SBTC transfer, then the transaction will consume about 100 bytes, consuming 6800 units of gas, which represents 32% of the transaction cost. Both signature verification and transmission storage together is 9800 gas, which represent 46% of the transaction cost. 

Let's assume that a healthy network only broadcasts a replacement if gas price is 40% greater than the previous one.  We can think that under heavy load conditions, the gas price could rapidly increase 3X within in a single day (this has been observed in Ethereum). In one day there are approximately 2.8K blocks, which is enough for miners to adjust the minimum gas price. However, let's assume for a moment there is a price spike that triples the gas price without miners having enough time to adjust the block's minimum gas price. This would enable an attacker to spam the network with 3 transactions. The cost to the network 21K plus and additional 9.8K*3=29.4K gas, but only 21K gas is paid. The attacker is abusing the network, but not so much.

## Specification

Wallets that require user confirmation should sign several versions of a transaction. The user should be notified of the number of transactions being signed and the gas price range that is covered. It is recommended that at least 5 transactions are signed, each one increasing the gas price by 40%. This generates transactions with the following gas price increases: +0% (1X), +40% (1.4X), +96% (1.96%), +174% (2.74X), +284% (3.84X) ,+437% (5.37X).



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



---
rskip: 10
title: Transactions never invalidate blocks 
description: 
status: Accepted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-10-21
---

# Transactions never invalidate blocks

|RSKIP          |10           |
| :------------ |:-------------|
|**Title**      |Transactions never invalidate blocks |
|**Created**    |21-OCT-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Accepted |

# **Abstract**

This RSKIP defines a reorganization of fields in the RSK block header and a change in the semantics of transaction execution to allow nodes to decide if a block is valid without executing any transaction in the block.

# **Motivation**

An e-invalid block is defined as a block with correct PoW but is considered invalid by consensus rules when its transactions are executed. It is generally considered that the attack of creating e-invalid blocks is expensive because miners have to invest electricity to build a block that passes the PoW check. That same amount of electricity could have been spent to mine a block containing a reward, and therefore the attack has at least the cost of electricity and the loss of the block reward. As RSK blockchain is merge-mined, mining RSK blocks have very little additional electricity cost. The block reward will be low during an initial period of the platform. Therefore, in RSK, the miners could try to perform attacks that are normally economically discouraged in other blockchains. Miners that do not participate in the normal merge-mining of RSK could decide to attack the RSK blockchain by mining invalid blocks. Since RSK is planning to improve the network latency by propagating the header plus the transactions IDs (see RSKIP58), the attacker can force the whole network invest CPU resources in partial validation, and force miners to work on invalid parent blocks, if “SPV mining” is used. A single attack has no longstanding consequences, but if sustained, an attack can delay the creation of honest blocks. 

This discussion assumes that each block defines a minimum gas price (minGasPrice, as defined by [RSKIP09]).

# **Specification**

When a block is propagated, it is propagated with all transactions so that the txTrieRoot can be validated. 

This RSKIP proposes the following changes:

1.	Verification-less mining is implemented, as described in RSKIP08.
2.	If a transaction does not have enough balance to cover its gas or value transfer specifications, then the transaction is skipped, along with all transactions following the flawed one. However, the block is valid, and can be added to the blockchain.
3.	The block header includes a new field: prevTxValidCount, which indicate how many transactions in the previous block were valid (all must be consecutive and the counter starts at the first transaction).

To help header-first propagation, the following additional rules are applied:

*RULE 1*. The maximum block size is limited to gasLimit/68. [RSKIP06]

*RULE 2*. The zero discount rule in the data field is removed.  [RSKIP06]

*RULE 3*. The block header includes a new field TxCount that specifies the number of transactions in the block.

*RULE 4*. The maximum number of transactions per block is set to gasLimit/21000
 
The rule 2 prevents a miner from filling a block with transactions without funds, which cannot even pay the cost of block minGastPrice. The miner/attacker can only put a single flawed transaction. The receipt tree must not take into account invalid transactions.

The underlying principle of this design is that once the block header is verified and transactions have been verified to be syntactically correct, the block is valid.

## Side-effects

This change has some side-effects that should be taken into account. SPV wallets can be given proofs that point to transactions which are being skipped, but the SPV client has no way to detect this.
The way to prevent this is that the following block must include the number of transactions from the previous block that have been actually processed correctly (rule 3).
Therefore a SPV proof will include a following header, and SPV clients would need to check it.
Another side effect is that if the miner can include invalid transactions, then nothing prevents the miner from including a huge number of them. There are two ways to prevent this:

1. Transactions are stored in the leaves of a Merkle tree. The first invalid transactions is left, but the following are removed from the tree and from storage.

This does not prevent the invalid transaction to contain a huge amount of data in the “data” field.

2. We limit the maximum size of a block

This is what RSKIP06 does (rule 5).

## Other Competing Proposals

[RSKIP58] presents a header-first block propagation method that does not require verification-less mining and allows arbitrary transaction inclusion. 
[RSKIP58] consumes more bandwidth than this RSKIP, and therefore delays the propagation of blocks compared to RSKIP56, however RSKIP58 is superior because 
it does not require all the modifications to the block header presented in this RSKIP.

[RSKIP58]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP58.md
[RSKIP08]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP08.md
[RSKIP06]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP06.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
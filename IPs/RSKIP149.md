---
rskip: 149
title: Improved asset transfers
description: 
status: Draft 
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2019-11
---
# Improved asset transfers

|RSKIP          |149           |
| :------------ |:-------------|
|**Title**      |Improved asset transfers |
|**Created**    |NOV-2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes a method to parallelize the execution of transactions related to ERC-20 token transfers even if an ERC-20 token account is both sending and receiving tokens.

# **Motivation**

A set of ERC-20 tokens transfers over the same token can be parallelized in a block according to RSKIP144 as long as no token address is both receiving and sending tokens. In practice, it's probable that a single token address receives many payments from different addresses. This situation can easily appear with transaction relayers that receive token payments in exchange of publishing transactions.
Token transfers could be parallelized coding transfers using two commutative primitives:  balance subtractions (BSUB) and balance additions (BADD). This holds as long as the source and destination values are neither inspected nor over-written. The only drawback is that only new token contracts can use this optimization. However proxy contracts can be created that manage a subset of tokens using the new BADD/BSUB primitives.

Finally, to prevent balances to become negative, only final check per sender balance is be performed.

Nota that the use of BADD/BSUB does not guarantee the contract cannot create tokens or burn tokens. This could be enforced by a BMOVE opcode, but this RSKIP does not attempt to do so.

# **Specification**

There are two possible implementations: using native BADD/BSUB opcodes and using a pre-compiled contract. We'll use a pre-compile for simplicity.

A pre-compiled smart-contract is created at address 0x000000000000000000000000000000000100000A with two ABI-coded methods.

The contract should be called with the DELEGATECALL/CALLCODE opcode to give it access to the calling contract storage. If it's called with CALL opcode, it does nothing, but the gas is consumed anyway.

The pre-compiled contract exports two methods:

function badd(uint storageKey,uint amount) external;
function bsub(uint storageKey,uint amount) external;

badd adds a value to a storage cell identified by its key.
bsub substracts the amount from a storage cell identified by its key.

Apart from the cost of DELEGATECALL, the cost of badd/bsub is 10000 gas. 

Internally badd will add amount to a dictionary changeBalance<uint changeKey,uint difBalance>. (mapping a changeKey to a difBalance). Reciprocally, bsub decrements this balance.
It's assumed that all balances in the dictionary start being zero when the transaction starts processing. Afterward difBalance can be become positive or negative. For example, if 100 is added, and then 150 is subtracted, the final value will be -50.
Note that the storageKey is not the same as the changeKey, but represents the same storage cell. 

changeKey can be either a unitrie key, or a pair (account,storagekey), where account is a 20-byte account identifier and storageKey is a 256-bit value. Both implementations should produce the same results. In this specification we'll use a function mapKey(account,storageKey)->changeKey to abstract from the implementation, but using directly unitrie keys is probably more efficient.


The size of difBalance is unbounded, but since each sub/add can add a maximum of 2^255 units to the balance, the maximum difBalance size is limited by 2^255\*blockGasLimit/opGasCost. Assuming blockGasLimit of 6.8M and opGasCost=5700, the maximum size for each balance is 268 data bits plus the sign bit. To allow future expansion, using a variable-length integer is recommended.

badd/bsub behaves as they had modified the storage, but for efficiency reasons the actual change is delayed.
If SLOAD is performed for an address x in a contract a, let y=mapKey(a,x), then first y is looked-up in the changeBalance dictionary. If found, then the change is applied before continuing with SLOAD, and changeBalance(y) is set to zero.
If SSTORE is performed, changeBalance(y) is set to zero and SSTORE continues as normal.

When the block has finished processing, the dictionary changeBalance is iterated. For every entry with non-zero balance the original value is retrieved from the unitrie, and the difBalance is applied. If the resulting value becomes negative, then the block is invalid. If non-negative, the new value is written to the unitrie. If the value is zero, then the storage cell will be removed as if zero had been written with SSTORE. However, the cost of badd/bsub is independent of the final value, and it costs 10000 always. This means that the cost of transferring with badd/bsub will be higher than using SLOAD,SSTORE if the destination cell already accounts exists, but lower if it does not.

## Caveats

As specified until now, the badd/bsub opcodes enable the parallel processing of a set of transactions that otherwise would be invalid by serialized means.
For example, suppose the following state:

- there are two storage keys A,B with balances A=100 and B=0. 
- A transaction txA performs bsub(A,500), badd(B,500)
- A transaction txB performs bsub(B,500), badd(A,500), 
- The block executes txA and txB in parallel.

The final balances of the accounts are unchanged and therefore the block is valid. If token may even emits events such as "500 tokens transferred from A to B" and "500 tokens transferred from B to A". These events  if may confuse wallets or other listeners, because A never had 500 tokens in the first place. Wallets could be prepared to handle this gracefully handled but it's far from an ideal semantic.

To eliminate this problem, we check that after processing the transaction that every entry in changeBalance has a non-negative balance.
This implies that specific sequence of transitive transfers won't be able to be parallelized, such as the following example:

- there are two storage keys A,B with balances A=100 and B=0. 
- A transaction txA performs bsub(A,100), badd(B,100)
- A transaction txB performs bsub(B,100), badd(C,100)

Transaction txB will fail because the balance of B is zero when it begins executing. Therefore txB will not be executed in this block, and will be postponed to a following block (the sender may even be penalized by the miner). This doesn't seem to be an important issue because the owner of B should not have issued txB without first waiting until txA is confirmed in the blockchain.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

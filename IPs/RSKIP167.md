---
rskip: 167
title: Install Code Precompile
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2020-07
---
# Install Code Precompile


|RSKIP          | 167 |
| :------------ |:-------------|
|**Title**      |Install Code Precompile|
|**Created**    |JUL-2020 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

One of the features that can help the blockchain technology reach mass adoption is transaction enveloping (also called meta-transactions). Enveloping allows a third party (sometimes called sponsor or relayer) to pay for the gas of a transaction originating in an account that does not have RBTC.  Implementing this feature efficiently in terms of gas requires modifications to the RSK consensus. One possible solution is to let an EOA become a smart-wallet (also know as "account abstraction" in Ethereum). Once an EOA becomes a smart-wallet, it can be commanded by messages that carry a payload signed by the account owner to perform several tasks atomically. For the purpose of meta-transactions paid in tokens,  the payload must execute two actions:

1. pay in tokens to a relayer (or a contract that is a temporary holder of the tokens).
2. call a contract of user choice. 

Therefore this modification is enough to build a fully functioning meta-transaction layer on top of RSK. In fact, if the smart-wallet becomes the Forwarder component of the Gas Station Network, a huge part of this platform can be adapted from GSN at very low cost. 

# **Specification**

A new precompile contract is created at address: 0x0000000000000000000000000000000001000011

A working implementation with test cases can be found here: https://github.com/rsksmart/rskj/tree/AddCodePrecompile. 

## Execution

This precompile accepts 4 arguments in native serialization form: 

* dwDstAddress (in 32 bytes word,  MSB-padded with 12 zeros)

* v (32 bytes word)

* r (32 bytes word)

* s (32 bytes word)

* code (variable length)

  

The contract will return a single byte which will be 0 on failure and 1 on success. If the return buffer is shorter than 1, no value fill be returned.  The return buffer (accessible with RETURNDATACOPY / RETURNSIZE) will always contain the returned element.

The precompile performs the following actions:

1. if input length is lower than 96, then fail.

2. if any of the signature fields is invalid, the fail.

3. if the dwDstAddress field is not correctly padded with 12 zeros in MSBs, then fail.

4. Extract the address with destinationAddress = last20Bytes(dwDstAddress)

5. if destination account exists, extract the account nonce into dwNonce. (a 32-byte word).

6. if destination account does not exists, assume dwNonce == all-zeros.

7. Compute the code hash with keccack to obtain dwCodeHash (32 bytes word).

8. Build the **hashToSign** according to the following function:

   **hashToSign** = keccak(**message**)
   **message** =  0x19 0x10 dwDstAddress dwNonce dwCodeHash

9. perform an Ethereum address recovery from the signature (v,r,s) on the hashToSign:
   **recoveredAddress** = addressRecovery(v,r,s,hashToSign)

10. if (destinationAddress != recoveredAddress) then fail.

11. if the the destination account does not exists, create it with zero balance and nonce.

12. if destination account is not a contract already, make it a contract (with setupContract).

13. Save the code into the account.

  ​    

## Execution of code inside an EOA

The execution of CREATE inside the EOA would lead to the increment of the account nonce, and that may cancel external transactions with the skipped nonce. This is considered a violation of the mempool invariants. There are two solutions: use a different nonce queue for CREATE or just ban CREATE when executed in the context of an EOA. In this RSKIP for simplicity we opt for banning CREATE, since CREATE2 provides the same functionality without the need to share nonces.

A similar problem occurs if the EOA transfers RBTC. That could collude with RBTCs that are paying transaction fees in transactions in the mempool. Therefore RBTC transfers will also be banned when running in the context of an EOA. A CALL/DELEGATECALL/SELFDESTRUCT or any future opcode that decrements the balance will fail. In the case of CALL, the failure will be equivalent of an immediate REVERT after the CALL. SELFDESTROY will remove the code, but do not transfer the balance to another account.

## Gas Cost

The following conditions are evaluated in the order they are presented. When costs are added, the deduction only occurs at the end, as in any other precompiled contract. The contract never reverts. 

1. If the input length is lower than 96 bytes the cost is zero.
2. If the dwDstAddress field is not zero padded by 12 bytes the cost is zero.
3. A base cost of 10000 gas is added.
4. If the account does not exists, an additional cost of 25000 (GasCost.NEW_ACCT_CALL) is added.
5. If the code length is higher than 64 bytes, a cost of 200 (GasCost.CREATE_DATA) is added per byte over the first 64 (i.e. 65 bytes of code add only 200 to the cost)



## Rationale

The data that is signed includes the account nonce, so the owner of the contract can cancel any pending code installation by issuing a transaction.

The message to sign is built according to EIP191.

The gas cost does not consider the first 64 bytes ad an additional incentive for users to call proxies stored in contract libraries (using static calls). Also because its has been discussed that in a following hard fork the minimum node value size before storing the value separately will be incremented to allow more efficient contract proxies.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

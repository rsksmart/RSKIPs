---
rskip: 209
title: Stack-overflow removal
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-01
---
# Stack-overflow removal


|RSKIP          | 209 |
| :------------ |:-------------|
|**Title**      |Stack-overflow removal|
|**Created**    |JAN-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     |https://research.rsk.dev/t/rskip-209-stack-overflow-removal/147 |


# **Abstract**

One of the contributions to the security of the EVM virtual machine was the removal of the stack-overflow check in [EIP150](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md), making calls fully dependent only on the available gas. However, the way this was accomplished brought several problems, as described in depth [this](https://medium.com/iovlabs-innovation-stories/the-dark-side-of-ethereum-1-64th-call-gas-reduction-ba661778568c) article. This RSKIP proposes a better method to achieve the same result, based on reimbursing the gas locked in `CALL` at the end of the transaction, instead of when `CALL` ends. 


## Motivation

The EIP150 allows the whole state of a call to be specified by only 3 parameters, the value transferred, the calldata and the gas limit. Before, the stack-depth was also part of the environment of a call, that could influence the call outcome. EIP150 introduces a lock of 1/64 of the gas in each `CALL` and this amount is reimbursed when the `CALL` finishes. However, this prevents the easy creation of a gas exactimator. A gas exactimator is a function that can compute the correct gas limit for a transaction by executing it only once.

For this reason the EIP150 did not achieve consensus by the RSK community. This RSKIP brings the same functionality, but in a way that enables a simple gas exactimator.


## Specification

Every  `CALL` locks 1/64 factor of the current available gas, like EIP150, but the locked gas is not immediately reimbursed when the CALL finishes, but at the end of transaction processing. Internally, the environment stores, for each stack depth, the amount of gas locked.  Assuming **lockedGasByDepth** is an array of integer values that are all initialized with zero, the pseudo-code is the following:

```
consumeGas(standard_call_costs); // 700 + value-transfer cost, etc.
callGas = min(top-of-stack item,availableGas)
lockGas = availableGas/64;
maxGasToPassCallee = availGas-lockGas
if (callGas>maxGasToPassCallee) {
 	forbiddenGas =callGas-maxGasToPassCallee
 	if (forbiddenGas>lockedGasByDepth[currentCallDepth]) {
		expectedLock = forbiddenGas-lockedGasByDepth[currentCallDepth];
  		consumeGas(expectedLock);
  		callReimbursement +=expectedLock;
  		lockedGasByDepth[currentCallDepth] = forbiddenGas; // can only increase
	} 
}
```

The command consumeGas() reduces the available gas by the given amount and can rise an OOG exception. 

After the transaction has been processed, the gas callReimbursement is reimbursed to the origin, by the same mechanism other reimbursements occur, such as reimbursements storage freed by `SEFLDESTROY`.

## Rationale

The objective of releasing the gas at the end of the transaction is to enable a gas exactimator to be used. Three methods are compared to decide a good balance between cost for simple and complex transactions, while still preventing DoS attacks. We evaluated 3 parameters sets, involving both constant and proportional gas cost per call:

1. Locking 1/64 of the available gas (as EIP150)
2. Locking a 10000 gas
3. Locking 3000 gas and 1/80 of the available gas

First we present an optimized algorithm for constant amount subtraction.

### Constant Amount Subtraction

The execution environment stores, along the call stack, two non-negative integer values: **maxDepthReached** and **callReimbursement**.  Both values start at zero when transaction execution begins. The following pseudo-code is executed before a `CALL` is performed. The variable **currentCallDepth** refers to the current call stack depth (zero means no call has been performed).

```
if (currentCallDepth==maxDepthReached) {
  consumeGas(10000);
  callReimbursement +=10000;
  maxDepthReached += 1;
} 
```

### Comparison

We compare the 3 algorithms in 4 scenarios A,B,C and D:

- [ ] **A**. Standard DeFi transaction consuming 200k gas and reaching a call depth of 20
- [ ] **B**. Complex  DeFi transaction consuming 500k gas and reaching a call depth of 50
- [ ] **C**. Attack on the client to reach the maximum possible stack depth in order to overflow the program stack, consuming all gas of block with a 10M block gas limit.
- [ ] **D**. Smart-contract that tries to reach the a stack level where it can perform the maximum number of SLOADs in order to force a state cache data structure to query linearly over caches created at each depth, consuming all gas of block with a 10M block gas limit.

We present the results in a table. To compute the results, we add a constant gas consumption of 700 gas per CALL, which accounts for the cost of the `CALL` opcode itself. 

For scenario A we show the amount of gas available at depth 20. For scenario B we show the amount of gas available at depth 50. For scenario C we show the depth reached. For scenario D we show the maximum amount of SLOADs that can be achieved and the depth in brackets.

| Method / Scneario | A               | B               | C    | D                    |
| ----------------- | --------------- | --------------- | ---- | -------------------- |
| 1. gas/64         | 133867          | 203111          | 343  | 1159083 (depth 63)   |
| 2. 10000          | OOG at depth 18 | OOG at depth 46 | 934  | 11682238 (depth 467) |
| 3. gas/80+3000    | 89686           | 128411          | 282  | 1392302 (depth 74)   |

The desired solution should not require a stack deeper than 400. The solution to charge a constant amount (10K) has the benefit that requires a simpler implementation. However, it is expensive for scenarios A and B and does not limit the stack depth enough. The chosen solution (1) surpasses 400 if the transaction gas limit reaches 25M gas, and therefore the block gas limit should be hard limited to 24M. In general, solution 1 provides the best results, and has the added benefit that provides higher compatibility with EIP150.



## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. While contracts are discouraged to embed gas constants in the code, those contracts may break because of the added CALL cost. 

## Test Cases

TBD

## Security Considerations

This RSKIP protects the RSK network from attacks to contracts that perform user-specified calls where gas is paid by a sponsor who is reimbursed. In those cases, a relayer-induced stack overflow can cause the user-specified call to fail and the reimbursement is performed. By limiting the environment that affects a call to controlled set of variables that contracts can check, this risk is mitigated. Other solutions to the same problem are [RSKIP203](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP203.md) (not generic enough) and [RSKIP208](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP208.md), which helps these contracts only if they are programmed to use the precompiled enabled by the RSKIP.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

---
rskip: 174
title: Preserve balance in contract creation
description: 
status: Adopted
purpose: Usa, Fair
author: VK <volodymyr@iovlabs.org>
layer: Core
complexity: 1
created: 2021-07-23
---
# Preserve balance in contract creation

|RSKIP          |174           |
| :------------ |:-------------|
|**Title**      |Preserve balance in contract creation |
|**Created**    |23-JUL-21 |
|**Author**     |VK |
|**Purpose**    |Usa/Fair |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

Contract addresses are deterministic. So it is possible to calculate an address of a contract before actual deployment
(by having an `Externally Owned Account` and its `Nonce`). Due to this property of contract addresses, it's also possible
to send coins to an address of a future contract in advance and, as a result, make its balance positive without having
any deployed contract at that address yet.

A side effect of deploying a smart contract to the address with positive balance is that the balance is being burnt
and becomes zero.

Note: this side effect is only relevant in the case of contract creation via the JSON-RPC interface,
not via `CREATE` / `CREATE2` opcodes (these two opcodes do preserve balances).

## Motivation

There's no benefit of burning such funds after smart contracts creation. The funds are not being sent to any address,
nor are they being re-used in the network in some other way. They simply disappear.

This RSKIP proposes to preserve the balance after contracts creation.

This will also make the network compatible with the Ethereum ecosystem, where pre-funded addresses also preserve
their balances after deploying contracts at those addresses.

## Implementation

The implementation should check if there's any balance by a contract address before creation and add that amount (if any)
to the contract's balance afterwards.

```java
    Coin oldBalance = getBalance(addr);
    newAccount = createAccount(addr);
    addBalance(addr, oldBalance);
```

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

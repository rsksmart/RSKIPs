---
rskip: 412
title: BASEFEE instruction
description:
status: Adopted
purpose: Usa
author: VK
layer: Core
complexity: 2
created: 2023/11/9
---
# BASEFEE instruction


|RSKIP          | 412                 |
| :------------ |:--------------------|
|**Title**      | BASEFEE instruction |
|**Created**    | NOV-2023            |
|**Author**     | VK                  |
|**Purpose**    | Usa                 |
|**Layer**      | Core                |
|**Complexity** | 2                   |
|**Status**     | Adopted             |


# **Abstract**

This RSKIP implements [Ethereum BASEFEE instruction](https://eips.ethereum.org/EIPS/eip-3198). The `BASEFEE (0x48)` opcode returns the base fee value of the current block it is executing in.

# **Motivation**

In order to maintain compatibility with the EVM, Rootstock needs to implement the `BASEFEE` opcode.

There's an important difference when comparing *Rootstock* vs *ETH* `BASEFEE` opcode: as Rootstock doesn't define a base fee for its blocks, the block `minimum gas price` is returned instead. More details on the specification section.

# **Specification**

The `BASEFEE` opcode pushes onto the stack the minimum gas price of the block in which the transaction is being executed. The cost of the `BASEFEE` opcode is 2. 

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)

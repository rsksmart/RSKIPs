---
rskip: 169
title: Rectify EXTCODEHASH implementation
description: 
status: Draft
purpose: Sca
author: NPS
layer: Core
complexity: 2
created: 2020-07
---
# Rectify EXTCODEHASH implementation


|RSKIP          | 169 |
| :------------ |:-------------|
|**Title**      |Rectify EXTCODEHASH implementation|
|**Created**    |JUL-2020 |
|**Author**     |NPS |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


# **Abstract**

EXTCODEHASH opcode was added in RSKIP140 which in turn was modelled after EIP1052. 

Because of a wrongly done implementation, test case number 1 of the RSKIP140 (the `EXTCODEHASH` of the account without code) was not correctly implemented: it returned 0 instead of `c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470` which is the keccack256 hash of empty data. To make things more complex, because of the implementation of MutableCacheImpl, if EXTCODEHASH was run in the same transaction that created the contract, in that case it did return `c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470` instead of zero.

# **Specification**

Adhere to the right implementation after the activation per RSKIP140.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

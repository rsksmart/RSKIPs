---
rskip: 34
title: Contract const DATA Sections
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2017-01-20
---

# Contract const DATA Sections

|RSKIP          |34           |
| :------------ |:-------------|
|**Title**      |Contract const DATA Sections|
|**Created**    |20-JAN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

# **Abstract**

Contracts can be used as data chuncks because CODECOPY allows copying other contracts code. However code passes semantic checks, and RSK adds headers to code. Therefore code is not suitable anymore to hold unformatted data. This RSKIP discusses how to allow const data sections.

## Discussion

There are several possible solutions:

1. The code header will allow the specification of two sections. Only two sections will be allowed: code and data. Each section will specify an base offset. Data will be memory-mapped, so that memory gaps do not consume gas"

2. A new opcode is created: DATA which automatically stops any code analysis. No JUMP will be allowed past this limit. The VM will stop processing if the PC enters the data section as if it was the end of code. The contract that uses CODECOPY must known the offset where data is located.

3. Another approach is that two STOP opcodes in sequence means DATA section starts. The second STOP opcode can not serve any useful use, because there is no JUMPDESTS between them

Solution 3 is simpler but may break compatibility with Ethereum contracts.


# **Specification**

New opcode: DATA

This opcode executes as STOP.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
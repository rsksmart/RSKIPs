---
rskip: 26
title: DUPN and SWAPN opcodes
description: 
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2016-12-27
---

# DUPN and SWAPN opcodes

|RSKIP          |26           |
| :------------ |:-------------|
|**Title**      |DUPN and SWAPN opcodes |
|**Created**    |27-DIC-16 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Pre-git revisions

Revision date: 27/OCT/2017

Revision: 3

Last Modification: Clarification added that opcode arguments are passed on the stack.

# **Abstract**

Currently the RSK/Ethereum opcode set has no way to access elements that are too far in the stack. The opcodes DUP1 to DUP16 allow to retrieve elements up to a depth of 16. The opcodes SWAP1 to SWAP16 allow to swap an element up to depth 16 with the first element. When creating stack frames, and nested stack structures, it’s very common to require access to an element in the stack whose depth must be computed. Therefore the need for DUPN and SWAPN. This RSKIP defines the new opcodes DUPN and SWAPN.

## Discussion

An alternative to passing the argument N is by creating a compound opcode, where N is specified as a 16-bit word in the code addresses following the DUPN/SWAPN opcodes. However this argument passing format makes static code analysis more difficult. However it does not restrict static code analysis since the code is scanned from top to bottom to identify jump destinations before it is executed.

# **Specification**

Both DUPN and SWAPN receive a single argument **on the stack**: the stack depth. The argument is popped from the stack before the command is executed. 

DUPN with depth 0 is equivalent to DUP1. DUPN opcode number is 0xa8.

SWAPN with a depth of 0 is equivalent to SWAP1. SWAPN opcode number is 0xa9.

If there are not enough stack elements to execute DUPN /SWAPN, then an Out of Gas exception will be thrown.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---
rskip: 112
title: Unitrie Node identifiers
description: 
status: Draft
purpose: Sec, Sca
author: SDL (@sergiodemianlerner)
layer: 
complexity: 1
created: 2019
---
#  Unitrie node identifiers

| RSKIP          | 112                      |
| :------------- | :----------------------- |
| **Title**      | Unitrie Node identifiers |
| **Created**    | 2019                     |
| **Author**     | SDL                      |
| **Layer**      | Sec, Sca                 |
| **Complexity** | 1                        |
| **Status**     | Draft                    |

# Abstract

In the future RSK may need to add different types of addresses of different lengths in its Unitrie. Currently the only possible address length is 20 bytes. If this happens, then the length of a trie key will not be enough to uniquely determine the type of a node. In this RSKIP we propose appending a node type identifier to each node that contains data. This will enable future SPV clients to recognize the node type and avoid type-confusion attacks.

# Discussion

Adding one more byte per node implies more resources will be required to store and transfer the trie state. However the benefits outweigh the drawbacks.

 A different approach would be to use some bits that are part of the trie serialization definition to store the type. This idea however is discarded because is mixes elements of different abstraction layers.

# Specification

The non-zero values in the trie are prefixed by the following type specifier as a single byte:

- A for account or contract
- S for contract storage cell
- R for contract storage trie root
- D for coDe (C is reserved in case contracts are assigned their own type specifier)

Specifiers are shown in ASCII.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



```

```

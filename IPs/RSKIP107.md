---
rskip: 107
title: Smaller Unitrie Nodes for Higher Scalability
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2019
---
#  **Smaller Unitrie Nodes for Higher Scalability**  

| RSKIP          | --                                           |
| :------------- | :------------------------------------------- |
| **Title**      | Smaller Unitrie Nodes for Higher Scalability |
| **Created**    | 2019                                         |
| **Author**     | SDL                                          |
| **Purpose**    | Sca                                          |
| **Layer**      | Core                                         |
| **Complexity** | 1                                            |
| **Status**     | Draft                                        |

# Abstract

A RSK trie node consumes 6 bytes, plus the hash digest of the parent node referencing the node, and the key/value database overhead to store the node. This is approximately 80 bytes.  This RSKIP shows how to decrease the overhead for small leaf account nodes from 266% to 7%, with minimal changes, that can be applied concurrently with the Unitrie change.



# Motivation

A node in RSK storage consumes:

- **ARITY**, 1 byte: indicates the tree arity (always 2)
- **flags**, 1 byte: indicates the type of node (secure and hasLongValue bits)
- **bits**: 2 bytes: child presence bitmask
  - bit 0: left node present
  - bit 1: right node present
- **lshared**, int16 (2 bytes): length of shared prefix in bits
- **encodedSharedPath**, variable: shared prefix  (if lshared>0)
- **leftNodeHash**, 32 bytes: left node hash (if node exists)
- **rightNodeHash**, 32 bytes: right node hash (if right node exists)
- **valueHash**, 32 bytes: Hash digest of value stored (if hasLongVal)
- **value**, variable: stored value (!hasLongVal)

The overhead of a leaf node is 6 bytes, plus the hash digest of the parent node referencing the node, and the key/value database overhead to store the node. This is approximately 80 bytes.

## Specification

The new node format is as follows:

- flags, 1 byte: indicates the type of node 
  - **NodeVersion**: 2 bits indicate serialization version (bits 6,7). Currently 01 (bit 6=1).
  - **hasLongValue**: 1 bit indicate if value length > 32 bytes (bit 5)
  - **sharedPrefixPresent**: 1 bit indicates if there is any prefix (bit 4)
  - **nodePresent**: 2 bits indicate left/right embedded node (bit 2 = left, bit 3 = right)
  - **nodeIsEmbedded**: 2 bits indicate left/right node presence (bit 0 = left, bit 1=right)
- if sharedPrefixPresent>0

  - **lsharedCompressed**, uint (0-3 bytes): length of shared prefix, in compressed form.
- if lshared>0

  - **encodedSharedPath**, variable: shared prefix
- if nodePresent[0] 
  - if  !nodeIsEmbedded[0]:
    - **leftNodeHash**, 32 bytes: left node hash
  - if nodeIsEmbedded[0]:
    - **leftNodeSize**, uint8, 1 byte
    - **leftNodeEncoded**, up to 40 bytes: left node encoded
- if nodePresent[1]:
  - if  !nodeIsEmbedded[1]:
    - **rightNodeHash**, 32 bytes: right node hash
  - if  nodeIsEmbedded[1]:
    - **rightNodeSize**, uint8, 1 byte
    - **rightNodeEncoded**, up to 40 bytes: right node encoded
- if hasLongVal:
  - **valueHash**, 32 bytes: Hash digest of value stored
  - **valueLength**, uint24 (3 bytes): size of the value contained (if lvalue>0 and hasLongVal)

- if the left and right nodes are not present:

  - **treeSize**, 0-9 bytes: the size of the tree, variable length integer

- if !hasLongVal:
  - **value**, variable-size extends up to the bounds of the node buffer: stored value 

  

The overhead for a small leaf node is 2 bytes (flags and 1-byte of prefix length)

Because an Unitrie account having 1 RBTC-wei occupies 3 bytes. Assuming a 27-byte shared prefix, the space consumed without node overhead  is 30 bytes. Therefore we've reduced the overhead of small accounts from 266% (80/30) to 7% (2/30).

The value treeSize enables sharding the state tree between nodes and requesting pieces in parallel from different nodes by specifying and offset and size of the requested chunk, while still being able to validate each piece independently, without the need to collect all pieces first.

**Shared Prefix Size Compression**

Let lshared be the actual length of the prefix. The lsharedCompressed expands to lshared as follows:

- range 0..31: lshared = lsharedCompressed  + 1
- range 32..254: lshared = lsharedCompressed + 128
- 255: use from 1 to 9 additional bytes (following) to a Bitcoin VarInt

The small range (0..31) is left for top nodes that create only the branching structure of the trie.  Also it's used for receipts and transactions, where the index is a sequential number.  The second range (32..254) is used for almost all terminal nodes. In the extreme case a sufficiently long prefix collision appears, the third option (2 additional bytes) is used.

**Variable length integer**

Integer can be encoded depending on the represented value to save space. Longer numbers are encoded in little endian.

| Value          | Storage length | Format                                  |
| -------------- | -------------- | --------------------------------------- |
| < 0xFD         | 1              | uint8_t                                 |
| <= 0xFFFF      | 3              | 0xFD followed by the length as uint16_t |
| <= 0xFFFF FFFF | 5              | 0xFE followed by the length as uint32_t |
| -              | 9              | 0xFF followed by the length as uint64_t |



**Recursive Embedding**

While it's possible in a generic trie that one node has an embedded node which in turn has another node embedded, this will be highly rare in the Unitrie for RSK. It can could happen for a group of four short storage cell, if RSKIP108 is implemented. However, to reduce traversal complexity, we limit that embedded nodes must be terminal (they cannot contain child nodes, even if small).


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



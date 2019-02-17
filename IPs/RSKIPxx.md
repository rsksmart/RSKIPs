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
- **valueHash**, 32 bytes: Hash digest of value stored (if lvalue>0 and hasLongVal)
- **value**, variable: stored value  (if lvalue>0 and !hasLongVal)

The overhead of a leaf node is 6 bytes, plus the hash digest of the parent node referencing the node, and the key/value database overhead to store the node. This is approximately 80 bytes.

## Specification

The new node format is as follows:

- flags, 1 byte: indicates the type of node 
  - **NodeVersion**: 1 bits indicate serialization version (bit 7). Currently 0.
  - **hasLongValue**: 1 bit indicate if value length > 32 bytes (bit 6)
  - **sharedPrefixLengthSize**: 2 bit indicates size of shared prefix (bits 4 and 5): 
    - 00 = zero bytes
    - 01 = 1 byte
    - 10 = 2 bytes
  - **nodePresent**: 2 bits indicate left/right embedded node (bit 2 = left, bit 3 = right)
  - **nodeIsEmbedded**: 2 bits indicate left/right node presence (bit 0 = left, bit 1=right)
- if sharedPrefixLengthSize>0
  - **lshared**, uint (0-2 bytes): length of shared prefix (bytes used depending on sharedPrefixLengthSize)
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
- if lvalue>0 and hasLongVal:
  - **valueHash**, 32 bytes: Hash digest of value stored
  - **valueLength**, uint24 (3 bytes): size of the value contained (if lvalue>0 and hasLongVal)

- if the left node is present:

  - **leftTreeSize**, 0-8 bytes: the size of the left tree, variable length

- if lvalue>0 and !hasLongVal:
  - **value**, variable-size extends up to the bounds of the node buffer: stored value 

  

The overhead for a small leaf node is 2 bytes (flags and 1-byte of prefix length)

Because an Unitrie account having 1 RBTC-wei occupies 3 bytes. Assuming a 27-byte shared prefix, the space consumed without node overhead  is 30 bytes. Therefore we've reduced the overhead of small accounts from 266% (80/30) to 7% (2/30).

The value leftTreeSize enables sharding the state tree between nodes and requesting pieces in parallel from different nodes, while still being able to validate each piece independently, without the need to collect all pieces first.

The version bit could be extended to 2 bits, by changing the field sharedPrefixLengthSize to uint16SharedPrefixLength, and forcing that if this bit is not set the default size is uint8. This reduces the compression of middle nodes, which will start to be packed without shared prefix as soon as the number of valued items in the Unitrie increases (e.g. for 1M valued-items, there will be approximately 500k packed middle nodes).

While it's possible in a generic trie that one node has an embedded node which in turn has another node embedded, this will be highly rare in the Unitrie for RSK. Therefore, to reduce traversal complexity, we limit that embedded nodes must be terminal (they cannot contain child nodes, even if small).


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



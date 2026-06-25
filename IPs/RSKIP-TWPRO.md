# Trustless Work Provider Protocol (TWPRO)

|RSKIP          |           |
| :------------ |:-------------|
|**Title**      | Trustless Work Provider Protocol (TWPRO) |
|**Created**    |06-JUL-18 |
|**Author**     |JR, MM |
|**Purpose**    |Usa, Sec |
|**Layer**      |Node |
|**Complexity** |2 |
|**Status**     |Draft* |

## Abstract

This RSKIP proposes a mechanism that allows mining pools to get work and submit solutions from a RSK node they do not host or trust (TWPRO node). This will allow mining pools to mine without the need of hosting a node or to use a TWPRO node as back up in case of self hosted node malfunctioning.

## Motivation

One of the problems of RSK node, as with any piece of software, is uptime. For example, node can be down for maintenance, be suffering a DDoS attack, etc. 
Previously mentioned circumstances are cases where a mining pool can not retrieve work or submit new solutions to that RSK node. From a mining pool's perspective, it implies a full stop on their RSK mining operation and this generates a loss of money. From RSK network perspective, hashing power will be reduced.

Mining pools can continue working when their RSK node is down by connectng to a TWPRO node or hosting a back up RSK node.

Having a TWPRO node available for pools can also make simpler the process of joining RSK network for mining . Reason for the later is that miners will have the option of not hosting their own RSK miner node  and just connect to a TWPRO node. 
Another benefit is that mining pools can be free of performing tasks like keeping RSK node updated with the last version and other maintenance related tasks.

Finally, this new kind of RSK node can allow the existence of a subset of the network focused on providing services mining pools. The moderated trust between the members of this subset could allow optimizations to increase performance and resistence to attacks.

A drawback of this approach is that it is possible for the owner of a TWPRO node to force client mining pools to earn less than the maximum possible. For example, a TWPRO node can add fill a block with lower cost transactions.
Therefore, a TWPRO node allows mining pools to do security checks on the work provided but possible checks are by no means exhaustive. 
Previously mentioned reasons indicate that using a TWPRO node requires a certain degree of trust from its user.

The following approach has been devised in order to add TWPRO feature to RSK node:

1. JSON-RPC mining API must include a method to allow mining pools to set a desired payment address for mining.
2. MinerServer must be able to manage work for more than one client at the same time.
3. MinerServer should have a method to provide information that allows mining pools to verify that payment address included in work received is theirs. This can probably be done by adding this information to `mnr_getWork` method or by just creating an *extended* `mnr_getWork` method
4. Methods for submitting solutions should remain unchanged.

Some extra topics that are worth mentioning:
* Mining pools who want to connect to a TWPRO node must implement this new protocol. 
* TWPRO only extends the existing JSON-RPC mining API so it's backward compatible with all previous uses of this API.

This RSKIP specifies the approach presented earlier. 

## Specification

It is worth describing the new data that must be returned by a TWPRO node mining API. Data must allow the pool to validate that the coinbase used for hashing the block is the right one and that it has not been changed by the TWPRO node.

`blockHashForMergedMining` is the value that references the hash for the block been mined and it is returned in `mnr_getWork` method response. Hash is calculated by applying Keccak256 hashing algorithm to the RLP encoding of the block header. Note that `bitcoinMergedMiningHeader`, `bitcoinMergedMiningMerkleProof`
 and `bitcoinMergedMiningCoinbaseTransaction` header fields are excluded from the block hash calculation.

A TWPRO node must return the data for allowing the pool to add the coinbase and then apply the hashing algorithm to check if there is a match with the value of `blockHashForMergedMining`

Proposed data to be returned is, the RLP encoding of the block header removing the RLP encoding of the coinbase. This will split the data returned in two (`digest_1`, `digest_2`)
Finally, `blockHashForMergedMining` can be calculated from a mining pool's perspective as follows 
`Keccak256_HASH(digest_1 + RLP_ENCODE(coinbase) + digest_2)`


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
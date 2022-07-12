---
rskip: 173
title: Chunk-Based Code Merkleization using the Unitrie
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2020-09
---
# Chunk-Based Code Merkleization using the Unitrie


|RSKIP          | 173 |
| :------------ |:-------------|
|**Title**      |Chunk-Based Code Merkleization using the Unitrie|
|**Created**    |SEP-2020 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


# **Abstract**

Code merkleization and storage rent are considered as the main levers for decreasing block witness sizes in stateless or partial-stateless RSK roadmaps. Here we specify a fixed-sized chunk approach to code merkleization, outline how the transition of existing contracts to this model would look like, and pose some questions to be considered. This RSKIP is inspired EIP-2926, but makes use of the Unitrie to facilitate the upgrade. However there is a major difference with EIP-2926: this EIP mandates a specific format for code storage in the Unitrie to simplify deployment but it **may affect negatively code retrieval time**, therefore it may require larger chunks and more research on the performance implications.

## Motivation

Bytecode would be the main contributor to block witness size for a stateless RSK client. By breaking contract code into chunks and committing to those chunks in a Merkle tree, stateless clients would only need the chunks that were touched during a given transaction to execute it. By using the Unitrie to store the chunks, we simplify considerably the stateless client wire protocol and stateless execution logic.

## Specification

 What follows is structured to have three sections:

1. How a given contract code is split into chunks and then merkleized
2. How to merkleize all existing contract codes during a hardfork
3. What EVM changes are required to account for the new way of representing code

The proposal adds specific optimizations to deal with short codes (smaller or equal to 64 bytes) to maintain high encoding efficiency. These codes are used for proxy contracts.

### Constants

- `CHUNK_SIZE`: 64 (bytes)
- `KEY_LENGTH`: 4 (bytes) starting from unitrie account key
- `METADATA_KEY`: `0x081`
- `NEWCODE_KEY`: `0x081`\<unit24>
- `OLDCODE_KEY`: `0x80`
- `METADATA_VERSION`: 1
- `HF_BLOCK_NUMBER`: to be defined

### Code merkleization

The procedure for merkleizing a contract affects all contracts whose size is equal or larger than 64 bytes. The procedure starts with splitting the code into chunks. To do this assume a function that takes the bytecode as input and returns a list of chunks as output, where each chunk is a tuple `(startOffset, firstInstructionOffset, codeChunk)`. *Note that `startOffset` corresponds to the absolute program counter, and `firstInstructionOffset` is the offset within the chunk to the first instruction, as opposed to data.* This function runs in two passes:

- **First Pass**: Divide the bytecode evenly into `CHUNK_SIZE` parts. The last part is allowed to be shorter. For each chunk, create the tuple, leaving `firstInstructionOffset` as `0`, and add the tuple to an array of `chunks`. Also set `chunkCount` to the total number of chunks.

- **Second Pass**: Scan the bytecode for instructions with immediates or multi-byte instructions (currently only `PUSHN`, opcodes `0x60 .. 0x7f`). Let `PC` be the current opcode position and `N` the number of immediate bytes following, where `N > 0`. Then:

    - **If** `(PC + N) % CHUNK_SIZE > PC % CHUNK_SIZE` (i.e. instruction does not cross chunk boundary), **continue** to next instruction.
    - **Else**, compute the index of the chunk after the current instruction: `chunkIdx = Math.floor((PC + N + 1) / CHUNK_SIZE)`.
        - **If** `chunkIdx + 1 >= chunkCount` then **break** the loop. This is to handle the following case: in a malformed bytecode, or the "data section" at the end of a Solidity contract, there could be a byte *resembeling* an instruction whose immediates exceed bytecode length.
        - **Else**, assuming `nextChunk = chunks[chunkIdx]`, assign `(PC + N + 1) - nextChunk.startOffset` to `nextChunk.firstInstructionOffset`.

The `firstInstructionOffset` fields allows safe jumpdest analysis when a client doesn't have all the chunks, e.g. a stateless clients receiving block witnesses. 

To build a sub-Unitrie we iterate `chunks` and process each entry, `C`, as follows:

- Let `k` be `0x81 chunkIdx` where the chunk index is stored in big-endian extended with zero padding on the left to reach `KEY_LENGTH`. This prevents the code trie from having data in inner nodes. Note that the code trie is not vulnerable to grinding attacks, and therefore the key is not hashed as in the account or storage trie.
- Let `v` be `RLP([firstInstructionOffset, C.code])`.
- Insert `(k, v)` into the `codeTrie`.

We further insert a metadata leaf into `codeTrie`, with the key being `METADATA_KEY` and value being `RLP([METADATA_VERSION, codeHash, codeLength])`. Since the serialized data occupies less than 64 byes, it should fit into a single trie node. This metadata is added mostly to facilitate a cheaper implementation of `EXTCODESIZE` and `EXTCODEHASH`. It includes a version to allow for differentiating between bytecode types (e.g. [EVM1.5/EIP-615](https://eips.ethereum.org/EIPS/eip-615), [EIP-2315](https://eips.ethereum.org/EIPS/eip-2315), etc.) or code merkleization schemes (or merkleization settings, such as larger `CHUNK_SIZE`) in the future.

The root of this `codeTrie` computed it is inserted in the a trie containing only the `METADATA_KEY`. Because the `METADATA_KEY` is a prefix of the `NEWCODE_KEY`, then metadata node becomes the root of the `codeTrie`. For each accounts with code larger or equal to 64 bytes, its `codeTrie` is inserted into the  account trie and the `OLDCODE_KEY` is removed. For accounts with empty code (including Externally-owned-Accounts aka EoAs) there will be no `codeTrie`.

### Updating existing code (transition process)

The transition process involves reading all contracts in the state and apply the above procedure to them. Intuitively the process should take longer than the time between two blocks (but in the order of minutes). We recommend clients to pre-process the changes before the EIP is activated, by populating the state database with the new trie before the hard-forking block occurs, but not too far before so that the garbage collection does not remove the new `codeTrie` nodes in the Unitrie. Alternatively nodes can temporarily disconnect garbage collection and reactivate it some time after the hard fork.

### Unitrie Changes

Minimum unitrie node payload that is not offloaded from the trie node is incremented from 32 bytes to 64 bytes to reduce the overhead of code merkelization and so allow efficient storage of contract proxys.

### Execution of Calls 

The new procedure retrieve the code of a contract when there is no code cache is the following:

1. The node with `OLDCODE_KEY` of the account sub-trie is searched. 
2. If the node exists, then the code is extracted from this node and procedure continues as before.
3. If the node does not exists, then the node with `NEWCODE_KEY` and all its children are retrieved, assembling the full code.

### Opcode and gas cost changes

The gas cost analysis in this section assumes a specific new way of storing the code split in chunks within the unitrie. Stateful nodes could also store the full bytecode as well as its hash and size in the database to speed up loading contracts, but this results in doubling the space consumption for code. If nodes do not cache full code, then the gas scheme would have to be modified to charge per chunk accessed or conservatively charge for traversing the whole code trie. We believe that with storage rent and intelligent in-memory caches the cost to load the code by traversing the `codeTrie` could be reduced so that no DoS attacks are possible. This hypothesis must be tested.

Another option is to load chunks on demand while code is executing. This could benefit large contracts and, it would reduce the cost of cost execution by storage rent. Research is needed to evaluate the performance of on-demand loading under these conditions.

#### Contract creation cost

Contract creation –– whether it is initiated via `CREATE`, `CREATE2`, or an external transaction –– will execute an *initcode*, and the bytes returned by that initcode is what becomes the account code stored in the state. As of now there is a charge of 200 gas per byte of that code stored. This is lower than SSTORE costs (20K gas per 64 bytes [key/value] which results in 312 gas/byte) but considering the overhead of trie nodes in SSTORE, both code byte and SSTORE byte costs seem similar. The code byte cost will not vary much because of the additional hashing required. If the trie hashes are lazily updated, then the hashed data cost is two times the code length. The added cost represents 12 gas units per 32 bytes, or less than a unit of gas per byte. 

We do add a new fixed charge for `COST_OF_METADATA`, which equals to a fixed cost of 5000 gas, whenever the code size is larger than 64 bytes.

Preliminary cost comparison (note these values are *not* informed by benchmarks):

|             | Pre-EIP | Post-EIP | Increase |
| ----------- | ------- | -------- | -------- |
| 50 bytes    | 10000   | 10000    | 0%       |
| 100 bytes   | 20000   | 25000    | 25%      |
| 24576 bytes | 4915200 | 4920200  | 0.1%     |

#### CALL, CALLCODE, DELEGATECALL, STATICCALL, EXTCODESIZE, EXTCODEHASH, and EXTCODECOPY

We expect no gas cost change. Clients may need to perform two lookups instead of one for codes longer than 64 bytes (one for `OLDCODE_KEY` and one for `NEWCODE_KEY`). However, because of proximity of keys and the fact that only one of the nodes will be present, the trie traversal algorithm dos not need to fetch more trie nodes from storage.

#### CODECOPY and CODESIZE

There should be no change to `CODECOPY` and `CODESIZE`, because the code is already in memory, and the loading cost is paid by the transaction/call which initiated it.

## Rationale

More information on the different design options can be found in EIP-2926.

### Rationale behind chunk size

The current recommended chunk size of 64 bytes has been selected based on a few observations. Smaller chunks are more efficient (i.e. have higher [chunk utilization](https://ethresear.ch/t/some-quick-numbers-on-code-merkelization/7260/3)), but incur a larger hash overhead (i.e. number of hashes as part of the proof) due to a higher trie depth. Larger chunks are less efficient, but incur less hash overhead. We plan to run a larger experiment comparing various chunk sizes to arrive at a final recommendation.

### The keys in the code trie (and `KEY_LENGTH`)

Even though the current bytecode limit is `0x6000` (as per [EIP-170](https://eips.ethereum.org/EIPS/eip-170)), we have decided to have a hard cap of `2^30-1` (influenced by [EIP-1985](https://eips.ethereum.org/EIPS/eip-1985)), which makes the proposal compatible with proposals increasing, or "lifting" the code size limit.

### Index-based keys in the code trie

This RSKIP differs from EIP-2926 because we use the chunk index in the codeTrie instead of the chunk offset. This simpler and lowers tree building costs. By using indices as keys the structure of the trie is known only given the number of chunks, which is given in the metadata node, which is always part of a trie path.

## Backwards Compatibility

From the perspective of contracts, the design aims to be transparent, with the exception of changes in gas costs. Outside of the interface presented for contracts, this proposal introduces minor changes to the way contract code is stored, and does not need a substantial modification of the RSK state. However it can only be introduced via a hard fork.

## Test Cases

TBD

Show the `codeRoot` for the following cases:

1. `code=''`
2. `code='PUSH1(0) DUP1 REVERT'`
3. `code='PUSH32(-1)'` (data passing through a chunk boundary)

## Implementation

TBA

## Security Considerations

TBA
# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

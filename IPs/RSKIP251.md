# Support eth_getProof JSON RPC method

|RSKIP          |251           |
| :------------ |:-------------|
|**Title**      |Support eth_getProof JSON RPC method |
|**Created**    |30-06-2021 |
|**Author**     |FJ |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

RSK implements a Unitrie data structure to store all the relevant blockchain data, such as account states and storage values, this particular data structure enables an easy verification of each stored value, just by getting a merkle proof. Currently these proofs are only useful for blockchain nodes, but in order to allow verification of accounts outside the client, we need an additional function delivering us the required proof. 

Combined with a stateRoot (from the blockheader) it enables offline verification of any account or storage-value. 

## Motivation

Offering these function through the RPC-Interface would enable applications to store and send proofs to devices which are not directly connected to the p2p-network and still are able to verify the data. This could be used for mobile applications, which are currently only using a remote client.

Also it might be useful for L2 solutions.

## Specification

Returns the account and storage-values of the specified account including the Merkle-proof.

Params
- address (UNFORMATTED)
- storage keys that we want to prove (if needed)  (UNFORMATTED[])
- block number, block hash or an id (such as “earliest” or “latest”)  (UNFORMATTED)|(TAG)

As response we’ll receive an Account object, composed by:

- balance (QUANTITY)
- codeHash (UNFORMATTED): SHA3 of the code associated to the specifica account (for EOA just retrieve SHA3(EMPTY_BYTE_ARRAY)) 
- nonce (QUANTITY)
- storageHash (UNFORMATTED): hash of the code of the account. For a EOA it will return "0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470" == SHA3(EMPTY_BYTE_ARRAY)
- accountProof (UNFORMATED[]): Array of rlp-serialized MerkleTree-Nodes, starting with the stateRoot-Node, following the path of the SHA3(address) as key.
- storageProof: Array of storage-entries as requested.
    - Storage-entry:
        - Key (UNFORMATTED): Requested storage key 
        - Value (QUANTITY): Value contained for that key
        - Proof (UNFORMATED[]): Array of rlp-serialized MerkleTree-Nodes, starting with the storageHash-Node, following the path of the SHA3(key) as path.

Each unitrie node should be serialized as described in the [RSKIP107](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP107.md)

### Address or storage non existent values

If a non-existent address or key from storage is requested, we’ll need to provide enough information to verify that ausence from the outside world. For example, a non existing key will lead to a different node.

Note that this is not the same as requesting with wrong parameters, if those params are invalid, we just need to respond as a failed request. 

## References

[1] EIP-1186 https://eips.ethereum.org/EIPS/eip-1186

[2] https://blog.rsk.co/noticia/towards-higher-onchain-scalability-with-the-unitrie/

[3] https://github.com/ethereum/EIPs/issues/1186

[4] https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP107.md

[5] https://github.com/rsksmart/rskj/issues/1493



### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# Obtain Bitcoin Block information from bridge methods

|RSKIP          |220           |
| :------------ |:-------------|
|**Title**      | Obtain Bitcoin Block information from bridge methods |
|**Created**    |26-MAR-21 |
|**Author**     |PGP |
|**Purpose**    |Sca,USa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

The Bridge contract includes in its storage information about the BTC blockchain that could be usable for contracts to get cryptoeconomic randomness from it. This information, is not accesible at this moment, preventing possible use cases from onboarding on the platform.

This RSKIP proposes to implement new methods that could let contracts to extract information from Bitcoin blocks to be used as they please.

This change implies a consensus change and should be included in a network upgrade.

## Specification

In order to implement this change the Bridge needs to include a number of new methods that will be described in the following lines.

### getBtcBlockchainBestBlockHeader

#### ABI signature

```
function getBtcBlockchainBestBlockHeader() returns (bytes)
```

#### Specification

This method simply returns the serialized bitcoin block header that is at the moment the tip of the mainchain.

### getBtcBlockchainBlockHeaderByHash

#### ABI signature

```
function getBtcBlockchainBlockHeaderByHash(bytes32) returns (bytes)
```

#### Specification

This method expects to receive a Sha256 value representing a Bitcoin block header hash. If the hash matches a registered bitcoin block it will serialize the header and send it to the user, otherwise, it will return an empty byte array.

### getBtcBlockchainBlockHeaderByHeight

#### ABI signature

```
function getBtcBlockchainBlockHeaderByHeight(uint256) returns (bytes)
```

#### Specification

This method expects to receive an integer value representing a bitcoin block height. The Bridge will lookup its storage for a block header on the mainchain matching the passed height. The height should be lower than the tip of the blockchain, otherwise the method will return an empty array.

This method relies on a block index to avoid expensive lookups. Heights lower than IRIS activation won't return blocks.

### getBtcBlockchainParentBlockHeaderByHash

#### ABI signature

```
function getBtcBlockchainParentBlockHeaderByHash(bytes32) returns (bytes)
```

#### Specification

This methods expects a Sha256 value representing a bitcoin block hash. The Bridge will lookup the block the hash belongs to and later will get the parent of said block as the return value. For this method, if the parameterized block hash or the hash of its parent are not found, it will return an empty array.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
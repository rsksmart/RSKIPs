# Bridge events data and formatting

|RSKIP          |146           |
| :------------ |:-------------|
|**Title**      |Bridge events data and formatting |
|**Created**    |25-OCT-19 |
|**Author**     |MI & JD |
|**Purpose**    |USa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes redefining Bridge events encoding format and the data they include.

## Motivation

The Bridge precompiled contract has several events. Those events are formatted using RLP, which renders them incompatible with the existing web3/solidity tooling (explorer included).
Converting them to a compatible format would help users to easily get the data from those events.

In addition to this, the current specification of the data included within these events is scarce and, at the same time, doesn't expose as much data as potential use cases may need.

The proposal is to redefine the events and their data. To provide the means for a user to track the flow of the critical processes, these being lock, release and federation change.

## Specification

### Formatting

A possible implementation would be creating a `org.ethereum.core.CallTransaction.Function` of an `Event` type, and then use this defined event to generate the properly encoded topics and data.

From `org.ethereum.core.CallTransaction.Function` class, `encodeArguments` method can be used to encode arguments in Solidity format and `encodeSignatureLong` method to encode topic as Keccak256 hash.

### Topics

There will be one topic per event, at least, and four as much following the Solidity specification.

The first topic will represent the full signature of the event encoded as specified in the Solidity docs.
Additionally, other topics could be added to include data considered valuable for filtering out the events. e.g. RSK/BTC Address.

### Data

The data field will join all the remaining information to be emitted. As a general rule, the data should be encoded following the Solidity specification. Additionally, the following clarifications are made:
- Coins: should be encoded as a uint256. We recommend using the smallest available unit, i.e.: Wei or satoshi.
- Numbers: should be encoded in their equivalent solidity type, never as a string.
- Public keys: encode their compressed format. Never encode their hash.
- Hashes, Transaction Ids, and BTC addresses: should be encoded as binary elements. 
- Transactions: encode the hash of the transaction, this applies for both BTC and/or RSK.

### Events

#### Lock - _new event_

A new event was defined when a lock is performed. This event will be emitted when a lock is successfully registered in the RSK network.

Topics: 
- signature: `lock_btc(address indexed receiver, bytes32 btcTxHash, string senderBtcAddress, int256 amount)`
- receiver: where the RBTC will be received

Data:
- btcTxHash: hash of the BTC transaction that produced the lock
- senderBtcAddress: who performed the lock. The string representation of the BTC address, e.g. base58, bech32
- amount: weis transferred to the recipient

#### Update collections

The existing event for update collections will only change its formatting.

Topics:
- signature: `update_collections(address sender)`

Data:
- sender: RSK address of the Federate who performed the call

#### Release requested - _new event_

For the Federation to send BTC to another address, the Bridge itself is responsible for creating the transaction.
There are a number of reasons for this:
- a user sends a transaction to the Bridge, with value and no data. This is when a user manually wants to convert their RBTC to BTC.
- a user locked BTC, but their BTC address is not whitelisted, so the BTC are refunded.
- a user locked BTC, but this transaction makes the Federation balance exceed the configured lockin cap. [see RSKIP-134](RSKIP134.md)
- a retiring Federation migrates its funds to the active Federation.

The Bridge will emmit an event reflecting the requested Release, immediately after the BTC transaction has been created, and before it has been confirmed.

Topics:
- signature: `release_btc_requested(address indexed sender, bytes32 indexed rskTxHash, bytes32 indexed btcTxHash, int256 amount)`
- sender: who requested the release. If this event was emitted in the Federation migration the address will be the Bridge address.
- rskTxHash: hash of the transaction that originated this release.

    If this event was emitted during a user release, the transaction hash will be one of the users calling the Bridge (different to the executing transaction).
  
    If this event was emitted in the Federation migration, the transaction hash to be used will be the executing transaction.

    Once the RSKIP gets implemented, there is a chance there will be events that won't have the rsktTxHash set because the storage won't have the data. In that case, the presence of rskTxHash with value 0x0 should be assumed as release that were requested before the activation of the corresponding Network Upgrade, but were confirmed after its activation.
- btcTxHash: hash of the created Bitcoin transaction

Data:
- amount: weis transferred to the Bridge

### Add signature

This existing event will now link to the RSK transaction that originated the BTC transaction being signed.

Topics:
- signature: `add_signature(bytes32 indexed releaseRskTxHash, address indexed federatorRskAddress, bytes federatorBtcPublicKey)`
- releaseRskTxHash: hash of the transaction that originated this release
- federatorRskAddress: address of the federation member who signed the transaction

Data:
- federatorBtcPublicKey: public key of the Federator signing the release transaction

### Release ready

Once a requested BTC release got the minimum signatures to be broadcasted, this event is emitted.

Topics:
- signature: `release_btc(bytes32 indexed releaseRskTxHash, bytes btcRawTransaction)`
- releaseRskTxHash: hash of the transaction that originated this release

Data:
- btcRawTransaction: full content of the BTC transaction to be manually broadcasted if there were any problems with the automatic broadcasting.

### Commit Federation

During the Federation change process, once the new Federation election is finished, this event is emitted.

Topics:
- signature: `commit_federation(bytes oldFederationBtcPublicKeys, string oldFederationBtcAddress, bytes newFederationBtcPublicKeys, string newFederationBtcAddress, int256 activationHeight)`

Data:
- oldFederationBtcPublicKeys: list of the BTC public keys that conformed the old Federation
- oldFederationBtcAddress: address of the old Federation. The string representation of the BTC address, e.g. base58, bech32
- newFederationBtcPublicKeys: list of the BTC public keys that conform the new Federation
- newFederationBtcAddress: address of the new Federation. The string representation of the BTC address, e.g. base58, bech32

## References

[1] https://solidity.readthedocs.io/en/v0.5.3/abi-spec.html#events

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

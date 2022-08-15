# Change btcDestinationAddress format to string in `release_request_received` event, create new `pegout_confirmed` event, emit `add_signature` when a tx is actually signed.

|RSKIP          |326           |
| :------------ |:-------------|
|**Title**      |Change btcDestinationAddress format to string in `release_request_received` event, create new `pegout_confirmed` event, emit `add_signature` when a tx is actually signed. |
|**Created**    |21-JUNE-22 |
|**Author**     |KI & JT |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

### release_request_received format

As of now, the `btcDestinationAddress` field logged by the `release_request_received` event is a hash160 bytes format which is not user friendly. This RSKIP proposes updating the `btcDestinationAddress` field to the string format which is more user friendly.

### New pegout_confirmed event

Currently no event is being emitted when a pegout that is waiting for confirmation (in the `releaseTransactionSet` structure) has the required confirmations and is moved to waiting for signatures (to the `rskTxsWaitingForSignatures`).
Right now, to check if a pegout has enough confirmation and that is moved to the `rskTxsWaitingForSignatures` structure, we need to get the state of the bridge and look for the pegout in `releaseTransactionSet` or `rskTxsWaitingForSignatures`, which is not that performant or convenient.

We need to add a new event to the Bridge that will be emitted when the Bridge moves a pegout to the `rskTxsWaitingForSignatures`.
I propose this event is called `pegout_confirmed`.

### add_signature event logging order

At the moment the `add_signature` event is being emitted before actually adding the signature.
## Motivation

### release_request_received format

These changes will improve the user experience of getting event information from the blockchain in a human readable/friendly format.

### New pegout_confirmed event

At the moment clients don't have an easy way to know if a pegout has enough confirmations and is already waiting for signatures. With this new event, clients will be able know, probably live, when a pegout is already confirmed and waiting for signatures.

An example of this is the 2wp-app/2wp-api. This tool informs users on the status of their pegout. Right now, to know if a pegout has been confirmed and is waiting for signature this tool makes intricate logic.

### add_signature event logging order

In some cases a signature will not be added to a transaction, so there would not be a need to actually emit the `add_signature` event.

## Specification

### release_request_received changes

#### Current signature:

```
release_request_received(address indexed sender, bytes btcDestinationAddress, uint256 amount)
```

The btcDestinationAddress field is currently in the bytes format.

#### Proposed signature:

```
release_request_received(address indexed sender, string btcDestinationAddress, uint256 amount)
```

The btcDestinationAddress field will be updated to a user friendly string format.

##### Data

- sender: the RSK address of the peg-out requester.
- btcDestinationAddress: the derived BTC address of said RSK address, where the BTC will be transferred to.
- amount: amount in weis the user sent to the Bridge (this doesn't reflect how many BTC will be received since transfer fees will be deducted from that amount).

If `RSKIP326` is activated, the event will log the `btcDestinationAddress` field in a user friendly string format.

### pegout_confirmed event signature and description

This event will exist after the batch pegout consensus changes (HOP, RSKIP271).

```
pegout_confirmed(bytes32 indexed btcTxHash, uint pegoutCreationBlockNumber)
```

- `btcTxHash`: the hash of the Bitcoin transaction without signatures that was just confirmed
- `pegoutCreationBlockNumber`: the block where the pegout transaction was created, this is just informational value and totally optional

### add_signature event logging order

At the end of the `BridgeSupport::addSignature` method, we can see that is calling `eventLogger.logAddSignature` before `BridgeSupport::processSigning`.
So, I propose to remove the `eventLogger.logAddSignature` call from `BridgeSupport::addSignature` and use it inside `BridgeSupport::processSigning` when the signature is actually added to the btcTx.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes, block explorers must be updated. SPV light-clients do not need to be updated. 

## References

[1] [RSKIP-185](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md)
[2] Other RSKIP https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

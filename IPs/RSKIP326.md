# Pegout events improvements

|RSKIP          |326           |
| :------------ |:-------------|
|**Title**      |Pegout events improvements |
|**Created**    |21-JUNE-22 |
|**Author**     |KI & JT |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

Improve pegout related events for a better UX by emitting meaningful events, updating the `btcDestinationAddress` field in `release_request_received` event signature, creating a new `pegout_confirmed` event, and only emitting the `add_signature` event if a signature was actually added.

## Motivation

### release_request_received event data format

These changes will improve the user experience of getting event information from the blockchain in a human readable/friendly format.
Right now, the `btcDestinationAddress` is being stored as the hash160 representation of the address. This RSKIP suggests changing the format to a string representation (`base58` or `bech32` format depending on the address type).

### New pegout_confirmed event

At the moment clients don't have an easy way to know if a pegout has enough confirmations and is already waiting for signatures. With this new event, clients will be able to know, probably live, when a pegout is already confirmed and waiting for signatures.

An example of this is the 2wp-app/2wp-api. This tool informs users on the status of their pegout. Right now, to know if a pegout has been confirmed and is waiting for signature this tool makes intricate logic.

### add_signature event emitting instances

In some cases, a signature will not be added to a transaction when the Bridge method `addSignature` is called (for example, if the given signature is not valid). So there would not be a need to actually emit the `add_signature` event in these cases.

## Specification

### release_request_received changes

As of now, the `btcDestinationAddress` field logged by the `release_request_received` event is a hash160 bytes format which is not user friendly. This RSKIP proposes updating the `btcDestinationAddress` field to its string representation (base58 or bech32) which is more user friendly.

#### Current signature:

```
release_request_received(address indexed sender, bytes btcDestinationAddress, uint256 amount)
```

The btcDestinationAddress field is currently in the bytes format.

#### Proposed signature:

```
release_request_received(address indexed sender, string btcDestinationAddress, uint256 amount)
```

The btcDestinationAddress field will be updated to a user friendly string format (`base58` or bech32 depending on the address type).

##### Data

- sender: the RSK address of the peg-out requester.
- btcDestinationAddress: the derived BTC address of said RSK address, where the BTC will be transferred to.
- amount: amount in weis the user sent to the Bridge (this doesn't reflect how many BTC will be received since transfer fees will be deducted from that amount).

If `RSKIP326` is activated, the event will log the `btcDestinationAddress` field in a user friendly string format.

### New event, pegout_confirmed

Currently no event is being emitted when a pegout that was waiting for confirmation (in the `releaseTransactionSet` structure) has gotten the required confirmations and is moved to waiting for signatures (to the `rskTxsWaitingForSignatures`).
Right now, to check if a pegout has enough confirmation and that is moved to the `rskTxsWaitingForSignatures` structure, we need to get the state of the bridge and look for the pegout in `releaseTransactionSet` or `rskTxsWaitingForSignatures`, which is not that performant or convenient.

We need to add a new event to the Bridge that will be emitted when the Bridge moves a pegout to the `rskTxsWaitingForSignatures`.
I propose this event is called `pegout_confirmed`.


This event will exist after the batch pegout consensus changes (HOP, RSKIP271).

```
pegout_confirmed(bytes32 indexed btcTxHash, uint256 pegoutCreationRskBlockNumber)
```

- `btcTxHash`: the hash of the Bitcoin transaction without signatures that was just created
- `pegoutCreationRskBlockNumber`: the RSK block where the pegout transaction was created, this is just informational value and totally optional

### add_signature event logging order

Emit the `add_signature` event after the transaction has been actually signed, not when there's an attempt to sign it.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes, block explorers must be updated. SPV light-clients do not need to be updated. 

## References

[1] [RSKIP-185](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md)
[2] [RSKIP-271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---
rskip: 415
title: Fix pegnatories address derivation from public keys
created: 30-JAN-24
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |415           |
| :------------ |:-------------|
|**Title**      |Fix pegnatories address derivation from public keys |
|**Created**    |30-JAN-24 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a change in how the Rootstock address of the pegnatories is derived from their public key.

## Motivation

Since RSKIP123 [1] implementation, pegnatories have 3 different keys each. One for Bitcoin transactions, one for Rootstock transactions and a last one, called MST, reserved for future use. When deriving the Rootstock address of a given pegnatory from it's public key, the Rootstock public key should be used.

There are 2 places in rskj code where the pegnatories Rootstock address is being derived from the Bitcoin public key, resulting in an incorrect address value.

- [add_signature event](https://github.com/rsksmart/rskj/blob/FINGERROOT-5.0.0/rskj-core/src/main/java/co/rsk/peg/utils/BridgeEventLoggerImpl.java#L77-L81) When a peg-out transaction is signed by one of the pegnatories this event is emmited, part of the event information is the Rootstock address of the pegnatory that signed.
- [REMASC rewards payment](https://github.com/rsksmart/rskj/blob/FINGERROOT-5.0.0/rskj-core/src/main/java/co/rsk/remasc/RemascFederationProvider.java#L56-L59) [2] Part of the mining fees collected from Rootstock transactions are paid to the current pegnatories. The reward is sent in RBTC to their Rootstock address.

## Specification

Derive the pegnatories Rootstock address from their Rootstock public key instead of from their Bitcoin key when emmiting `add_signature` event and when paying REMASC rewards.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP 123](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP123.md)

[2] [REMASC](https://dev.rootstock.io/rsk/architecture/mining/remasc/)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

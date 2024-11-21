---
rskip: 455
title: PowPeg migration to multiple outputs
created: 20-NOV-24
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |455           |
| :------------ |:-------------|
|**Title**      |PowPeg migration to multiple outputs |
|**Created**    |20-NOV-24 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

Ensure that after a PowPeg composition change the Bridge is still able to process peg-out requests without delays. To do so, update the PowPeg composition change process so that migration transactions include multiple outputs to the new PowPeg address.

## Motivation

When a PowPeg composition change process is executed, all funds are migrated to the new PowPeg address. This is done by creating a transaction that includes up to 50 UTXOs [1] as inputs and a single output to the new PowPeg address. This causes the Bridge to have a small number of UTXOs available after a migration, producing possible delays in processing peg-out requests.

## Specification

Update the process of creating the migration transaction to include multiple outputs to the new PowPeg address, ensuring that the Bridge has enough UTXOs available to operate normally.

### How many outputs

The Bridge contract creates a single peg-out transaction every 360 Rootstock blocks, this transaction includes all the peg-out requests sent during that period [2]. Under current network conditions, this is approximately every 3 hours.

After a peg-out transaction is created it requires 4,000 Rootstock blocks to be confirmed and signed, approximately 33 hours. Once the transaction is broadcasted it requires 100 Bitcoin confirmations for the change UTXO to be registered in the Bridge, approximately 16 hours.  Adding these values, we can estimate that after a UTXO is spent in a peg-out transaction it will take approximately 49 hours for the change UTXO to be available in the Bridge to be used for other peg-out transactions.

Considering one peg-out transaction created every 3 hours, in a 49 hour period the Bridge creates approximately 17 peg-out transactions. Estimating 2 inputs per peg-out, the Bridge requires at least 34 UTXOs to function normally and prevent delays in peg-out processing times.

The proposal is that migration transactions include **50 outputs** to new PowPeg address, to ensure availability for the Bridge to process peg-out requests in time.

### Restrictions

If at the time of the PowPeg composition change there are more than 50 UTXOs available in the Bridge, then multiple migration transactions will be created as specified in RSKIP294 [1]. In this case, the first migration transaction should include the 50 UTXOs with the highest value and 50 outputs to the new PowPeg address. Remaining migration transaction should include a single output.

## Rationale

### Additional cost

Considering a typical transaction output for a P2SH address is 32 bytes, including 50 outputs in a migration transaction adds 1,600 bytes to the transaction size. The cost will vary depending on the network fees at the moment the transaction is created.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP294](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP294.md): Limit the number of inputs to include in a migration transaction

[2] [RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md): Bridge peg-out Batching

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

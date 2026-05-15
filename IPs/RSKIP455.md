---
rskip: 455
title: PowPeg migration to multiple outputs
created: 20-NOV-24
author: MI
purpose: Usa, Sca
layer: Core
complexity: 1
status: Draft
---

# PowPeg migration to multiple outputs

## Abstract

Ensure that after a PowPeg composition change the Bridge is still able to process peg-out requests without delays. To do so, update the PowPeg composition change process so that migration transactions include multiple outputs to the new PowPeg address.

## Motivation

When a PowPeg composition change process is executed, all funds are migrated to the new PowPeg address. This is done by creating a transaction that includes up to 50 UTXOs [[1]](#references) as inputs and a single output to the new PowPeg address. This causes the Bridge to have a small number of UTXOs available after a migration, producing possible delays in processing peg-out requests.

## Specification

Update the process of creating the PowPeg composition change migration transaction to include multiple outputs to the new PowPeg address, ensuring that the Bridge has enough UTXOs available to operate normally.

### How many inputs

Current restriction, implemented as part of RSKIP294 [[1]](#references), limits the number of inputs in a migration transaction to **50**. This restriction was part of a limitation of a previous version of the PowHSM firmware. New versions currently in use by the pegnatories no longer have this restriction.

The new restriction should consider the max standard transaction size accepted by the Bitcoin network. The max standard transaction size defined by Bitcoin is [400,000 weight units](https://github.com/bitcoin/bitcoin/blob/67ea4b9994e668dcea5e5d0f62f886d92e3737dc/src/policy/policy.h#L34C26-L34C48) or [100,000 virtual bytes](https://github.com/bitcoinj/bitcoinj/blob/b2d8af7aad0baff7a4dc2fb9bf67648805327ce7/core/src/main/java/org/bitcoinj/core/Transaction.java#L140), both values representing the same size.

Considering the current PowPeg redeem script format [[3]](#references), with a max number of 20 members in the multisig and adding the flyover information to the redeem script, then the max number of inputs that a migration transaction with 50 outputs can have is **199**. Evidence has been created for [mainnet](https://mempool.space/tx/035373430d0de72e9bf67d0f715f30489109df4ea8ac3924e00e3e63b00dbe6a) and [testnet](https://mempool.space/testnet/tx/23ce5a3d959e3175b337c4c99511f7f48846569bacd5a94902a88f220c0734cd).

The proposal is to select a maximum of **150 inputs** per migration transaction, increasing the current maximum but staying safely away from the upper limit.

### How many outputs

The Bridge contract creates a single peg-out transaction every 360 Rootstock blocks, this transaction includes all the peg-out requests sent during that period [[2]](#references). Under current network conditions, this is approximately every 3 hours.

After a peg-out transaction is created it requires 4,000 Rootstock blocks to be confirmed and signed, approximately 33 hours. Once the transaction is broadcast it requires 100 Bitcoin confirmations for the change UTXO to be registered in the Bridge, approximately 16 hours. Adding these values, we can estimate that after a UTXO is spent in a peg-out transaction it will take approximately 49 hours for the change UTXO to be available in the Bridge to be used for other peg-out transactions.

Considering one peg-out transaction created every 3 hours, in a 49 hour period the Bridge creates approximately 17 peg-out transactions. Estimating 2 inputs per peg-out, the Bridge requires at least 34 UTXOs to function normally and prevent delays in peg-out processing times.

The proposal is that migration transactions include maximum of **50 outputs** to the new PowPeg address, to ensure availability for the Bridge to process peg-out requests in time.

### Strategy

The current logic in the migration process migrates all existing funds to a single output. As a consequence of this behaviour the Bridge always has one "big UTXO" that is the result of the past migration. Smaller UTXOs are then added as users perform peg-ins, but the "big UTXO" is always present. Sometimes it is spent in peg-out operations and as a result a new "big UTXO" is registered in the Bridge, being the change output of the peg-out.

From the moment the "big UTXO" is used in the creation of a peg-out until the change is registered back, 49 hours will go by approximately. If the migration process is executed during the period where the "big UTXO" is not available, this will result in a split of low value UTXOs.

To prevent this, and other scenarios where the Bridge may have low availability of funds, the following strategy is proposed.

1. Given a threshold T = 20 BTC
2. Define MAX_INPUTS = 150 inputs in a migration transaction. Select up to 150 UTXOs and include them in a BTC transaction as inputs
3. Sum up its values, resulting in V
4. If V < 2T, then the migration is made in a single output
5. If 2T ≤ V < 50T, then add multiple outputs of 20 BTC until V value is covered. If the remaining value to add is below 20 BTC, add it to the last output
6. If V ≥ 50T, split the amount into 50 outputs as evenly as possible with sat remainder in the last output

If there are more than 150 UTXOs in the retiring federation or new UTXOs are registered during the migration, the same process should be repeated with the remaining UTXOs until the migration is complete.

The rationale behind these values is taken from an analysis of the historical amount of BTC locked in Rootstock two-way peg [[4]](#references). A minimum of 1,500 BTC has been locked in the two-way peg since 2021.

### Summary

In summary, migration transactions are limited to **150 inputs** and **50 outputs** to be considered standard by the Bitcoin network and properly mined. These values should be revisited if there is ever a change to the current format of the PowPeg redeem script.

During the migration period, the Bridge will try to migrate available funds on each call to `updateCollections`. Every time there are funds available to migrate, it should follow the steps described in [#Strategy](#strategy) section.

## Rationale

### Additional cost

Considering a typical transaction output for a P2SH address is 32 bytes, including 50 outputs in a migration transaction adds 1,600 bytes to the transaction size. The cost will vary depending on the network fees at the moment the transaction is created. 

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP294](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP294.md): Limit the number of inputs to include in a migration transaction

[2] [RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md): Bridge peg-out Batching

[3] [RSKIP305](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md): Peg-out efficiency improvement (Segwit)

[4] [Historical Bridge balance](https://explorer.rootstock.io/address/0x0000000000000000000000000000000001000006?tab=balances)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

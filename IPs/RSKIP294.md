# Limit the number of inputs to include in a migration transaction

|RSKIP          |294           |
| :------------ |:-------------|
|**Title**      |Limit the number of inputs to include in a migration transaction |
|**Created**    |03-FEB-22 |
|**Author**     |MI |
|**Purpose**    |Sca,Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes setting an upper limit to the number of inputs that can be included in a single migration transaction created during a federation change. This is to ensure that the PowHSM devices can properly sign the migration transaction. If necessary, additional migration transactions will be created until all UTXOs registered in the Bridge have been migrated to the new federation.

## Motivation

When a federation change is voted, a single migration transaction is created by the Bridge contract to send all funds from the retiring federation to the newly created federation. This transaction includes one input for each of the UTXOs registered in the Bridge and a single output to the new federation address. This transaction then needs to be signed by the PowHSM devices. These devices have a limitation that allows them to sign a maximum of 253 inputs per transaction. If the Bridge had more than 253 registered UTXOs at the time of a federation change, then the resulting migration transaction would not be signable by the PowHSM devices preventing the federation change to be completed.

## Specification

When creating a migration transaction, the Bridge should include a maximum of N UTXOs in the transaction. If funds remain in the retiring federation wallet to be migrated, those will be included in another transaction created on the next call to updateCollections. As many migration transactions as necessary will be created ensuring none of them has more than the maximum number of inputs defined.

### Maximum number of inputs per transaction

The proposal is to set a maximum of 50 inputs per transaction. Even though this value is pretty far from 253 which is the maximum number of inputs that a PowHSM can sign at once, the advantage is that for every 50 UTXOs migrated the new federation will have one UTXO available reducing in part the UTXO consolidation problem. 

With the current process, all UTXOs in the retiring federation are migrated into a single UTXO in the new federation. This causes delays when processing peg-out requests since only one UTXO is available and after each peg-out is completed, the Bridge needs to wait for 100 BTC blocks to confirm and register the change UTXO.

## Rationale

This limitation on the amount of UTXOs to be included is only for migration transactions, not affecting regular peg-out transactions. The rationale behind this decision is that it's currently highly unlikely that a single peg-out transaction would consume over 253 UTXOs. There are already multiple RSKIPs written with proposals to improve UTXO management in the Bridge that would solve this problem along with other improvements. See [RSKIP265](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md).

## References

[1] RSKIP265 https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---
rskip: 419
title: Powpeg Spendability Validation Protocol
created: 31-AUG-23
author: JD, JZ
purpose: Sec
layer: Core
complexity: 2
status: Draft
description: New Mechanism for current Powpeg to ensure the newly elected Powpeg will safely hold user funds
---

|RSKIP          |419           |
| :------------ |:-------------|
|**Title**      |Powpeg Spendability Validation Protocol |
|**Created**    |31-AUG-23 |
|**Author**     |JD, JZ |
|**Purpose**    |Sec|
|**Layer**      |Core|
|**Complexity** |2|
|**Status**     |Draft|


## Abstract

The Powpeg composition changes from time to time for different reasons, mainly members joining or leaving and/or POWHSM upgrades that require onboarding.
The Powpeg Bitcoin address also changes when the redeemscript is upgraded, for instance when the ERP redeemscript was added.

Everytime this happens, the Bridge undergoes a process that is irreversible: after a certain amount of blocks, the new Powpeg composition will receive all the funds and become responsible for its safety. If the newly elected Powpeg were to be unable to securely store the funds, the users could have their funds at risk.

This RSKIP proposes a new process that will happen before the elected Powpeg is actually committed, and could imply discarding it and stopping the change process if it is considered not safe.


## Motivation

The Powpeg election is a critical process that should have as many reassurances as possible.
As mentioned in the previous section, nowadays once the election takes place, the process is irreversible.
This RSKIP proposes an automated and decentralized process to add one more layer of security to this process to avoid any human mistake putting funds at risk.

This process protects funds from:
- An invalid redeemscript that would render the funds burned.
- A composition of pegnatories that are not responsive and, therefore, unable to sign transactions.


## Specification

This RSKIP proposes changes in the Powpeg change process. The subsections will expand on the changes for each step.

The current process consists in the following phases:
1. Election start: m/n authorized members propose to start a new election.
2. Composition proposal: m/n vote for the new composition.
3. New composition commitment: m/n vote to commit new composition. New elections are disabled.
4. Pre activation phase: immediately after commitment and during a number of blocks. No real change for end users.
5. Activation phase: new Powpeg address activates, two-way peg starts operating from this new address.
6. Migration phase: Funds from the retiring Powpeg wallet are moved to the active Powpeg.
7. Post migration phase: Retiring Powpeg information is removed from the Bridge. New elections are re-enabled.

The election process in itself doesn't change: the authorized members will still propose a new composition and once the voting finishes the commit event will be emitted, but the proposed Powpeg will not be committed yet and its validation will start.


### New phase - Validation

A new phase will be added, in between the _composition proposal_ and _new composition commitment_ phases, the _validation_ phase.
During a certain number of blocks a new protocol will be started, Spendability Validation Protocol (SVP).

If this protocol succeeds, the Bridge will progress to the next phase (_new composition commitment_).
If this protocol fails, things will return to their previous state.


### Spendability Validation Protocol (SVP)

The Spendability Validation Protocol is a new process in which the current Powpeg will send funds to the newly elected Powpeg, for it to send them back, proving that it can truly hold and spend funds, before proceeding with the activation and migration of the funds in the wallet.

This protocol involves three parts:
- Proposed Powpeg funding: the proposed Powpeg won't have funds to create the proof so the current Powpeg will create an UTXO covering the expenses.
- Proof transaction creation: once the proposed Powpeg receives the funds, it will create an UTXO to the current Powpeg to proof that it can spend them.
- Proof transaction verification: the current Powpeg will receive the proposed Powpeg spending-proof transaction, validate it was created by the proposed Powpeg from the funding transaction and proceed with the commitment.

#### Proposed Powpeg funding

Once the SVP starts, the Bridge will send the required satoshis to the proposed Powpeg address. It should be enough for it to be able to send back a transaction. Since the size of the Proof transaction will vary depending on the size of the proposed Powpeg input script, the calculation of the funds sent from the current Powpeg should consider that.


The process consists of the following 3 steps:

##### 1. Funding transaction creation

As soon as the proposed Powpeg has been voted (and saved in a new Bridge storage entry, key `proposedFederation`), the Bridge will create a SPV funding transaction for it.

This transaction is similar to a peg-out where the recipient is the proposed Powpeg Bitcoin address, and the amount to receive is calculated based on the minimum amount of satoshis required to spend an UTXO by it.

Once the transaction is created, it has to be stored in the set for peg-outs waiting for confirmations, and the sighash has to be stored in the pegout index so the change could be registered.

Also, a new value in the storage should be added to identify this special kind of transaction:
    
    KEY: "FUND_TX_HASH_UNSIGNED"
    VALUE: hash of SVP funding tx without signatures

##### 2. Funding transaction confirmation

The SVP funding transaction goes through the usual peg-out confirmation process. Once the transaction accumulates at least 4_000 Rootstock blocks (for mainnet) it should be ready to be moved from the current set to the waiting for signatures set.

##### 3. Funding transaction signing

The funding transaction should be signed as any other peg-out. Once it is fully signed it should be removed from the set and a release_btc event should be emitted.
Eventually, when the funding transaction is registered in the Bridge, the Proof transaction could be created.


### Proof transaction creation

Once the funding transaction is registered, the Proof will be created for the proposed Powpeg to sign and broadcast it so it can prove it can spend funds.

##### Funding transaction registration
The funding transaction should be registered through the registerBtcTransaction method.
For that purpose, the method will be modified to be able to identify it.
It should verify:
- SVP period is ongoing
(i.e. proposedFederation exists & CURRENT_RSK_BLOCK < PROPOSED_FED_CREATION_BLOCK_NUMBER + SVP_PERIOD)
- Transaction hash without signatures matches the record in the FUND_TX_HASH_UNSIGNED storage entry

Once everything is verified, the transaction will be registered and the transaction hash will be saved in a new storage entry, FUND_TX_HASH_SIGNED.
The funding transaction hash signed being saved means that the proof Transaction can be created.

**Important remark**: This method will be public, meaning any user could call it.

#### Proof transaction creation or validation process failure

A new method, `processPowpegChangeValidation` will be called from the `updateCollections` method.

The `processPowpegChangeValidation` will perform some actions if a proposedFederation exists depending on this two scenarios:
- SVP period ended. This implies the validation was not successful.
    -  The Bridge will emit a new event,
        
        `COMMIT_FEDERATION_FAILED(blockNumber indexed uint)`
    - The proposed Powpeg will be eliminated.
    - The election will be allowed once again.

- SVP period didn't end        
    - If the funding transaction signed hash is registered in the FUND_TX_HASH_SIGNED storage entry and the Proof transaction is not created yet, the Bridge will:
        - Create it as described [here](#proof-transaction)
        - Add it to a new storage set, key `spendTxWaitingForSignatures`
        - Set it in a new Storage entry:
            ```
            KEY: "SPEND_TX_HASH"
            VALUE: hash of SVP proof tx
            ```    

##### Proof transaction

The Proof transaction is a Bitcoin transaction where an UTXO from the proposed Powpeg is sent to the current Powpeg.
This transaction will be as simple and small as possible to avoid spending much satoshis as fees.

Proof transaction structure:
```
{
    inputs: [
        {
            txid: FUND_TX_HASH_SIGNED,
            output_index: FUNDING_TX_OUTPUT_INDEX
        }
    ],
    outputs: [
        {
            script: CURRENT_POWPEG_OUTPUT_SCRIPT,
            value: TOTAL_UTXO - FEES
        }
    ]
}
```

#### Proof transaction confirmation

The SVP proof transaction should go through a new "peg-out signing process", since it has to be signed for the proposed pegnatories which belong neither the active nor the retiring powpeg, but the proposed one.

#### Proof transaction signing

The proof transaction should be signed as any other peg-out, but by the proposed powpeg. For this to happen, a new Bridge method `ADD_SPEND_TX_SIGNATURE` should be created to perform the exact same actions as the `ADD_SIGNATURE` but to accept signatures from public keys that don't belong to an active/retiring Powpeg (let's recall the proposed Powpeg is not active at this point).
The `addSpendTxSignature` method should check that the public key belongs to the proposed Powpeg and that the hash of the transaction intended to be signed exists in the `SPEND_TX_HASH` storage entry.

Once fully signed, the Bridge should remove the entry from the `spendTxWaitingForSignatures` and emit the release_btc event as usual.

### Proof verification

With the SVP proof transaction broadcasted, the Bridge now needs to validate it and proceed with the election commitment as expected.

The proof transaction should be registered through the registerBtcTransaction method. As before, it should verify:
- SVP period is ongoing
- Transaction hash without signatures is registered in the `SPEND_TX_HASH` storage key.

If the hash matches the one registered in the storage, then the SVP process is considered completed and we can proceed with the committment of the proposed -and now validated- Powpeg.

### Validation phase duration

The duration of this phase must account for:
- Spending an UTXO from the current Powpeg (Bridge confirmations + pegnatories signing)
- Confirming the UTXO on Bitcoin (Bitcoin confirmations for the UTXO to be accepted)
- Sending back the UTXO to the current Powpeg (Proposed pegnatories signing)

The recommendation is that this phase takes at least 3 times the blocks a peg-out confirmation has. **TODO: CONFIRM THIS**
- Mainnet: 12000
- Testnet: 60 - For testnet it is recommended to wait a bit longer as the Bitcoin miners don't tend to follow the 10' block creation time.
- Regtest: 10

### Bridge Storage changes

|Storage Key   |Type          |Description   |
|:------------ |:-------------|:-------------|
|FUND_TX_HASH_UNSIGNED|bytes32|hash of SVP funding tx unsigned|
|FUND_TX_HASH_SIGNED|bytes32|hash of SVP funding tx signed|
|SPEND_TX_HASH|bytes32|hash of SVP proof tx|

### New Bridge methods
|Method   |Type          |Description   |
|:------------ |:-------------|:-------------|
|ADD_SPEND_TX_SIGNATURE|void|method to sign the proof transaction by the proposed pegnatories|


## Rationale

Discuss design decisions, community debates and possible attacks.

## References

- [RSKIP170 - Peg-in to any address](RSKIP170.md)
- [RSKIP353 - Align RSK P2SH redeem script with Bitcoin Core standard transactions checks](RSKIP353.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

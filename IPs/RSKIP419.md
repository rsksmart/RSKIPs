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
|**Title**      |Powpeg Spendability Validation Protocol|
|**Created**    |31-AUG-23|
|**Author**     |JD, JZ|
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

The election process in itself doesn't change: the authorized members will still propose a new composition and once the voting finishes the commit event will be emitted with the proposed Powpeg information, but the proposed Powpeg will not be committed yet and its validation will start.


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

### Backwards compatibility
This change is a hard-fork and therefore all full nodes must be updated.

#### Proposed Powpeg funding

Once the SVP starts, the Bridge will send the required satoshis to the proposed Powpeg address. It should be enough for it to be able to send back a transaction. Since the size of the Proof transaction will vary depending on the size of the proposed Powpeg input script, the calculation of the funds sent from the current Powpeg should consider that.


The process consists of the following 3 steps:

##### 1. Funding transaction creation

As soon as the proposed Powpeg has been voted (and saved in a new Bridge storage entry, key `proposedFederation`), the Bridge will create a SVP funding transaction for it.

This transaction is similar to a peg-out but with two outputs, where the recipients are the proposed Powpeg Bitcoin address, and the proposed Powpeg Bitcoin address with a custom flyover prefix to make sure the flyover protocol would work too. The amount to receive by both is calculated based on the minimum amount of satoshis required to spend an UTXO by each of them (+ bitcoin fees).

Once the transaction is created, the release request should be settled as if it were a peg-out: it has to be stored in the set for peg-outs waiting for confirmations, the sighash has to be stored in the pegout index so the change could be registered, and the events `release_requested` and `pegout_transaction_created` should be emitted.

Also, a new value in the storage should be added to identify this special kind of transaction:
    
    KEY: "svpFundTxHashUnsigned"
    VALUE: hash of SVP funding tx without signatures

##### 2. Funding transaction confirmation

The SVP funding transaction goes through the usual peg-out confirmation process. Once the transaction accumulates at least 4_000 Rootstock blocks (for mainnet) it should be ready to be moved from the current set to the waiting for signatures set.

##### 3. Funding transaction signing

The funding transaction should be signed as any other peg-out. Once it is fully signed it should be removed from the set and a `release_btc` event should be emitted.
Eventually, when the funding transaction is registered in the Bridge, the Proof transaction could be created.


### Proof transaction creation

Once the funding transaction is registered, the Proof will be created for the proposed Powpeg to sign and broadcast it so it can prove it can spend funds.

##### Funding transaction registration
The funding transaction should be registered through the `registerBtcTransaction` method.
For that purpose, the method will be modified to be able to identify it.
It should verify:
- SVP period is ongoing
(i.e. `proposedFederation` exists & CURRENT_RSK_BLOCK < PROPOSED_FED_CREATION_BLOCK_NUMBER + SVP_PERIOD)
- Transaction hash without signatures matches the record in the `svpFundTxHashUnsigned` storage entry


Once everything is verified, the transaction will be registered and the transaction hash will be saved in a new storage entry, `svpFundTxSigned`. Also, the `svpFundTxHashUnsigned` will be removed from the storage.
The funding transaction hash signed being saved means that the proof transaction can be created.


#### Proof transaction creation or validation process failure

A new method, `updateSvpState` will be called from the `updateCollections` method.

The `updateSvpState` will perform some actions if a proposed federation exists depending on this two scenarios:
- SVP period ended.
Since the `proposedFederation` still exists, this implies the proof transaction was not received, and therefore that the validation was not successful
    -  The Bridge will emit a new event,
        
        `COMMIT_FEDERATION_FAILED(bytes proposedFederationRedeemScript, uint blockNumber)`
    - The proposed Powpeg redeem script will be logged for debugging.
    - The proposed Powpeg will be eliminated.
    - The election will be allowed once again.

- SVP period still ongoing        
    - If the funding transaction signed hash is registered in the `svpFundTxSigned` storage entry, the Bridge will:
        - Create it as described [here](#proof-transaction)
        - Add it to a new storage set, key `svpSpendTxWaitingForSignatures`
        - Set it in a new Storage entry:
            ```
            KEY: "svpSpendTxHashUnsigned"
            VALUE: hash of SVP proof tx without signatures
            ```
        - Remove the `svpFundTxSigned` from the storage    

##### Proof transaction

The Proof transaction is a Bitcoin transaction where an UTXO from the proposed Powpeg is sent to the current Powpeg.
This transaction will be as simple and small as possible to avoid spending much satoshis as fees.

Proof transaction structure:
```
{
    inputs: [
        {
            txid: SVP_FUND_TX_SIGNED_HASH,
            output_index: SVP_FUNDING_TX_OUTPUT_INDEX
        },
        {
            txid: SVP_FUND_TX_SIGNED_HASH,
            output_index: SVP_FLYOVER_FUNDING_TX_OUTPUT_INDEX
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

The proof transaction should be signed as any other peg-out, but by the proposed powpeg. For this to happen, two new Bridge methods `getStateForSvpClient` and `addSvpSpendTxSignature` should be created.

The first one should perform the exact same actions as the `getStateForBtcReleaseClient` but to inform if the SVP spend tx is waiting for signatures (i.e., that `svpSpendTxWaitingForSignatures` is not empty).

The second one should perform the exact same actions as the `addSignature` but to accept signatures from public keys that belong to the proposed Powpeg instead of the active/retiring ones (let's recall the proposed Powpeg is not active at this point), and to check that the hash of the transaction intended to be signed exists in the `svpSpendTxHashUnsigned` storage entry.

Once fully signed, the Bridge should remove the entry from the `svpSpendTxWaitingForSignatures`.

### Proof verification

With the SVP proof transaction broadcasted, the Bridge now needs to validate it and proceed with the election commitment as expected.

The proof transaction should be registered through a new `registerSvpSpendTransaction` method that would behave similar to the `registerBtcTransaction`. As before, it should verify:
- SVP period is ongoing
- Transaction hash without signatures is registered in the `svpSpendTxHashUnsigned` storage key.

If the hash matches the one registered in the storage, then the SVP process is considered completed and we can proceed with the committment of the proposed -and now validated- Powpeg.
The `svpSpendTxHashUnsigned` and the `proposedFederation` will be removed from storage. The utxo sent to the current Powpeg will be registered so it can be spend.

### Validation phase duration

The duration of this phase must account for:
- Spending an UTXO from the current Powpeg (Bridge confirmations + pegnatories signing)
- Confirming the UTXO on Bitcoin (Bitcoin confirmations for the UTXO to be accepted)
- Sending back the UTXO to the current Powpeg (Proposed pegnatories signing)

The recommendation is that this phase takes approximately the blocks a peg-out confirmation has plus the bitcoin blocks needed to register the change, twice. And a couple of thousands more blocks to be safe.
- Mainnet: 20_000
- Testnet: 2_000 - For testnet it is recommended to wait a bit longer as the Bitcoin miners don't tend to follow the 10' block creation time.
_Disclaimer: this implies also increasing the federation activation age to 2400, since the activation period has to be higher than the validation one._
- Regtest: 125

### New Bridge storage fields

|Storage Key   |Description   |
|:------------ |:-------------|
|proposedFederation | voted federation pending to be validated|
|svpFundTxHashUnsigned | hash of SVP funding tx unsigned|
|svpFundTxSigned | SVP funding tx signed|
|svpSpendTxHashUnsigned | hash of SVP proof tx|
|svpSpendTxWaitingForSignatures | SVP proof tx that is to be signed|

### New Bridge methods
```
getProposedFederationAddress() returns string
```
```
getProposedFederationSize() returns int256
```
```
getProposedFederatorPublicKeyOfType(int256 index, string type) returns bytes
```
```
getProposedFederationCreationTime() returns int256
```
```
getProposedFederationCreationBlockNumber() returns int256
```
```
getStateForSvpClient() returns bytes 
```

### New Bridge events
```
commitFederationFailed(bytes proposedFederationRedeemScript, uint blockNumber)
```

### Other changes
To follow EVM common practices, the `getProposedFederationCreationTime` Bridge method returns the value from the epoch seconds.
To be consistent with this, the creation time being returned from `getFederationCreationTime` and `getRetiringFederationCreationTime` Bridge methods is modified to also return the value from the epoch seconds instead of milliseconds.

## References

- [Rootstock devportal - Powpeg](https://dev.rootstock.io/rsk/architecture/powpeg/)
- [Building the Most Secure, Permissionless and Uncensorable Bitcoin Peg](https://medium.com/iovlabs-innovation-stories/building-the-most-secure-permissionless-and-uncensorable-bitcoin-peg-b5dc7020e5ec)
- [Units and globally available variables in Solidity](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#block-and-transaction-properties)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

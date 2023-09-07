---
rskip: pull_request_number_here
title: Powpeg Spendability Validation Protocol
created: 31-AUG-23
author: JD
purpose: Sec
layer: Core
complexity: 2
status: Draft
description: New Mechanism for current Powpeg to ensure the newly elected Powpeg will safely hold user funds
---

|RSKIP          |pull_request_number_here           |
| :------------ |:-------------|
|**Title**      |Powpeg Spendability Validation Protocol |
|**Created**    |31-AUG-23 |
|**Author**     |JD |
|**Purpose**    |Sec|
|**Layer**      |Core|
|**Complexity** |2|
|**Status**     |Draft|

## Abstract

The Powpeg composition changes from time to time for different reasons, mainly members joining or leaving and/or POWHSM upgrades that require onboarding.
The Powpeg Bitcoin address may also change when the redeemscript is upgraded, for instance when the ERP redeemscript was added.
Everytime this happens, the Bridge undergoes a process that is irreversible, after a certain amount of blocks, the new Powpeg composition will receive all the funds and become responsible for its safety. If the newly elected Powpeg were to be unable to securely store the funds, the users could have their funds at risk.

This RSKIP proposes a new process that will happen before the new Powpeg takes over, and could imply reverting the Powpeg election if the new Powpeg is considered not safe.

## Motivation

The Powpeg election is a critical process that should have as many reassurances as possible.
As mentioned in the previous section, nowadays once the election takes place, the process is irreversible.
This RSKIP proposes an automated and decentralized process to add one more layer of security to this process to avoid any human mistake putting funds at risk.

This process protects funds from:
- An invalid redeemscript that would render the funds burned.
- A composition of pegnatories that are not responsive.

This process doesn't include:
- Verifying proposed new members use a valid HSM version.

## Specification

This RSKIP proposes changes in the Powpeg change process. The subsections will expand on the changes for each step.

The election process in itself doesn't change. The authorized members will still propose a new composition, once the voting finishes the new composition will be committed and the Powpeg change will start.

The current process consists in the following phases:
1. Election start: m/n authorized members propose to start a new election.
2. Composition proposal: m/n vote for the new composition.
3. New composition commitment: m/n vote to commit new composition. New elections are disabled.
4. Pre activation phase: immediately after commitment and during a number of blocks. No real change for end users.
5. Activation phase: new Powpeg address activates, two-way peg starts operating from this new address.
6. Migration phase: Funds from the retiring Powpeg wallet are moved to the active Powpeg.
7. Post migration phase: Retiring Powpeg information is removed from the Bridge. New elections are re-enabled.

### New phase - Validation

If this RSKIP is implemented a new phase will be added, in between the _pre activation_ and _activation_ phases, the _validation_ phase. This phase will, in a sense, replace the _preactivation_ phase.

During a certain number of blocks a new protocol will be started, Spendability Validation Protocol (SVP).
If this protocol succeeds, the Bridge will progress to the next phase (_activation_).
If this protocol fails the election will be rollbacked.

### Spendability Validation Protocol (SVP)

The Spendability Validation Protocol is a new process in which the soon-to-be-retired Powpeg will send funds to the newly elected Powpeg, for it to send them back, proving that it can truly hold and spend funds, before proceeding with the activation and migration of the funds in the wallet.
This protocol involves three parts:
- Proof funding: The new Powpeg won't have funds to create the proof so the current Powpeg will create a UTXO covering the expenses.
- Proof creation: Once the new Powpeg receives the funds, it will create a UTXO to the current Powpeg to proof that it can spend funds.
- Proof verification: The current Powpeg will receive the proof, validate it was created by the new Powpeg and proceed.

#### Proof transaction

The Proof transaction is the UTXO that the new Powpeg will create to the current Powpeg.
This transaction will be as simple and small as possible to avoid spending much satoshis as fees.

This is the Proof transaction structure:
```
{
    inputs: [
        {
            txid: FUNDING_TX_ID,
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

#### Proof funding

Once the SVP starts, the Bridge will send the minimum required satoshis to the new Powpeg address. It should be enough for the new Powpeg to be able to send back a transaction. The size of the Proof transaction will vary depending on the size of the new Powpeg input script.

The process consists of the following 3 steps.

##### Funding creation

As soon as a new Powpeg has been committed, the Bridge will create a SPV funding transaction for the new Powpeg.

This transaction is similar to a peg-out where the recipient is the new Powpeg Bitcoin address, and the amount to receive is calculated based on the minimum amount of satoshis required to spend a UTXO by the new Powpeg.

** :warning: TODO: HACER CALCULOS REALES**

Possible calculation:
```
NUMBER_OF_SIGNATORIES => NS
(NS * 73 + 600) * FEE_PER_BYTE

Given 9 signatories and a feePerKb of 30_000 satoshis:
37710 satoshis are to be sent to the new Powpeg
```

Once the transaction is created, it has to be stored in the set for peg-outs waiting for confirmations (i.e. "releaseTransactionSet").
At the same time a new value in the storage should be added to identify this special kind of transaction.

** :warning: TODO: DEBERIA CONSIDERAR MULTIPLES REGISTROS?**
Suggested implementation:
```
KEY: "SVP_FUNDING_TX"
VALUE: sighash of SVP funding tx
```

##### Funding confirmation

The SVP funding transaction goes through the usual peg-out confirmation process. Once the transaction accumulates at least 4_000 Rootstock blocks (for mainnet) it should be ready to be moved from the current set to the waiting for signatures set (i.e. "rskTxsWaitingFS").

##### Funding signing

The funding transaction should be signed as any other peg-out. Once it is fully signed it should be removed from the set and an release_btc event should be emitted.
From now on, the funding transaction should be broadcasted to Bitcoin, and the new Powpeg will be responsible for registering in the Bridge, and create the Proof.

### Proof creation

For the new Powpeg to be able to proof its spendability, the funding needs to be registered, and once registered the Proof will be created, confirmed, signed and broadcasted, for the current Powpeg to validate it.

#### Funding registration

The funding registration should be registered through a new Bridge method, dedicated specifically for SVP operations.

New Bridge method signature:
```
registerSVPBitcoinTransaction(rawBtcTx bytes, blockHeight uint256, partialMerkleTree bytes) returns uint8
```

This method will be publicly callable, but only from EOA addresses.
It will perform a number of verifications:
- SVP period is still enabled (i.e. CURRENT_RSK_BLOCK - NEW_POWPEG_CREATION_BLOCK < SVP_PERIOD) ** :warning: TODO: VALIDAR!!!**
- Transaction hash is not already registered in the Bridge (through accessing storage key "btcTxHashesAP" as already done in `registerBtcTransaction` method)
- Block height is registered and has at least 6 confirmations (notice that we intentionally don't propose using the minimum confirmations of a regular Bitcoin transaction)
- Transaction hash is part of provided partial merkle tree
- Partial merkle tree is well formed and its merkle root matches the merkle root of the provided block height
- Transaction sighash of first input is registered in the new storage (key SVP_FUNDING_TX, see [here](#funding-creation) for more details)

Once everything is verified, the transaction will accepted and the SVP Proof Transaction will be created.

The method has a response value. Please see the possible value [here](#bridge-method-registersvpbitcointransaction---response-values)

**Important remark**: This method will be public, meaning any user could call it.

#### Proof creation

Although this is written in a separated section, this should happen in the same `registerSVPBitcoinTransaction` method.
With the newly registered UTXO, the Bridge will spend it back to the current Powpeg.
The Bridge will create the Proof transaction as described [here](#proof-transaction) and:
- Add the transaction to the release transaction set (storage key "releaseTransactionSet")
- Write to a new Storage entry specific for this process
```
KEY: "SVP_PROOF_TX"
VALUE: sighash of SVP proof tx
```

#### Proof confirmation

The SVP proof transaction goes through the usual peg-out confirmation process. Once the transaction accumulates at least 4_000 Rootstock blocks (for mainnet) it should be ready to be moved from the current set to the waiting for signatures set (i.e. "rskTxsWaitingFS").

#### Proof signing

The proof transaction should be signed as any other peg-out. But for this to happen, the Bridge needs to change the behavior of the method `addSignature` to accept signatures from public keys that don't belong to an active/retiring Powpeg (let's recall the new Powpeg should not be active at this point).
The `addSignature` method should check that the public key belongs to the new Powpeg and that the sighash of the transaction intended to be signed exists in the newly created storage entry (i.e. "SVP_PROOF_TX").

Once fully signed, the Bridge should remove the entry from the signatures set and emit the release_btc event as usual.

### Proof verification

With the SVP proof transaction broadcasted, the Bridge now needs to validated and proceed with the election as expected.

For this to happen, we propose a second purpose for the new Bridge method `registerSVPBitcoinTransaction`.

This means that we should add a new validation, checking the storage entry "SVP_PROOF_TX". If the sighash matches with the one registered in the storage and the rest of the validations are passed as well, then the SVP process is considered completed and the election can proceed.

The Bridge will emit a new event when this happens:
```
SVP_VERIFIED(btcTxHash indexed bytes32, newPowpegCreationBlockNumber indexed uint)
```

### Election rollback

If the time for the new Powpeg to provide the SVP proof expires and no proof is provided, the election needs to be rolled back.

This will happen in the first `updateCollections` execution after the validation time expires.
The new logic will revert the election by making the following changes:
- The new Powpeg must be eliminated.
- The current Powpeg moves back to be active.
- The current Powpeg UTXOs are restablished as active.
- Next powpeg creation block height is removed. (storage key: nextFedCreationBlockHeight)
- Last retired Powpeg is restablished. (storage key: lastRetiredFedP2SHScript) ** :warning: TODO: deberia preservar el valor del powpeg anterior y volver a ponerlo??**
- Election is allowed once again.

The Bridge will emit a new event when this happens:
```
SVP_FAILED(newPowpegCreationBlockNumber indexed uint)
```

### Validation phase duration

The duration of this phase must account for:
- Spending a UTXO from the current Powpeg (Bridge confirmations + pegnatories signing)
- Confirming the UTXO on Bitcoin by the new Powpeg (Bitcoin confirmations for the UTXO to be accepted)
- Sending back the UTXO to the current Powpeg (again Bridge confirmations + pegnatories signing)
The recommendation is that this phase takes at least 3 times the blocks a peg-out confirmation has.

- Mainnet: 12000
- Testnet: 60 - For testnet it is recommended to wait a bit longer as the Bitcoin miners don't tend to follow the 10' block creation time.
- Regtest: 10

### Bridge method registerSVPBitcoinTransaction - response values

```
0 => UNEXPECTED_ERROR
1 => NO_ELECTION_HAPPENING
2 => SVP_FINISHED
3 => NOT_EOA_CALL
4 => TX_ALREADY_REGISTERED
5 => INVALID_TX
6 => INVALID_PMT
7 => INVALID_MERKLE
8 => NOT_ENOUGH_CONFIRMATIONS
100 => SVP_FUNDING_REGISTERED
200 => SVP_PROOF_VALIDATED
```

## Rationale

Discuss design decisions, community debates and possible attacks.

## References

- [RSKIP170 - Peg-in to any address](RSKIP170.md)
- [RSKIP353 - Align RSK P2SH redeem script with Bitcoin Core standard transactions checks](RSKIP353.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

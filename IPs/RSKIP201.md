---
rskip: 201
title: Time-locked Emergency Multisignature
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-01
---
# Time-locked Emergency Multisignature


|RSKIP          | 201 |
| :------------ |:-------------|
|**Title**      |Time-locked Emergency Multisignature|
|**Created**    |JAN-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


# **Abstract**

This RSKIP proposes the addition of a time-locked emergency multi-signature in the Powpeg address, to be used in case the majority of PowHSMs (Powpeg Hardware Security Modules) fail simultaneously.

## Motivation

When RSK was launched in Jan 2018, each one of the functionaries of the RSK federation ran a node in a server computer to protect one of the multisig private keys. These keys were held in hot storage, because the peg-out process is automated. In March 2018, RSK began using HSMs, hardware security modules  exclusively dedicated to protect the private keys against external attackers.  In December 2020, RSK switched to a two-way-peg technology called Powpeg that is  even more secure. In the Powpeg each participant (called pegnatory), runs a RSK node connected to a special HSM called PowHSM. The PowHSM runs a part of the RSK consensus protocol which includes the validation of blocks linkage, cumulative proof-of-work, difficulty adjustments, consecutive block numbers and the best chain selection. This subset of validations is called *SPV mode*. The SPV mode validation performed by the PowHSM is enough to ensure with high confidence that it cannot be tricked into signing a peg-out transaction if the peg-out commands were not originated in the consensus of the RSK best chain, which has the highest proof-of-work. 
In a Powpeg, the pegnatories have almost no power over the bitcoins pegged. However, if the majority of pegnatories decide to (or are forced to) turn their PowHSM devices off, then the bitcoins will be locked forever. Also, if because of a hardware or firmware malfunction all the PowHSM devices get bricked simultaneously, there is no recourse to recover the funds in the peg. This RSKIP proposes that the scriptPub is modified to contain a subscript which represents a time-locked emergency multi-signature. The time-lock expires after one year of complete inactivity of the UTXOs, and the UTXOs need to be periodically renewed to automatically extend the time-lock.  The protocol to automatically renew the time-locks to avoid expiration is specified separately in RSKIP207.

## Specification

The new scriptPub format is the following:

```
OP_NOTIF 
	OP_PUSHNUM_M
	OP_PUSHBYTES_33 pubkey1
	...
	OP_PUSHBYTES_33 pubkeyN
	OP_PUSHNUM_N 
OP_ELSE 
	OP_PUSHBYTES_2 <time-lock-value>
	OP_CSV 
	OP_DROP 
	OP_PUSHNUM_2 
	OP_PUSHBYTES_33 emergencyPubkey1
	OP_PUSHBYTES_33 emergencyPubkey2
	OP_PUSHBYTES_33 emergencyPubkey3
	OP_PUSHNUM_3 
OP_ENDIF 
OP_CHECKMULTISIG


```

The exact value for the time-lock is TBD, but it should correspond to one year either in blocks or by time.

The scriptSig is modified to match the new scriptPub by introducing an OP_0 (push of the zero value) at the beginning.  This element is consumed by the OP_NOTIF opcode, selecting the first sentence of the conditional.

The emergency multi-signature is a 3-out-of-4 multisig.

## Signatories

Following the requirements established in [RSKIP225-Emergency Multisig public keys](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP225.md), the signatories for the emergency multi-signature are listed below:

- IOVlabs ([link to message](https://iovlabs.org/pow-peg-emergency-multisig.html))
- MoneyOnChain ([link to message](https://twitter.com/moneyonchainok/status/1428161275721302027?s=19))
- Jameson Lopp ([link to message](https://keybase.pub/lopp/RSK-key.txt))
- Adrian Eidelman ([link to message](https://keybase.pub/aeidelman/RSK-key.txt))

## Rationale

This proposal uses the OP_CSV opcode. An alternative implementation is that the Powpeg issues a time-locked transaction for every UTXO destined to the peg, including peg-ins and change UTXOs. This alternative is inferior because it requires that the PowHSM performs one additional signature per peg-in and peg-out transaction, and also leaves peg-ins funds unprotected until the 100 block confirmation period elapses and the funds are accepted by the Bridge smart-contract.

Regarding the addition of OP_0 to the scriptSig, it's possible to eliminate it by adding the following 3 lines at the beginning of the scriptPub: 

```
OP_DEPTH 
OP_PUSHNUM_3 
OP_EQUAL 
```

These lines allow selecting between the normal script branch, and the emergency script branch basing the decision on the number of arguments in the script stack. If the number of arguments in the stack is not 3, then the first sentence of the conditional will be chosen. This is disadvantageous because it consumes more script bytes overall, and prevents the Powpeg to have only 3 pegnatories.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

## Test Cases

TBD

## Implementation

TBD

## Security Considerations

Any attempt to preserve the bitcoins in the peg available in case of an emergency represents a trade-off between availability and security. By using an emergency multi-signature that is delayed one year we prevent the two main vectors of attack:
1. Extortion by the pegnatories threatening users to block the funds in the peg.
2. Denial of service attacks to the pegnatories by the emergency multisig participants seeking to delay the recycle of the time-locks in order to take possession of the pegged funds.

At the same time, a one year period is a period of time users may be willing to wait to recover their funds in case of a disastrous failure.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



---
rskip: 305
title: Peg-out efficiency improvement (Segwit)
description: This RSKIP introduces Segwit V0 (Bitcoin Soft Fork 2017) for use in peg-ins and peg-outs to improve performance, cost reductions, and security.
status: Adopted
purpose: Sca Usa Sec
author: PDG (@patogallaiovlabs), RFV (@ramsesfv), NV (@NVescovo)
layer: Core
complexity: 2
created: 2022-06-01
---

# Peg-out efficiency improvement (Segwit)

|RSKIP          | 305        |
| :------------ |:-------------|
|**Title**      |Peg-out efficiency improvement (Segwit) |
|**Created**    |01-JUN-2022 |
|**Author**     |Patricio Gallardo, Ramsès Fernàndez-València, Nicolás Vescovo|
|**Purpose**    |Sca Usa Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

This RSKIP introduces Segwit V0 (Bitcoin Soft Fork 2017) for use in peg-ins and peg-outs to improve performance, cost reductions, and security.

## Motivation

The RSK peg out uses multisignatures to implement a threshold scheme that protects the bitcoins locked. Multi-signatures are simple and relatively 
efficient for small groups, but they become expensive as the number of signers increases. Each input signed in a peg-out transaction must transmit
m-out-of-n signatures, and each signature consumes 65 bytes, and n public keys, consuming 33 or 65 bytes each, depending on if they are compressed or not.
Bringing Segwit to the RSK peg will tackle all these problems.

## Specification

At the time of writing, the Bridge contract works with a Powpeg, which is an abstraction for a set of distinct secp256k1 public keys, which can ultimately be used to represent a Bitcoin multisig (N of M) P2SH.  The upgrade to this native contract involves replacing the current representation format type with a Segwit Compatible (P2SH-P2WSH).

### Create a new Powpeg type

In order to replace the current powpeg representation, a new powpeg type is going to be created incrementing by one the value of the version type and save a new slot in the powpeg storage. For more information refer to [this detailed description](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP123.md#storage-upgrade) on how a powpeg upgrade should be managed.

Although the schema and content of this slot is not to be changed more than the version number, it will serve to indicate how the powpeg multisig redeem script should be regenerated.

Currently the redeem script is as follow:
* `<redeemScript>`:
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

To receive funds we calculate the `<scriptPubKey>`:

```
HASH160 <redeemScriptHash> EQUAL
```

* Where `<redeemScriptHash>` is pre computed as follows: 

```
hash160(sha256(<redeemScript>))

```

* With the **new version** a `<zero>` is added to the `<redeemScriptHash>` : 

```
hash160(<zero> sha256(<redeemScript>))
```


### New Address

Also, this new powpeg type will have a new way of calculating the address, and this is how Segwit compatible addresses are computed:

```
Base-58 ( “05” + hash160 ( sha256 ( <segwitScript> ) ) )
```
Where:
- `<segwitScript>` is represented by: `“OP_0 <redeemScriptHash>”`
- `<redeemScriptHash>` by pre computing: `sha256 ( <redeemScript> )`
- and finally “05” is the *P2SH version byte*



### Spend output - Peg out

With this new implementation comes a new way of spending the output. Although it does not change much, it changes: the *transaction id computation*, the *scriptSig* field and the *segregated witness data*.
We can describe each of the changes as follow:

* **Transaction id computation**

SegWit can separate witness data from the block, thereby making it unavailable for modifications to change transaction IDs. 
The Segregated Witness protocol upgrade develops a sidechain for storing witness data separately from the main blockchain network. 
As a result, transaction IDs cannot be modified depending on witness data modifications.

* **scriptSig**

It will cointain the following content: 
```
    <zero> <sha256(redeemScript)>
```

* **Segregated witness data**

While segregated witness data will contain the redeeming data (signatures and flag for activating the ERP - [Emergency Recovery Protocol](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md)) and the actual redeem script (mentioned before as `<redeemScript>`)   

Example:
```
<zero> <Sig1> <Sig2>...<SigN> <FlagERP> <redeemScript>
```

### Summary
Putting it all together, the following is a list of the most important fields with their new values:

* witness:

```
<zero> <Sig1> <Sig2>...<SigN> <FlagERP> <redeemScript>
```

* scriptSig:    

```
<zero> <sha256(redeemScript)>
```

* scriptPubKey: 

```
HASH160 hash160(<zero> sha256(<redeemScript>)) EQUAL
```

### Migration to the new powpeg

During migration, both mechanisms (legacy and Segwit compatible) will be working at the same time. Legacy would be for the powpeg leaving and Segwit compatible for the incoming powpeg. 
All the funds (outputs) will be transferred from the current address to the new Segwit compatible address before mentioned (New Address section).

But once the migration is completed, only Segwit compatible will be the accepted address and the unique mechanism to support the peg outs.
The migration process description can be [found here (RSKIP186)](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP186.md).


## Rationale
Segwit stands for segregated witness, and it is an upgrade to the Bitcoin consensus rules and network protocol proposed and implemented as a BIP – 91 (https://github.com/bitcoin/bips/blob/master/bip-0091.mediawiki) soft-fork that was activated on Bitcoin’s mainnet in 2017.

Before segwit’s introduction, every input in a transaction was followed by the witness data that unlocked it. The witness data was embedded in the transaction as part of each input. It is called segregated witness because it implies separating the signature or unlocking script of a specific output. The witness data is moved outside the transaction.
In Segwit v0, the Block Size concept is replaced by Block Weight. Block Size is measured in Bytes and Block Weight is measured in Weight Units (WU). The maximum weight of a 1 MB block is 4000 WU. In calculating the weight of a transaction, a non-witness 1 byte weighs 4 WU and a witness byte weighs 1 WU.

We first analyze two other protocols Frost and MuSig2 (part of Segwit v1: [BIP 340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki), [BIP 341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki), and [BIP 342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)). Although some advantage is obtained in relation to performance, this is not enough to counteract the greater difficulty of implementation that they need. This is why we consider that implementing Segwit v0 is the best option in the medium/short term.
After doing a deep and detailed analysis of these protocols involved we summarized in a table a comparison between the different protocols in terms of complexity, fee costs and robustness.

| Topics            | FROST      | Musig2    | Segwit    |P2SH (current)  |
| ----------------- |:----------:| ---------:| ---------:| --------------:|
|PowHSM Complexity  | High       | Low       | Low       | -              |
|Fee Cost type      | Fixed      | Variable  | Variable  | Variable       |
|INPUT Fee Cost (vbytes) 7/13| 16| 120       | 238.75    | 955            |
|INPUT Fee Cost reductions (%)7/13| -98%    | -87%      | -75%      | -              |
|TX Fee Cost (vbytes) (1)| 106   | 210       | 328.75    | 1045           |
|TX Fee Cost reductions (%) (1)| -89%       | -78%      | -66%      | -              |
|Verification Complexity|Medium/Low| High    | Low       | Low            |
|Robustness         | Medium/Low | Low       | High      | High           |
|Implementation time| Long term  |Medium term|Short/Medium-term| -        |
|Threshold N limit (2)|Unlimited (3)| 34     | 67        | 15             |
 
**Observations*
1. *A TX is composed of 1 input and 2 outputs.*
2. *This is the higher N, independent of M. We are considering M as the half of N, which is the worst-case scenario. If M is more or less than half of N, the total starts to decrease.*
3. *FROST unlimited threshold limits it just in theory and what we can represent through aggregated signatures, practically we would have restrictions at the Bridge complexity level.*


## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated.


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

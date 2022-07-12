---
rskip: 225
title: Emergency Multisig public keys
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-04
---
# Emergency Multisig public keys


|RSKIP          | 225 |
| :------------ |:-------------|
|**Title**      |Emergency Multisig public keys|
|**Created**    |APRIL-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |
|**discussions-to**     | |


# **Abstract**

This document proposes that the signatories of the [Emergency Multisig](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md) must publish their public keys in xpub format (BIP32) in one or more websites or social media platforms that are known to be managed by the signatory.


## Motivation

In the Iris network upgrade, a time-locked emergency multi signature will be activated. This will take effect immediately for all new peg-ins and change addresses created in peg-outs.
Members of the RSK community must propose signatory candidates to hold one of the emergency backup keys. Also the community must finally selected a subset from the candidates, based on reputation, technical capabilities and incentive alignment with Bitcoin and RSK. However, after the decision is made, the community must also be able to:

* known for sure if the candidate accepts the responsibility
* authenticate that the public keys that will be embedded into the RSK reference code belong to the actual signatories selected.


## Specification

Each elected signatory must derive a public key from an existing or new seed. The derivation path should be hardened to avoid cross-path vulnerabilities. The recommended path is m/45'. The signatory must have a seed backup or the private key should exportable from the hardware wallet protecting it. 
An emergency is defined as the expiration of the emergency multisignature time-lock that protects the funds in the RSK powpeg. This RSKIP does not describe how an emergency will be detected by or notified to the community, nor the exact procedure for recovery. Each participating signatory will freely and independently attest and judge the emergency situation, and coordinate with the community for the retrieval and transfer of funds. No monetary compensation to signatories is expected for this task.


The most probably outcome in case of emergency is that the signatory private key, together with remaining multisignature public keys, will be entered in an open source tool for the execution of a multi-party protocol for the creation of a Bitcoin transaction to transfer the locked funds to a new RSK secure powpeg address. A candidate tool is an Electrum wallet. 

A signatory must publish in at least one social media platforms or website controlled by signatory the following message:

"My emergency multisig xpub for RSK is <xpub>" (without quotes)

xpub must be an xpub string in ASCII as specified in BIP32.

In case this is published in the signatory website, the message text or a downloadable file must be accessible by browser navigation.

Optionally, the signatory may use a pre-established method that associates owned social media accounts with public signature keys, such as keybase. The method used for proving the association of the RSK xpub with identity must be disclosed to the community.

From time to time the signatories may be requested to provide a proof of possession of the private key associated with the published public key as a signature of the following ASCII message:

"I have access to the RSK emergency multisig private key at Bitcoin block <blockhash>", where blockhash is a recent hash of a Bitcoin block, preferiably not older than one week and not earlier than one day. 

The message must be wrapped in the Bitcoin signing format:
"\x18Bitcoin Signed Message:\n" + compactSizeEncoding( len(message) ) <message>

The request will be announced with at least one month in advance. The request may be initially solicited by the RSK community but may later be generated autonomously by the RSK consensus. The requests should not be more frequent than once per year. The response would be published.

## Test Cases

This is an example of a valid message to be published by a signatory:

"My emergency multisig xpub for RSK is xpub661MyMwAqRbcF2GDHT2QfBsuqvL8DhJHYpW8jDjGr5kq8VueYfACVBRbPhmbwVzf29Q2xXv4t1GtUs3uiNY7EiPgPbndmCQiYMJ1YQQMF2P"



## Security Considerations

While PKI infrastructure could provide stronger guarantee of ownership of the public key, no such widespread infrastructure exists. Therefore we do not mandate the use of any specific PKI.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

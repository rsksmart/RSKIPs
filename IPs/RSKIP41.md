---
rskip: 41
title: Extended Bitcoin Bridge Transactions
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-04-25
---

# Extended Bitcoin Bridge Transactions

|RSKIP          |41           |
| :------------ |:-------------|
|**Title**      |Extended Bitcoin Bridge Transactions|
|**Created**    |25-APR-2017 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft | 

# **Abstract**

This RSKIP proposes different ways funds can be sent to RSK from Bitcoin, and how a Bitcoin transaction can remotely execute a method in an RSK contract.

## Discussion

The FTWP (see [RSKIP40]) functionality can be extended by the use of an EET (Extended Exit Transaction) to create a new contract, and execute code. The code will receive payments in SBTC and will be able to perform whatever action is needed, and then send back funds to a Bitcoin address by commanding the Bridge.

## EET Transactions

The easiest way to redeem SBTC in the Rootstock blockchain is by creating an EET transaction. An EET transaction has the following properties:

* It has only two outputs

* The first output represents the amount of BTC to transfer to the Rootstock blockchain [the Amount field]. The output address must be the Federators Exit Address.

* The second output only specifies data. The amount must be zero, and the scriptPub must contain an OP_RETURN opcode and OP_PUSHDATA1 opcode, specifying the size of the data following, followed by payload bytes. The payload contains the following fields, RLP encoded:

    * EET version (currently 0)

    * The destination address in Rootstock (20 bytes) [DestinationAddress field]

    * The redeem fee for automatic redeem (RegisterBtcLock message). This can be zero to indicate no automatic redeem is necessary [ RskRedeemFee field]

    * Optional if the destination address is a contract:

        * (0, The maximum gasPrice to pay (1 to 8 bytes)) [gasPrice field] 

        * (1, The maximum amount of gas to consume (1 to 8 bytes)) [gasAmount field]

        * (2, A Rootstock refund address, to transfer unused fees, if any (20 bytes) [ RskRefundAddress field]

        * (3, hold time in blocks) [HoldBlocks field]

        * (4, A Bitcoin refund address, to transfer the remaining if no execution occurred, if any (20 bytes)) [ BtcRefundAddress field]

The payload size must not exceed 80 bytes (including RLP bytes). If at the time the EET transaction is redeemed in the Rootstock blockchain the network gasPrice is over the specified gasPrice, the transaction will be held for HoldBlock blocks, waiting for manual triggering. No gasPrice re-check will be performed. If HoldBlocks is zero, the transaction will not be held. 

If no manual triggering occurs and if a RskRefundAddress is defined, all the funds will be transferred to the RskRefundAddress in Rootstock.

If no manual triggering occurs and no RskRefundAddress is defined, and a BtcRefundAddress is defined, then all the funds will be transferred back from RSK to Bitcoin. Note that any fees required to release BTC funds will be deducted from the Amount field. If the remaining is zero, no refund will take place.

If the transaction is executed automatically then the unused fees (gasAmount*gasPrice) will be transferred to the  RskRefundAddress (after deducting the standard transfer fee).

If the transaction is manually triggered, the all pre-specified fees (gasAmount*gasPrice) will be transferred to the RskRefundAddress ((after deducting the standard transfer fee).

## PXT Transactions

A Proxy Transaction (PXT) is a Bitcoin transaction that moves bitcoins to a Bitcoin address which represents a Rootstock address. Without the need for the Federation, a user can create a proxy Bitcoin address that maps to a Rootstock address. This is done by creating a script with the following format: OP_PUSH <RskAddress> (or payload of EET) OP_DROP <Federation script> and then hashing it (UserScriptHash). The funds must be sent to a P2SH address of this UserScriptHash.

## SAK (Same Account Key) Transaction

Send the funds to the Federation address. The funds are transferred in RSK to the account related to the same public key specified in the scriptSig of the first input of the transaction. 

## OPK (Owned-Private-Key) Transactions

Similar to SAK but the user has to redeem the money in Rootstock using the private key of the first input of the transaction. This is done by signing a redeem message "RskRedeem address", with the private key related to the first input and sending this message to the Bridge using the method ReleaseRTS(address, signature). This will be useful over a SAK only when we implement other Public Key architectures (we support now only ECDSA secp256k1)

The first input must be a P2PKH or P2SH and the script must be a standard P2PKH script.

[RSKIP40]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP40.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
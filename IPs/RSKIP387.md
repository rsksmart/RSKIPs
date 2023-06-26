---
rskip: 387
title: Support for Bridging Ordinals
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2023-06-19
---
# Support for Bridging Ordinals

|RSKIP          |387           |
| :------------ |:-------------|
|**Title**      |Support for Bridging Ordinals|
|**Created**    |19-MAY-2023 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This is a proposal for making Bitcoin ordinals and inscriptions available in Rootstock using the current Powpeg bridge. Once ordinals or inscriptions have representations in Rootstock,  they can be traded. Later they can be transferred back to any Bitcoin address.


# **Motivation**

Inscriptions are images, videos, recordings, or text that are stored in the Bitcoin blockchain, can be owned by a user and the ownership can be transferred to another user using standard Bitcoin transactions. We can think that every satoshi ever minted in Bitcoin is a separate “virtual cheque” and has a unique serial number. Then those serial numbers are called Ordinals. Inscriptions are linked to specific ordinals, and move with those ordinals with every transaction that involves those virtual cheques. There has been a rising interest on inscriptions since their introduction.

The Rootstock bridge is designed to transfer fungible tokens, but not non-fungible elements. To transfer non-fungible elements, the bridge must segregate them, and avoid mixing them with other elements. The bridge is not prepared to do this now, and sending an UTXO containing an ordinal to the Bridge will cause the ordinal to be commingled with other UTXOs, and its ownership would be inadvertently transferred to a user performing a peg-out at a later time.

# **Specification**

This is a proposal for making Bitcoin ordinals and inscriptions available in Rootstock using the current Powpeg bridge. Once ordinals or inscriptions have representations in Rootstock,  they can be traded as ONFTs (Ordinal NFTs). At any moment, the owner of the ONFTs can redeem them in Bitcoin and transfer them to any Bitcoin address.

Internally, the bridge does not know anything about NFTs or EIP-721. The bridge will only provide the service of the temporal custody of isolated UTXOs, what we’ll call parking. The bridge will provide one additional service: changing the owner of a parked UTXO. 

Using external user-provided smart-contracts, those ordinals can be turned into EIP-721-compatible NFTs. To enable this, the community needs to define a standard URI for the lookup of ordinals in the Bitcoin blockchain, so that NFTs marketplaces can retrieve the associated inscription. That standard is out of scope of this document.

To avoid attackers spamming the Bridge with ordinals, there are upfront and recurrent costs for parking ordinals within RSK. Ordinals whose annual fee has been unpaid will be assigned to the first user who pays its parking fees. In the future, a blinded auction process could be added.

The pegnatories will not participate in the peg-in of ordinals. We assume that the user has already rBTC in Rootstock for performing and paying for the registration of the ordinal. If the user doesn't have rBTC, it can use the flyover protocol to register the ordinal directly using Bitcoin transactions by negotiating with a Flyover liquidity provider. 
 
The Bridge will provide these new methods:

* **getFederationOrdinalsAddress**(): get the current address to transfer ordinals to Rootstock.
* **registerBtcTransactionWithOrdinals**(): registers all the ordinals in a transaction. The ordinals are identified by being sent to the ordinals federation address.
* **registerBtcTransactionWithOrdinalsDMFA**(): registers all ordinals in a transaction. Ordinals are identified by being sent to unique federation addresses mapped to a destination owner.
* **getOrdinalOwner**(): Returns the RSK owner of an ordinal. 
* **transferOrdinal**(): sets a new RSK owner for an ordinal. Only the owner can transfer ownership of  an ordinal.
* **releaseOrdinal**(): transfers the ordinal back to a Bitcoin user of the Bitcoin blockchain. Only the owner can release an ordinal.
* **getOrdinalsAnnualParkingFee**(): returns the fee that ordinals must pay annually for being parked. The amount is given in rBTC weis.
* **payParkingExtensionFee**(): Pays the parking fee of an ordinal to be extended one year. 
* **getOrdinalParkingExtensionFee**(): Returns how much the user needs to pay to extend the parking period for one year from the current date.
* **getOrdinalParkingExpirationDate**(): Returns the date the current parking period ends.

All ordinals that are parket are identified by a transaction ID and an output index corresponding to the Bitcoin parking transaction. This allows to easily track parked ordinals to Bitcoin transactions and recover the ordinal metadata. 
Starting from an activation block (TBD) these methods are activated.
 

## Peg-in of an ordinal

The bridge has a special address to send ordinals or inscriptions: the ordFedAddress. 

To obtain the ordFedAddress, the following bridge method must be called:

`function getFederationOrdinalsAddress() returns (string)`

The address is constructed differently from the BTC peg-in address. The construction is described in the section “Main Ordinals Address Derivation”

To register an ordinal, the following method can be called:

`function registerBtcTransactionWithOrdinals(bytes btcTxSerialized, int height, bytes pmtSerialized) payable returns (int executionStatus)`

This method will register the UTXOs as independent entities. Each UTXO registered will be assigned a serial number (or index) starting from 0 on each transaction.  This serial number matches the UTXO output index. If there are outputs that are not transfers to the federation ordinal address, then those serial numbers will be skipped.

The UTXOs will be owned in RSK by the same private key that controls the first input of the transaction or owned by a different RSK address, if there is an `OP_RETURN` output containing it, as specified by RSKIP-170.

The method call must transfer the full payment of the annual parking cost of all the registered ordinals, as if `payParkingExtensionFee()` would have been called for each ordinal. If the amount paid is less than the amount required, the transaction will fail and the amount paid will be reimbursed to the calling address. If the amount is greated than the required, the excess will be reimbursed.

Ordinals require only 6 Bitcoin confirmations for peg-in (1 hour on average).

The method returns an integer value indicating the execution status:

  `< 0 : Error `

   `0 : Call method before fork activation `

   `> 0 : Success. Number of transaction outputs in the transaction btcTxSerialized`


#### Error codes

`ORD_UNPROCESSABLE_TX_INVALID_SENDER_ERROR_CODE = -301`

`ORD_UNPROCESSABLE_TX_ALREADY_PROCESSED_ERROR_CODE = -302`

`ORD_UNPROCESSABLE_TX_VALIDATIONS_ERROR = -303`

`ORD_UNPROCESSABLE_TX_VALUE_TOO_HIGH_ERROR = -305`

`ORD_BRIDGE_GENERIC_ERROR = -900`

## Restrictions on peg-in amounts

Registering ordinal UTXOs that contains an amount of BTC over 0.01 BTC is forbidden. In the case it happens, the Bridge will migrate the bitcoins contained in that UTXO to the normal Powpeg address, and consider tthe funds a donation to the Rootstock sidechain. The control of the associated ordinal will be lost. The remaining UTXOs of the transaction can be registered.

An UTXO with an amount lower than 600 satoshis won't be accepted either, because its value is too close to Bitcoin's current dust limit, and therefore the Bridge cannot perform a peg-out using standard Bitcoin transactions. If an ordinal UTXO specifying a lower value is sent to the ordinal Powpeg address, the ordinal will be lost. 

## Peg-in using Destination-Mapped Federation Addresses

Similar to the flyover protocol, it’s possible to create a Bitcoin address that  is controlled by the Powpeg but when sent an ordinal it automatically assigns its ownership to a specific Rootstock address or contract. We call this type of addresses **destination-mapped federation addresses** (DMFA). DMFA addresses can be used to transfer ordinals to Rootstock when the source Bitcoin wallet doesn't allow to add an `OP_RETURN` output to the Bitcoin transaction, nor controlling the private key associated with the first input.

To use this peg-in mechnism, the user needs to create a special unique DMFA reception address for the ordinal. This DMFA address can be reused for many ordinals in subsequent transactions. One the Bitcoin ordinal transfer transaction is confirmed with at least 6 confirmations, the transaction can be registered with the following method:


`function registerBtcTransactionWithOrdinalsDMFA(
   bytes btcTxSerialized,
   uint32 height,
   bytes pmtSerialized,
   address rskOwner,bytes256 extraData) payable returns (int executionStatus)`

The rskOwner field specifies the owner of the ordinal. The extraData field allows transferring the ordinal directly to a specially designed EIP-721 contract. In a following section we specify how the DMFA address is constructed. Anyone can call registerBtcTransactionWithOrdinalsDMFA() , but the ownership will always be transferred to the rskOnwer associated with the dataHash field computed by the bridge when called.

The return code is interpreted similar to the registerBtcTransactionWithOrdinals() method.

## Changing the Ownership of an Ordinal

To query who owns an ordinal, the following method is added:

`function getOrdinalOwner(uint256 tId, int index) returns (address)`

The method will return 0x00... (the zero 20-byte address) if the ordinal does not exist.

To change ownership of an ordinal, the following method must be called by the owner:

`function transferOrdinal(uint256 tId, int index, address newOwner)`

The method does nothing if the ordinal does not exist.

## Ordinal Peg-outs

To send an ordinal, the following method must be called:

`function releaseOrdinal(uint256 txId, uint256 index, bytes btcScriptPubkey) returns (uint32 err)`

This method will queue the release of an ordinal, sending it to the Bitcoin address associated with the btcScriptPubkey specified. The caller must be the owner of the ordinal. Every 1 hour the ordinals that are queued to be released will be processed. The bridge may attempt to create a single transaction containing all ordinal UTXOs, and all ordinal outputs, plus a single additional input used for fee payments, and a single additional output used for change. Ordinal peg-outs will not be mixed with BTC peg-outs.

Ordinals can be sent to any segwit or Pay-to-Script-Hash (P2SH) Bitcoin address by directly specifying the btcScriptPubkey. The Bridge must perform validations on the btcScriptPubkey to make sure it corresponds to a standard segwit address or P2SH. If the scriptPubKey is invalid, the release will fail with an error of 2. If the ordinal does not exists, the function will return 1. the On success, the function returns 0.

For a description of valid segwit scriptPubKey, refer to the segwit [BIP-141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) or to the function [Solver()](https://github.com/bitcoin/bitcoin/blob/153a6882f42fff3fdc63bf770d4c86a62c46c448/src/script/standard.cpp#L168) in Bitcoin Core's source code. 

Ordinals require 120 Rootstock confirmations (about 1 hour) for HSMs to be able to sign releases. HSMs must also require a cumulative proof of work equivalent to 120 Rootstock blocks.

## Paying Parking Fees
Each ordinal parked stores the date when the last annual parking fee was paid. Initially this date corresponds to the date of parking. The parking expires one year later. If the parking expires and the fee has not been paid, anyone will be able to claim the ordinal by paying its parking fee. If a bridge migration event occurs, all parked ordinals that have their fee unpaid will not be migrated, and they will be lost forever.

To know how much must be paid to park an ordinal, the Bridge provides the following method: 

`function getOrdinalsAnnualParkingFee() returns (uint)`

This method returns the fee that ordinals must pay annually for being parked. The amount is given in rBTC weis. The annual fee amount is defined in the section "Annual Parking Fees".

To pay the parking fee of an ordinal, the Bridge provides the following method:

`function payParkingExtensionFee(uint256 tId, int index)`

This method extends the parking time of the ordinal to a year starting from the current date.
 
To know exactly how much you need to pay to extend the parking time, the Bridge provides the following method:
 
`function getOrdinalParkingExtensionFee(uint256 tId, int index) returns(uint256)`

This method returns how much the user needs to pay to extend the parking period for one year from the current date. The value is expressed in weis.

To know when an ordinal parking period expires, the Bridge provides this method:

`function getOrdinalParkingExpirationDate(uint256 tId, int index) returns (uint64)`


When paying the parking extension fee, the user should transfer the amount returned by getOrdinalParkingExtensionFee(). Any excess amount will be reimbursed to the sender by direct balance update. Note that if the user pays the parking fee several times in the same block, only the first payment will be processed. The remaining will be fully reimbursed. The date is expressed in Unix time format.

## Annual Parking Fees

When the federation migrates, all ordinals need to migrate along the pegged bitcoins. This process may be expensive in terms of transaction fees. 
Therefore the annual cost of parking is set to 2 times the cost of a bridge BTC spending transaction with 2 inputs and 1 output. Also to park an ordinal for the first time the user needs to pay this amount upfront suring registration. 
Because the annual cost is chosen based on the peg-out cost of a transaction, it varies with the fee per kilobyte configured in the Bridge.


## Migration of Ordinals


To migrate the ordinals, the bridge will issue migration transactions. The migration transactions will consume many ordinal inputs, and create many ordinal outputs. For each ordinal parked, there will be an input and an output withing the same migration transaction and with similar indexes. Also the value of the output will match exactly the corresponding input. This ensures that ordinals are  preserved. The may be more inputs and change outputs to handle the payment of Bitcoin transaction fees. Those additional inputs and outputs will be added after the ordinal indexes. 

Since currently there is a maximum number of inputs (established by the HSM) and a maximum transaction size (established by Bitcoin standard rules), the Brtidge may issue more than one transaction for migrating ordinals.


## Internals

In the following sections we specify how the Bridge must internally handle ordinals.

### New Bridge Storage Address Mappings
The utility internal function `getOrdinalIndex(bytes btcTxHash, uint256 id)` returns the sequential index of the ordinal. The internal function `getOrdinalIndexKey(uint index)` returns a storage address used to lookup the index. The key is computed as the keccak256 hash of the following string:

`"OrdinalIndex-" + btcTxHash.toHexString() + id.toHexString()`

Where "+" represents string concatenation and the toHexString() method returns the ASCII hexadecimal representation of the data without the "0x" prefix, using lowercase letters, and without trimming leading zeros . The btcTxHash hexadecimal string is always 64 bytes in length. The id hexadecimal string is always 64 bytes in length. No separator is used between these two hexadecimal strings.
If the storage cell is empty, then the ordinal does not exist. This means that the address 0x00.. (20-byte all zeros) cannot own any ordinal.
The cells contain an unsigned index. To lookup the ordinal metadata, a second storage key must be accessed. A parked ordinals is stored in a storage cels with an address that corresponds to the keccak256 hash of:

`"Ordinal-" + uint64(index).toString() `

Each cell contains the following fields:

* **btcTxHash**: hash of original parking transaction
* **outIndex**: output index of the ordinal
* **sats**: the BTC amount stored (which corresponds to the length of ordinals in sats).
* **parkDate**: the parking date (updated when the parking is extended)
* **owner**: the RSK address of the ordinal owner


The methods that change the owner and parking date of the ordinal will modify the content of the index cell.

The storage cell with an address equal to the hash of "OrdinalCount" stores the number of ordinals parked. Initially it is zero.

### Powpeg Public Keys for Ordinals 

To avoid type confusion attacks where the HSMs sign the release of ordinals as if they were BTCs or vice-versa, the Powpeg will use a new set of private and public keys that are used to control only  ordinals.
A new public key with index <TBD> will be added on pegnatory registration, and returned with `getFederatorBtcPublicKey()`.


### Single-use Ordinals Address Derivation
The Powpeg ordinals address derivation starts with the creation of a custom redeem script that pushes the verData field. The verData field is defines as:

* **version**: specifies the derivation version (1 byte) 
* **dataHash**: specifies the hash digest of the owner and the extraData

The field dataHash is the keccack256 hash digest of the following concatenated fields:
 
* **owner**: ordinal owner address (a 20 byte RSK address)
* **extraData**: a user-defined 32-byte data.

Therefore verData consist of a 33-byte binary chunk. The verData field is pushed and immediately dropped from the stack using the `OP_DROP` opcode, so the pushed data does not alter the processing of the redeem script.

The redeem script for Powpeg-derived addresses is a modification of the standard Powpeg redeem script. We show how to build such scripts, but we omit the emergency multisig script path for clarity of this document (in the code it must be present). The whole script must be considered when computing the address. The new custom redeem script has the following format:

```scriptPubKey: OP_HASH160 <redeemScriptHash> OP_EQUAL
scriptSig: OP_0 <signatures> <redeemScript>
redeemScript: <verData> OP_DROP OP_M <publicKeys> OP_N OP_CHECKMULTISIG
```

Note that the public keys are the ordinal public keys, not the public keys that protect the BTCs.

### Main Ordinals Address Derivation

The main ordinal address derivation uses the ordinal Powpeg public keys and additionally pushes a single byte 0 as a version specification.

Format:

`redeemScript: PUSH_0 OP_DROP OP_M <publicKeys> OP_N OP_CHECKMULTISIG`

### New Release Request Event

Since the `release_requested` triggers a 4000 block confirmation delay enforced by the Powpeg node and the HSM, a different event is required to request HSM signatures to signa transaction that release ordinals. A new event `release_requested2` is added, containing an additional argument, which is the public key index. The HSM will enforce a 4000 block confirmations or 120 block confirmations depending on this argument. If the argument is out of bounds, the Powpeg node and the HSM must do nothing.  


## Gas costs

The folllwing list provide the gas costs of the new methods:

* **getFederationOrdinalsAddress**(): 500
* **registerBtcTransactionWithOrdinals**(): 22000
* **registerBtcTransactionWithOrdinalsDMFA**(): 25000
* **getOrdinalOwner**(): 800 
* **transferOrdinal**(): 6000
* **releaseOrdinal**(): 23000
* **getOrdinalsAnnualParkingFee**(): 200
* **payParkingExtensionFee**(): 6000
* **getOrdinalParkingExtensionFee**(): 800
* **getOrdinalParkingExpirationDate**(): 800

# Rationale

## Parking Extension Costs

The extension cost returned by `getOrdinalParkingExtensionFee()` will generally be lower than the value returned bu `getOrdinalsAnnualParkingFee()`. As an example, if currently the expiration of an ordinal is 6 months in the future, then the value returned will be half of the annual cost, because 6 months have already been paid and only 6 months remain to be paid. Parking fees cannot be paid in advance for more than a year.


## Migration Costs

To simplify the Bridge code, this proposal uses a full UTXO to store each ordinal. It does not attempt to create UTXOs containing multiple ordinals.
In the future, the bridge want to consolidate many ordinals into a single UTXO to lower migration costs. The parking fee is enough to pay for this consolidation. 

Again, while all ordinals could be compressed into a single UTXO with several ordinal ranges, we maintain separate UTXO to reduce the complexity of the bridge and allow parallel releases of ordinals.

## Using ordinals as EIP-721 NFTs

It is possble to wrap parked ordinals in EIP-721 contracts to trade them on standard NFT marketplaces. This requires minting the associated NFT (what in this proposal we call an "ONFT"). When minting the ONFT, the EIP-721 contract will associate a new identifier to the ordinal (the tokenId). The tokenId should map uniquely to the ordinal (tID, index). One way of doing so is by making the tokenId equal to the hash of these two fields, and using a mapping within the contract.

The NFT marketplace will need to query the EIP-721 for metadata to find the image associated with the ONFT. The metadata will point to a Bitcoin transaction and output index. A simple method to do it is to let the ERC-721 contract retrieve the (tID, index) pair associated with a tokenId, and let the client (i.e. the markeplace app) perform the necessary steps to retrieve the image/text/sound related to the ordinal by looking up the transaction with a Bitcoin client that supports the ordinal wallet.

An alternative is that the user provides a “chain of ownership” transfer to the Bridge contract (maybe compressed as a ZK proof). However this may be very expensive to send in terms of calldata, or expensive to verify by the ordinals proxy contract.

To allow Roostock NFT marketplaces to recognize ONFTs, the system should deploy EIP-721 contracts owning the ordinals. Let's call each of these contracts an ONFTC. All ONFTCs must be trusted to correctly handle the ONFTs because a malicious ONFTC could expose an ONFT it doesn't really own. Therefore, either all ONFTCs are created using a single trusted factory contract or a single trusted NFTC shhould handle all ONFTs. We believe that having a single ONFTC contract (that we call ONFTMaster) is both simpler and cheaper.

To use the ONFTMaster, the user builds the dataHash field with the following information:

* owner: the address of ONFTMaster
* extraData: the Rootstock address of the real owner.

Then the user uses this dataHash value to build a distinct parking address.
 
To register the ordinal, the user calls a registration method on the ONFTMaster contract. We present such method prototype as an example: 

`function registerOrdinal(uint256 tId, int index, address rootstockOwner).`

The registerOrdinal() method in turns calls the Bridge registerBtcTransactionWithOrdinalsDMFA() method, specifying the same transaction ID and index, its own Rootstock address as the owner and the rootstockOwner field as the extraData field.

Note that when using an ONFTMaster contract, the ONFT ownership transfers between users will be handled by updating an internal mapping within the ONFTMaster contract, and not by calling the transferOrdinal() method of the bridge. Therefore, parking fees must also be paid throught new methods provided by the ONFTMaster contract.

## Moving Rootstock-native NFTs into Bitcoin Inscriptions

This proposal is about moving ordinals into Rootstock and back into Bitcoin, but there exists another use case of moving NFTs that were originally created in Rootstock into Bitcoin, performing inscriptions there.

This requires storing into the NFT contract the image/sound/blob related to the NFT, and letting the Powpeg use a bridge UTXO to perform the inscription. Performing the inscription requires doing two transactions. This may be a more complex procedure. Also, users may question the traceability of these inscriptions, because it’s difficult for a bitcoiner that has only access to the Bitcoin blockchain to know if a certain inscription was performed by the Powpeg, or by any other entity.

## Involvement of the Pegnatories

Pegnatories are not involved in the parking of ordinals. The reason is that the ordinal registration process must be paid by the user, and the ordinal parking transaction doesn't pay for the registration. To be able to pay for the registration, the ordinal parking transaction would need to either have a second output where bitcoins are transferred to the Powpeg or it should embedd the paymnet inside the ordinal UTXO. 
Using a second output makes the registration algorithm more complex, and forces the bridge to deal to refunds in case the amount paid is unecessary high and reclaims in case it's too low. Also the output amount would be below the peg-in minimum, preventing the Bridge to consume it efficiently, leading to UTXO fragmentation, and loss of funds. 
Embedding a pyayment in the ordinal UTXO itself has the same two drawbacks mentioned for separate UTXO, plus the problem of carefully extracting the rBTC without accidentaly sending the ordinal.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

# Security Considerations

We believe that while some ordinals and inscriptions could be highly valued, due to the low cost of creating spammy inscriptions, in the long term most ordinals will be low valued. The cost of an ordinal may become much lower than the BTC amount of an average bitcoin peg-in, and even lower than the cost of a UTXO consolidation. This led the design of the bridge towards two decisions:  first, ordinals should pay a rent to avoid spam, and second, ordinals can be carefully isolated with their own security model. 
While the double-spend of an ordinal during peg-in can cause distress to its owner, it does not affect any other user of Roostock, not any other ONFT. There can’t be a run to the peg if an ordinal is lost, while a mismatch of BTC collateral in Bitcoin and rBTC  issued in Roostock can be the cause of such an event. 

Since ordinals can be tracked independently in separate UTXOs, the double-spend of an ordinal does not affect the peg funds, nor the other ordinals. Therefore, we can lower the number of confirmations required for an ordinal peg-in. We proposed that ordinals require 6 Bitcoin confirmations for peg-in (about 1 hour), and 120 confirmations (about 1 hour) for peg-out. 

The ordinal spam risk on peg-in has been identified and analyzed. No security vulnerability was found due to the fact that the users must register the ordinals themselves.
 
Two distinct desposit addresses are used to avoid confusion between the BTC and ordinals deposited, and each address is based on a different public/private key pair.

A different `OP_DROP` scheme is used also to avoid confusion between ordinals and flyover BTC peg-ins.

A user releasing an ordinal can specify any scriptPub key, including the Powpeg address, or a flyover address derived from the Powpeg address. It's important that the Bridge does not assume that a transaction originating from the Bridge is a specific type of transaction (such as a migration transaction). For example, a user may register and then release thousands of ordinals to the BTC Powpeg address in an attempt to confuse the Bridge into believing that these releases are bitcoin migrations instead. This potential vulnerability must be ruled out before activating this proposal.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

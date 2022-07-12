---
rskip: 176
title: Programmable Peg-in Addresses for faster peg-ins
description: 
status: Adopted
purpose: Usa
author: SDL (@sergiodemianlerner), GM <guido@iovlabs.org>
layer: Core
complexity: 2
created: 2020-09-15
---
# Programmable Peg-in Addresses for faster peg-ins
|RSKIP          | 176 |
| :------------ |:-------------|
|**Title**      |Programmable Peg-in Addresses for faster peg-ins |
|**Created**    |15-SEP-20 |
|**Author**     |SDL, GM |
|**Purpose**    |USA |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes an RSK protocol change called **Programmable Peg-in Addresses (PPA)** (formerly known as "fast peg-in") that enables to send bitcoins to RSK over *Powpeg-derived* addresses, where each derived address has a peg-in execution code cryptographically linked into it. To receive the RBTC of a PPA peg-in, a smart contract must send to the Bridge the BTC transaction and Merkle proof, together with proof that the derived address is actually controlled by the Powpeg. If this proof is verified, the Bridge contract redeems the funds by executing the associated code. The code to be executed is succinctly specified as a contract method to be called, and the arguments to pass to the call. At the RSK consensus layer this protocol change only affects the Bridge contract. However, to make use of all features enabled by PPA, additional user-defined contracts that handle code execution must be deployed, and users and wallets must be aware of the new peg-in workflow.  One of the goals with this change is the deployment of the Flyover protocol, which is a specific peg-in execution engine that allows users to peg-in BTC to RSK in a fast way, where a third party advances the payment in RBTC to the user, while taking the bitcoin blockchain double-spend risk.

## Motivation

The RSK Bridge is presently slow, from the perspective of the end user. This is an intentional a security feature included by design, and not a bug. The bridge was designed so that users willing to accelerate the peg-in experience would either use centralized exchanges, wallets connected to token swaps APIs, or decentralized atomic swap protocols, while large liquidity providers would use the slow native bridge. In practice, users have been reluctant to use peg-in acceleration services for different reasons: high cost, lack of RBTC exchange pairs, lack of swap API integrations into wallets, lack of privacy, or a bad UX that comes from the need of pre-register an identity in many of these services. There is a clear need for fast, or even instantaneous, transfer of BTC on the Bitcoin network to RBTC on the RSK network.

A first solution is to accept peg-ins from Lightning network, using a technique called submarine swap, which doesn't require consensus changes. However, not all wallets have LN support. Another solution, described in this proposal, is to enable peg-ins to RSK with a only few Bitcoin confirmations, using third parties (called liquidity providers or LPs for short) to provide an advance of RBTCs in RSK to the users, while the LPs wait 100 bitcoin confirmations to be reimbursed. The number 100 is the number of blocks required by the RSK Bridge to release coins that were pegged-in. LPs can be assured they will be promptly reimbursed embedding execution scripts programmed right into the Powpeg addresses used for peg-in. Therefore, new peg-in addresses must be issued for each peg-in. The simplest way to describe an embedded script is to specify a destination contract (called Liquidity Bridge Contract or LBC) to handle the execution, and some arguments that the LBC knows how to parse. Since LPs are taking calculated risks that transactions on the Bitcoin network are not double-spent prior to attaining 100 block confirmations, they may charge a fee for providing the RBTC-advancement service. More complex LBCs may enable the creation of a free market between users who wants to peg-in faster and parties who provide the liquidity to advance the payment in a fully trust-minimized, open and censorship-resistant way. It is even possible to create an LBC that assigns addresses to smart-contracts on RSK, so users can send BTC directly to specific contracts on RSK just by specifying their unique Bitcoin address.

## Specification

The **Powpeg Redeem Script** (PRS, also called federation redeem script), is the the script referenced by script hash inside the Powpeg P2SH peg-in address. It contains the Powpeg multisig and emergency multisig spending paths. A **derived Powpeg address** is built by combining a value called **User Peg-in Identification** (UPI) with the PRS. The UPI consist of exactly 32 bytes provided by the user that performs the peg-in. The UPI in turn is the hash of some **Internal Derivation Arguments** (IDAs). The UPI is built as the keccack256 hash digest of the IDAs, serialized in a custom format. While the IDAs can be freely chosen by the user creating the peg-in transaction, each IDA has a special function recognized by the Bridge contract.  One of the IDAs is itself a hash of **External Derivation Arguments** (EDAs). The EDAs are referenced by the **derivationArgumentsHash** function argument described in the next section. The EDAs are referenced by a hash digest instead of explicitly because they are ignored by the Bridge. In the case of the Flyover protocol, both IDAs and EDAs represent an agreement signed between the Liquidity Provider and the user willing to perform the peg-in. The agreement contains the Terms of Service (ToS). Other protocols may use the EDAs for other purposes.

#### Bridge

A new method called **registerFastBridgeBtcTransaction**() is added to the Bridge contract will the following ABI signature:

```Solidity
    function registerFastBridgeBtcTransaction(
		bytes btcTxSerialized, 
		uint256 height,
		bytes pmtSerialized, 
		bytes32 derivationArgumentsHash, 
		bytes userRefundBtcAddress, 
		address liquidityBridgeContractAddress, 
		bytes liquidityProviderBtcAddress, 
		bool shouldTransferToContract) returns int executionStatus
```

**Parameters**:

- *bytes* **btcTxSerialized**: Serialized Bitcoin transaction (tx) following the standard Bitcoin serialization format.
- *uint256* **height**: BTC block number where the transaction is present. The value passed is truncated to a 32-bit unsigned integer. Caller must verify that the value passed is in the correct range.
- *bytes* **pmtSerialized**: Serialized partial Merkle tree (following bitcoinj structure) that proves that the transaction belongs to the indicated block.
- *bytes32* **derivationArgumentsHash**: A keccak265 hash of the external derivation arguments (EDAs).
- *bytes* **userRefundBtcAddress**: Binary encoding representing the user BTC refund address.
- *address* **liquidityBridgeContractAddress**: The RSK Liquidity Bridge Contract address.
- *bytes* **liquidityProviderBtcAddress**: Binary encoding representing  the Liquidity Provider BTC refund address.
- *bool* **shouldTransferToContract**: Status provided by the caller contract to know if should transfer value to the contract or not.

To compute the UPI, the IDAs are serialized as a concatenation of derivationArgumentsHash (32 bytes), userRefundBtcAddress (21 bytes), liquidityProviderBtcAddress (21 bytes), liquidityBridgeContractAddress (20 bytes). Each argument occupies a fixed number of bytes shown in brackets. Formally, this is:

`UPI = Keccak256(derivationArgumentsHash, userRefundBtcAddress, liquidityProviderBtcAddress , liquidityBridgeContractAddress)`

The total number of bytes hashed is 94. Note that the concatenation order is different from the argument passing order.

**Checks and Operations Performed by registerFastBridgeBtcTransaction**:

- The caller must be a contract (not an EOA). This check was found to be unecessary and may be removed by a future RSKIP.

- Compute **btcTxHash** as the Bitcoin hash of btcTxSerialized.

- Bridge receives a call from a contract (an alleged Liquidity Bridge Contract) and validates that the sender of that call is effectively the LBC address, that is received as parameter liquidityBridgeContractAddress.

- Performs the same validations done in **registerBtcTransaction** to determine if the BTC transaction is legitimate (e.g.: txAlreadyProcessed, confirmations, Partial Merkle Tree inclusion, etc).

- Computes the derived Powpeg address (**derivedPowpegAddress**) built using the UPI constructed from the passed arguments.  

- Computes the amount of funds sent by the transaction tx to the derivedPowpegAddress (**totalAmount**). If the amount is zero, aborts.

- Checks if the derivation hash was used already. If so, it refunds the bitcoin sender specified by the userRefundBtcAddress argument.

- Verifies that the caller matches the liquidityBridgeContractAddress. This address may not be a contract, but a External Owned Account (EOA).

- Depending on the shouldTransferToContract value received as parameter:

 - If **true**, Bridge will check if Locking Cap value is surpassed:
    - If not surpassed:
      - transfers the totalAmount to the Liquidity Bridge Contract
      - saves information related to the derivedPowpegAddress in the contract storage. The cell whose address is obtained by the **getStorageKeyForfastBridgeFederationInformation**(redeemScriptHash) method. The **redeemScriptHash** is the Bitcoin hash of the Powpeg P2SH Redeem Script associated with the derived Powpeg address (called **fastBridgeRedeemScriptHash**). The data to be stored is the RLP encoding of UPI and the Powpeg P2SH Redeem Script hash for the redeem script with the UPI removed (called **powpegRedeemScriptHash**). This last redeem script matches the standard generic Powpeg redeem script for any non-fast peg-in.
      - Marks the tuple (btcTxHash, UPI) in Bridge storage to that it cannot be reused. To do so the Bridge computes an unique storage address using the function **getStorageKeyForDerivationArgumentsHash**(btcTxHash, UPI), and stores the byte value 1 in the storage cell at that address. This enables the preventing the same transaction to be processed twice for the same UPI. This storage cell is never erased.
      - marks tx as processed.
      - stores all the UTXOs from tx that pay to the derived Powpeg address. The UTXOs are stored exactly as it is already done for regular peg-in UTXOs.

    - If surpassed, the Bridge refunds the Liquidity Provider at liquidityProviderBtcAddress.

 - If **false**:

   - Refunds bitcoin sender at address userRefundBtcAddress.

- Returns an integer value indicating the execution status:

  `< 0 : Error `

   `0 : Call method before fork activation `

   `> 0 : Value transfered to user`


#### Error codes

`FAST_BRIDGE_REFUNDED_USER_ERROR_CODE = -100`

`FAST_BRIDGE_REFUNDED_LP_ERROR_CODE = -200`

`FAST_BRIDGE_UNPROCESSABLE_TX_NOT_CONTRACT_ERROR_CODE = -300`

`FAST_BRIDGE_UNPROCESSABLE_TX_INVALID_SENDER_ERROR_CODE = -301`

`FAST_BRIDGE_UNPROCESSABLE_TX_ALREADY_PROCESSED_ERROR_CODE = -302`

`FAST_BRIDGE_UNPROCESSABLE_TX_VALIDATIONS_ERROR = -303`

`FAST_BRIDGE_UNPROCESSABLE_TX_VALUE_ZERO_ERROR = -304`

`FAST_BRIDGE_GENERIC_ERROR = -900`

#### New Bridge Storage Address Mappings

The utility function **getStorageKeyForDerivationArgumentsHash**(btcTxHash, UPI) computes a unique storage cell address as the keccack256 hash of the following string:

```
"fastBridgeHashUsedInBtcTx-" + btcTxHash.toHexString() + UPI.toHexString()
```

Where "+" represents string concatenation and the toHexString() method returns the ASCII hexadecimal representation of the data without the "0x" prefix, using lowercase letters, and without trimming leading zeros . The btcTxHash hexadecimal string is always 64 bytes in length. The UPI hexadecimal string is always 64 bytes in length. No separator is used between these two hexadecimal strings.

The utility function **getStorageKeyForfastBridgeFederationInformation**(redeemScriptHash) computes a unique storage cell address as the keccack256 hash of the following string:

```
"fastBridgeFederationInformation-" + redeemScriptHash.toHexString()
```

The redeemScriptHash hexadecimal string is always 64 bytes in length.

#### Limitations

- Bitcoin Transactions with a depth higher than 4320 will not be processed. Normally this depth will be enough to register a transaction that is a month old. An attempt to register a peg-in with higher depth will result in an error and the pegged-in funds will be lost.



#### Flow chart
![registerFastBridgeBtcTransaction() flow](RSKIP176/registerFastBridgeBtcTransactionFlow.png)


### Powpeg Address Derivation

The Powpeg address derivation starts with the creation of a custom redeem script that pushes the UPI (a 32 bytes hash). That data pushed is immediately dropped from the stack using the OP_DROP opcode, so the pushed data does not alter the processing of the redeem script. 

The redeem script for Powpeg-derived addresses is a modification of the standard Powpeg redeem script. In this section we show how to build such scripts, but we omit the emergency multisig script path for clarity.  The whole script must be considered when computing the address. A following RSKIP will specify the addition to the Bridge contract of a method to retrieve a custom redeem script based on a given UPI, as currently this process must be done by an ad-hoc library. The new custom redeem script has the following format:

```
scriptPubKey: OP_HASH160 <redeemScriptHash> OP_EQUAL
scriptSig: OP_0 <signatures> <redeemScript>
redeemScript: <derivationArgumentsHash> OP_DROP OP_M <publicKeys> OP_N OP_CHECKMULTISIG
```

This custom redeem script is executed in the same way to the standard script, allowing the same pegnatories to sign transactions consuming bitcoins from these ad-hoc addresses.


### Rationale
This RSKIP is an improvement over the [RSKIP 175](https://github.com/rsksmart/RSKIPs/blob/fast-bridge/IPs/RSKIP175.md "RSKIP 175"), but this alternative was chosen as the reference node does not support that the Bridge calls an external contracts, and adding this feature was risky. Therefore, the flow was inverted and the LBC first takes control, then calls the Bridge, and then regain control to finish the peg-in operation.

**Derivation Arguments**

The internal and external derivation arguments are values used as part of the agreement between the Liquidity Provider and the user. Some of these arguments are the RSK address where funds will be received, the BTC refund address, the Liquidity Bridge Contract address, the value to transfer, the amount of BTC blocks that the Liquidity Provider should wait before advancing the funds, the Liquidity Provider BTC refund address, the derived Federation address,.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

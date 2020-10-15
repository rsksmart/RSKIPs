# Fast BTC Bridge
|RSKIP          |           |
| :------------ |:-------------|
|**Title**      |Fast BTC Bridge |
|**Created**    |15-SEP-20 |
|**Author**     |GM |
|**Purpose**    |USa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Previous concepts

#####Derived Federation
The derived federation is the result of a process that will be carried out by creating a 32 bytes hash built from some provided derivation arguments and the active federation redeem script. The derivation arguments are values used to close the agreement between the Liquidity Provider and the user.

## Abstract

This RSKIP proposes a protocol change for the Bridge to provide a way for a contract to validate if a given BTC transaction sent funds to a derived Federation and to send those funds to that contract. This is part of a wider fast-bridge project that will allow a user to transfer BTC to RSK in a fast way, where a third party takes the risk to advance the payment in RBTC to the user.

## Motivation

Allowing users to transfer BTC to RSK immediately, without waiting for enforced peg-in confirmation blocks. This also generates a market between user who wants to peg-in faster and those whose who provide the liquidity to advance the payment and accept the risk of the reversal of the Bitcoin blockchain.

## Specification

The fast-bridge system connects users, Liquidity Providers (LP) and a certain Liquidity Bridge Contract (LBC).
This RSKIP specifies thehe specification of consensus changes to achieve the main goal of the use case without limiting the actual implementation of the fast-bridge system.

#### Bridge

The bridge needs to expose a new method called **registerFastBridgeBtcTransaction**().

#### registerFastBridgeBtcTransaction()

**ABI Signature**

        function registerFastBridgeBtcTransaction(bytes btcTxSerialized, uint256 height, bytes pmtSerialized, bytes32 derivationArgumentsHash, string userRefundBtcAddress, string LiquidityBridgeContractAddress, string LiquidityProviderBtcAddress, bool shouldTransferToContract) returns int executionStatus


**Derivation Arguments**

The derivation arguments are values used to close the agreement between the Liquidity Provider and the user. Some of these parameters are the RSK address where funds will be received, the BTC refund address, the Liquidity Bridge Contract address, the value to transfer, the amount of BTC blocks that the Liquidity Provider should wait before advancing the funds, the Liquidity Provider BTC refund address, the derived Federation address, and some other values that are domain of the Liquidity Provider.


**Parameters**:
- *bytes* btcTxSerialized: Serialized Bitcoin transaction following Bitcoin serialization
- *uint256* height: BTC block number where the transaction is present
- *bytes* pmtSerialized: Serialized partial merkle tree (following bitcoinj structure) that proves that the transaction belongs to the indicated block
- *bytes32* derivationArgumentsHash: A Keccak hash created from the derivation arguments
- *bytes* userRefundBtcAddress: Binary encoding representing the user BTC refund address
- *string* LiquidityBridgeContractAddress: String representing RSK Liquidity Bridge Contract address
- *bytes* LiquidityProviderBtcAddress: Binary encoding representing  the Liquidity Provider BTC refund address
- *bool* shouldTransferToContract: Status provided by the caller contract to know if should transfer value to the contract or not


**Flow**:

**Actions performed by the Bridge:**

- Bridge receives a call from a contract (an alleged Liquidity Bridge Contract) and validates that the sender of that call is effectively the LBC address, that is received as parameter

- Performs the same validations done in **registerBtcTransaction** to determine if the BTC transaction is legitimate (e.g.: txAlreadyProcessed, confirmations, PMT, etc)

- Verifies tx is sending funds to address derivated from Federation using the provided derivation argument by hashing them:

	`Keccak256(derivationArgumentsHash, userRefundAddress, LBCAddress, LPBtcAddress)`

- Checks if the derivation hash was used already. If so, it will refund the bitcoin sender.

- Verifies provided Liquidity Bridge Contract address actually belongs to a contract

- Depending on the shouldTransferToContract value received as parameter from Liquidity Bridge Contract:

 - If **true**, Bridge will check if Locking Cap value is surpassed:
    - If not surpassed:
		- Bridge will transfer value to the Liquidity Bridge Contract
		- will save Fast Bridge Federation P2SH - Derivation Arguments hash - Federation P2SH tuple in Bridge Storage
		- will save Derivation Arguments and boolean value in Bridge Storage (to verify if given derivation arguments were already used)
		- will mark tx as processed
		- will store the UTXO for the Bridge to use, among any data concerning to the Bridge operation.

	- If surpassed, the Bridge will refund Liquidity Provider

 - If **false**:
   - Refund bitcoin sender

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


#### Limitations
- Transactions with depth bigger than 4320 will not be processed. That is enough depth to be able to search backwards one month worth of blocks.
If given depth is higher than the accepted limit, unclaimed funds will be lost.

- If the given transaction is invalid or the provided address cannot be computed from the given derivation arguments, the transaction will be unable to claim the funds.


#### Flow chart
![registerFastBridgeBtcTransaction() flow](RSKIP176/registerFastBridgeBtcTransactionFlow.png)


### Federation Address Derivation

The address derivation starts with the creation of a custom redeem script that makes a push of a 32 bytes hash created from the provided derivation arguments. That data is then dropped.

This new custom redeem scrtipt has the following format:

    scriptPubKey: OP_HASH160 <redeemScriptHash> OP_EQUAL
    scriptSig: OP_0 <signatures> <redeemScript>
    redeemScript: <derivationArgumentsHash> OP_DROP OP_M <publicKeys> OP_N OP_CHECKMULTISIG

 This custom redeem script is executed in the same way to the standard one, allowing the same Federation members to sign the releases of these ad-hoc Federations.


### Rationale
This RSKIP derivates from the original [RSKIP 175](https://github.com/rsksmart/RSKIPs/blob/fast-bridge/IPs/RSKIP175.md "RSKIP 175"), but this alternative way was chosen as the platform does not support that the Bridge executes sub contracts.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

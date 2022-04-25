---
rskip: 37
title: Single Address Smart Wallets
description: 
status: Draft
purpose: Sca, Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-02-18
---

# Single Address Smart Wallets

|RSKIP          |37           |
| :------------ |:-------------|
|**Title**      |Single Address Smart Wallets|
|**Created**    |18-FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca/Usa |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

This RSKIP tries to address several problems with the current contract wallet designs. Currently smart wallets need to maintain two accounts. One is a simple account (no code) what is used to consume the gas required to perform operations on the smart wallet. The second is the smart wallet (a contract) that holds of most user funds.

This complicates smart wallet design.

Also this prevents Lumino (Lightning network) off-chain channels to be created when one of the parties has a multi-signature wallet.

To achieve true financial inclusion within the next 10 years we need to allow high transaction volumes, and having the LTCP protocol be the settlement layer for the Lumino network seems the optimal way to achieve it. 

Last, the current design does not allow signature algorithm agility.

Therefore this RSKIP must consider three different use cases into a single proposal.

Three different proposals are evaluated and discussed.  This RSKIP  proposes to add a new type of code to accounts/contracts, the SigVerCode.

# **Motivation**

This RSKIP tries to address the following problems with an unified solution.

1. Currently smart wallets need to maintain two accounts. One is a simple account (no code) what is used to consume the gas required to perform operations on the smart wallet. The second is the smart wallet (a contract) that holds of most user funds. This complicates smart wallet design.

2. In Lumino, a payment channel is a smart contract controlled by two parties. Each party could be identified by a ECDSA private key or by an address. If it is identified by a ECDSA private key, then channels are limited to the ECDSA signature standard and do not support that one of the parties controls its funds by a 2 of 3 multi-signature wallet. If the channel is controlled by two contract addresses (and msg.sender is used to authenticate them) then a channel settlement or top-up message cannot be compact and self-contained, because it requires the approval from two other contracts. If the message comes from one of the addresses, then the other party signature must be provided in the data payload. A good wallet design should allow a payment channel to be created by two parties where each party is identified by an address, and together they can create a single message, signed by both, that goes directly to the channel contract (both the origin and the destination of the messages will be the channel contract). This allows signatures to be easily compressed by LTCP.

3. Cryptographic algorithm agility. The weakest cryptographic construction in RSK is the ECDSA signature scheme. Giving the option to users to use other signatures schemes can help the network tolerate the weaken of ECDSA, without the need for costly hard-forks. 

4. One interesting feature to have is on-the-fly contract addresses. These are addresses that can be created off-line, but are not related to an ECDSA-controlled account, but to a contract-controlled account. Basically the address should be the hash of the contract information (code, memory), so that an user can submit this information at a later time to "de-hibernate" the address, and make it fully operable. This requires that the signature-verification code and the execution code be contained into a single address.

# Discussion on Problem 3

The current design and implementation of Bitcoin/RSK/Ethereum relies on ECDSA, and more specifically the secp256k1 curve, as the only signature algorithm that enables and protect payments. The ECDSA secp256k1 signature scheme is a Certicom standard and it's currently considered secure, but nevertheless it may become insecure in the future. Past cases of weakness discovered in cryptographic algorithms show that generally standard cryptographic schemes do not become suddenly completely insecure: there is generally a window of time to adopt a new standard before a catastrophic failure is found. If this happens to RSK, there will be probably some time to upgrade and hard fork the network. To prevent a hard-fork there are two possible solutions to this problem:

 

1. Launch RSK allowing a second signature standard for transactions as a "backup", such as RSA, or a quantum resistant one, such as Merkle-Winternitz. An algorithm prefix should be added to the public keys.

 

2. Allow any signature system to be used to protect RSK accounts. Because RSK already has a Turing complete language, alternate signature schemes could be specified by the user.

Allowing this is a bit tricky, but fits nicely in the current design. There are two additional possibilities: re-use the contract code to allow signature verification using the existent code in an account, or add a new account code special for signature verification. The second approach, which has many benefits over the "code reuse" approach, is explained below. There is also a third option.

Now we present three different methods to achieve user programmable signature schemes.

 

**SigVerCode proposal**

 

A new field "SigVerCode" is added to RSK accounts. When SigVerCode is not empty, if an external transaction is sent to the contract and the transaction has a signature field, then the signature is NOT verified by the standard ECDSA engine. The signature value is made available to be accessed by the contract SigVerCode, which is executed. Also a fixed pre-defined amount of GAS (SIGN-VERIFY-GAS) is given FOR FREE for the contract execution. The GAS is paid by the miner. The amount SIGN-VERIFY-GAS should be roughly equivalent to the time it takes to verify an ECDSA signature (or a bit more, such as 2 times more). If not all the GAS is used, the unused GAS is returned to the miner. The script SigVerCode script should return a single Boolean value, which indicates the correctness of the signature.The SIGN-VERIFY-GAS restriction prevents network spamming, allowing nodes to verify user signatures without the risk of denial of service attacks. 

The transaction (r, s,v) values are re-interpreted. 

* **nonce**

* **gasprice**

* **startgas**

* **to**

* **value**

* **data**

* **v**: **Rlp-List of:**

    * **Transaction format version**

    * **max signature length (Maximum 2^20-1) [optional, default is 80]**

    * **number of ephemeral data elements [optional, default is zero]**

    * **optional sender address. [optional]**

* **r: signature data**

* **s: Rlp-list of ephemeral data elements [optional]**

 

The signature must cover all the fields except r and s. The signature covers a merkle root of the merkle tree of all ephemeral data elements. The fee per byte applies also to all fields,  except for ephemeral data. The fee per ephemeral byte applies to ephemeral items.

Also value, gasprice and  startgas can be coded in a compact integer floating point representation with a variable number of bytes for mantissa and 5 bits of base-10 exponent.

A sample SigVerCode pseudo-code would be:

 

pubkey = contract.storage["pubkey"]

signature = Transaction.signature

txHash = Transaction.hash

if (not VerifySignature(pubkey,signature,txHash))

   return False

else

  return True

 

Other uses for SigVerCode:

 

•   SigVerCode could check the destination and prevent some destination to be used (filters)

•    SigVerCode could limit the amount of money extracted.

•    SigVerCode could check for other signatures such as multi-sig.

However, the SigVerCode would be unable to modify or event read the persistent storage (SLOAD/SSTORE), also SigVerCode would have no access to any other contract (e.g., EXTCODESIZE, CODECOPY, CALL). This restriction allows the miner to collect all valid transactions and then add them one by one to the block. If SigVerCode allows modification of the persistent state, then SigVerCode transactions would need to be re-analyzed over and over, leading to a possible DoS attack vector to miners.

So basically when a transaction is executed from a sender to a receiver, both code in the sender (SigVerCode) and the receiver (code) are executed. Another advantage of SigVerCode over multi-sig contracts is that SigVerCode allows the wallet application to easily support multi-signatures or any complex access procedure.

In this proposal requires a new field "sender" on each transaction that uses SigVerCode.

The following table shows the interaction between the sender and receiver fields and the normal code and wallet code.

<table>
  <tr>
    <td></td>
    <td>SigVerCode empty</td>
    <td>SigVerCode non-empty</td>
  </tr>
  <tr>
    <td>Code empty</td>
    <td>ECDSA Account
sender field is empty
receiver field is set</td>
    <td>UserSig account
sender is account address
receiver is set</td>
  </tr>
  <tr>
    <td>Code non-empty</td>
    <td>Contract
receiver is contract address
sender field is empty</td>
    <td>UserSig Contract account
sender is contract
if receiver field is empty the receiver is set to sender
If receiver is set, then message is sent to receiver </td>
  </tr>
</table>


**CodeReuse Proposal**

Transactions have as destination the contract that will pay the fees. The normal code is executed for SIGN-VERIFY-GAS units of gas. If the code executes the (new) ACCEPT opcode before OOG exception, the transaction is accepted. If it does not, the transaction is halted and considered invalid. If the persistent storage is modified or read (by SLOAD/SSTORE) or other contract is accessed before ACCEPT, the transaction is also considered invalid. The block containing the transaction is also considered invalid.

Mixing signature checking and normal contract code has the disadvantage that it obfuscates the signature scheme used.

**Static Contracts Proposal**

Another way to achieve the same effect is to split the wallet into two contracts, where one of them codes the signature verification (Code=SigVerCode) part and the other obeys the orders of the first (Code=Wallet). This way there is no need to introduce a new SigVerCode field. 

The transaction will originate in the SigVer contract, and will also have the same contract as destination. A special flay "receiver-pays" in transactions will change the way the transaction is processed. Two more fields need to be created “maxTxLenght” and “signatureData”.

When a node receives a transaction marked with  "receiver-pays", it first executes the destination contract. This contract can access the hash of the transaction *msg.ID* and the signatureData as the data field (msg.sigData).. The SigVer contract evaluates the signature and forwards the message to the wallet contract using the CALL opcode. 

Because SigVerCode must run with several restrictions, such as that no persistent memory cell can be written, no context inspection.

Therefore we Mark SigVerCode contracts as "static". A static contract can only call other static contract. This prevents users calling non-static contracts by mistake, and blocking access to wallets.

**Comparison**

In any of the three proposals, signature data could be segregated in a field similar to the "s" field, and accessed using new opcodes (e.g. SIGLOAD). This allows to dispose the signature data (as segregated witness) and reduce the blockchain storage on some nodes

This proposal uses an empty "s" field to mark that the origin of the transaction is the external world (origin=0). The receiver field semantics is unchanged.

<table>
  <tr>
    <td></td>
    <td>On-the-fly contract addresses</td>
    <td>hard-fork difficulty
(from 1 to 10) </td>
    <td>Allow better native compression through LTCP</td>
    <td>Obscures real destination in external tx</td>
  </tr>
  <tr>
    <td>SigVerCode</td>
    <td>Yes</td>
    <td>5</td>
    <td>Yes</td>
    <td>No</td>
  </tr>
  <tr>
    <td>CodeReuse</td>
    <td>Yes</td>
    <td>3</td>
    <td>No</td>
    <td>Yes</td>
  </tr>
  <tr>
    <td>Static Contracts</td>
    <td>No</td>
    <td>1</td>
    <td>No</td>
    <td>Yes</td>
  </tr>
</table>

# **Specification**

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
---
rskip: TBD
title: Implement EIP-7702 Account Abstraction in Rootstock
description:  
status: Draft
purpose: Sca, Usa
author: PDG (@patogallaiovlabs), SM (@smishraiov), SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2026-01-06
---

| RSKIP | TBD |
| :------------ |:-------------|
| **Title** | Implement EIP-7702 Account Abstraction in Rootstock |
| **Created** | 06-JAN-2026 |
| **Author** | Patricio Gallardo, Shreemoy Mishra, Sergio Demian Lerner |
| **Purpose** | Sca, Usa |
| **Layer** | Core |
| **Complexity** | 3 |
| **Status** | Draft |

## Abstract

This RSKIP proposes to implement [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) style Account Abstraction in Rootstock to improve user experience and smart contract wallet capabilities. This will allow Externally Owned Accounts (EOAs) to set code in their accounts in the form of delegation pointers (contract addresses) which contain the code to run in the EOA’s own context. As stated in the abstract of EIP-7702, This is done by attaching a list of authorization tuples – individually formatted as `[chain_id, address, nonce, y_parity, r, s]` – to the transaction. For each tuple, a delegation indicator `(0xef0100 || address)` is written to the authorizing account’s code. All code executing operations must load and execute the code pointed to by the delegation.

## Motivation

There have been several proposals for Account Abstraction in Ethereum and also in Rootstock ([RSKIP-167](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP167.md)).  EIP-7702 was the version that finally received very strong support and has seen strong adoption since it was included in Ethereum’s Pectra upgrade. Some benefits of EIP-7702 include batched transactions, sponsored transactions, and fine grained access control and permissions for smart-contract  wallets. This approach can even work with existing infrastructure and tooling for other account abstraction mechanisms, such as ERC-4337 wallets.

The existence of prior RSKIPS (such as Install Code Precompile, RSKIP-167), the sponsored transaction mechanism RIF Relay, and direct queries from Rootstock community members about the availability of EIP-7702 (and ERC-4337 in the past) provide further motivation to implement this proposal for Account Abstraction.

## Specification

The specification is intended to mirror that of [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702). Large sections of text are copied verbatim from EIP-7702. Points of divergence from the EIP specification are noted. 

### **Prerequisites**

EIP-7702 has several prerequisites that have not been implemented in Rootstock. Some of them are not strictly necessary for this RSKIP. For example, we do not necessarily require EIP-2929, EIP-2930, EIP-3607, or EIP-4844. However, we do require the implementation of [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) style typed transactions. While transaction versions have not been implemented in Rootstock - *Transaction Receipts* have already been modified to include a “type” field consistent with EIP-2718. We also require [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541) to prevent deployments of contracts that contain deployed bytecode starting with `0xEF`. This is to avoid interference with EIP-7702’s “delegation indicator”. As initial drafts (corresponding to EIP-2718 and EIP-3541) we present the pull requests for [RSKIP-543](https://github.com/rsksmart/RSKIPs/pull/543) and [RSKIP-544](https://github.com/rsksmart/RSKIPs/pull/544).

Todo: if the PRs for new RSKIPs are merged into main, then update the links. 

### Parameters

| Parameter | Value |
| --- | --- |
| `SET_CODE_TX_TYPE` | `0x04` |
| `MAGIC` | `0x05` |
| `PER_AUTH_BASE_COST` | `15500` |
| `PER_EMPTY_ACCOUNT_COST` | `25000`  |

A new (EIP-2718 style) transaction known as the “set-code transaction” is introduced, where the `TransactionType` is `SET_CODE_TX_TYPE` . Though not strictly necessary, the transaction type  `0x04` is maintained for consistency with Ethereum. This only matters if [RSKIP-543](https://github.com/rsksmart/RSKIPs/pull/543) (or something similar) corresponding to EIP-2718 is also implemented in Rootstock.

`MAGIC` is used to recover (`ecrecover` from signed message) the address of the EOA (an *authorizer*) that authorizes  the change to set code into its own account. 

A set-code transaction will attempt to set or modify code for a list of EOAs (called *authorizers*). `PER_EMPTY_ACCOUNT_COST`  is the up-front gas collected from the transaction sender for each EOA in the list. This is collected based on just the size of the list regardless of whether the code for an EOA is actually modified or not. `PER_AUTH_BASE_COST` is the cost for each EOA whose code is actually modified successfully, through this transaction. The difference between upfront and ultimate costs (per EOA) is refunded to the sender. To account for gas schedule differences between Rootstock and Ethereum, the base cost in this RSKIP is a bit higher than the value in the EIP.

### Set-code transaction

The `TransactionPayload` is the RLP serialization of the following:

```jsx
rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit,
destination, value, data, access_list, authorization_list, signature_y_parity,
signature_r, signature_s]) 
```

where 

`authorization_list = [[chain_id, address, nonce, y_parity, r, s], ...]`

and this is described more fully below.  Note that the transaction payload includes fields that do not currently exist on Rootstock. This includes the fields `max_priority_fee_per_gas` and  `max_fee_per_gas` from EIP-1559 and `access_list` from EIP-2930. These field names are retained to ensure compatibility for preexisting wallet software and tooling in use in Ethereum. The `access_list` can simply be ignored by RSKJ - at least initially. The gas fee variables (`max_priority_fee_per_gas`,  `max_fee_per_gas`) can be reinterpreted internally in terms of `gasprice` and `minimumGasPrice`. 

Following EIP-7702, a null destination is not valid for the set-code transaction. Thus, it cannot be used for contract creation. The original EIP also prohibits the use of this transaction for EIP-4844 style blobs. Since Rootstock currently has no implementation for Ethereum’s blob-carrying transactions, this restriction is currently moot. However, we leave this remark here for future reference in case there is a RSKIP to implement a system similar to EIP-4844.

The `signature_y_parity, signature_r, signature_s` elements of this transaction
represent a secp256k1 signature over `keccak256(SET_CODE_TX_TYPE ||
TransactionPayload)`.

The `authorization_list` is a list of tuples that indicate what code the signer of each tuple (signer is called an “authorizer”) desires to execute in the context of their EOA.  A single set-code transaction can therefore be used to set or modify code for multiple EOAs. Each EOA’s authorization is a tuple in the list. While each EOA must sign its own authorization, they need not send their set-code transaction themselves. The owner of an EOA can send their own set code transaction - if they wish. A set-code transaction is invalid if the length of `authorization_list` is zero. 

The set-code transaction is also considered invalid when any field in an authorization
tuple cannot fit within the following bounds:

`assert auth.chain_id < 2**256
 assert auth.nonce < 2**64 - 1
 assert len(auth.address) == 20
 assert auth.y_parity < 2**8
 assert auth.r < 2**256
 assert auth.s < 2**256` 

The  `ReceiptPayload` (expected to be in EIP-2718 format) for this transaction is
`rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`.

### Behavior

The authorization list is processed before the execution portion of the transaction begins, but after the sender’s nonce is incremented.

For each `[chain_id, address, nonce, y_parity, r, s]` tuple, perform the
following:

1. Verify the chain ID is 30 or 31, the ID of the current chain. A chain ID of 0 can be used to indicate universal cross-chain deployment.
2. Let `authority = ecrecover(msg, y_parity, r, s)`.
    - Where `msg = keccak(MAGIC || rlp([chain_id, address, nonce]))` and  `MAGIC` is defined above in parameters.
    - Verify `s` is less than or equal to `secp256k1n/2`, as specified in [EIP-2](https://eips.ethereum.org/EIPS/eip-2).
3. Verify the code of `authority` is empty or already delegated (see *delegation indicator* below).
4. Verify the nonce of `authority` is equal to `nonce`.
5. Add `PER_EMPTY_ACCOUNT_COST - PER_AUTH_BASE_COST` gas to the global refund
counter if `authority` is not empty.
6. Set the code of `authority` to be `0xef0100 || address`. This is a delegation
indicator.
    - If `address` is `0x0000000000000000000000000000000000000000`, do not write the delegation indicator. Clear the account’s code by resetting the account’s code hash to the empty code hash `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`.
    - This indicates that set-code transaction can be used to ‘insert code’ (delegate address, actually) into a pure EOA, to update the account to point to a code in a different delegated address, or to delete delegation entirely in order to reset the account back to a pure EOA.
7. Increase the nonce of `authority` by one.

If any step above fails, immediately stop processing the tuple and continue to the next tuple in the list. When multiple tuples from the same authority are present, set the code using the address in the last valid occurrence.

Note, if transaction execution results in failure (e.g. any exceptional condition or code reverting), the processed delegation indicators is *not rolled back*.

### Delegation indicator

Delegation indicators use the undefined EVM opcode `0xEF` from  [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541). This EIP has not been implemented on Rootstock. Since its adoption in Ethereum , EIP-3541 made it  impossible to deploy a contract if the intended deployed bytecode starts with `0xEF`.   EIP-7702 takes advantage of this “banned opcode” and uses it as a delegation marker for an EOA. This indicates that the code must be handled differently than regular code. The complete delegation indicator is a sequence of 3 bytes `0xEF0100.` This is followed by the address of the contract with the actual implementation logic. The 3-byte format starting with `0xEF` follows a pattern introduced in [EIP-3540](https://eips.ethereum.org/EIPS/eip-3540) where the first two bytes represent a “magic” sequence and the next byte represents a version number.

The presence of a delegation indicator (`0xef0100 || address`) forces all code-executing operations to follow the address pointer to obtain the code to execute. For example, `CALL` loads the code at `address` and executes it in the context of `authority` i.e. the EOA.

The affected executing operations are:

- `CALL`
- `CALLCODE`
- `DELEGATECALL`
- `STATICCALL`
- any transaction where `destination` points to an address with a delegation
indicator present

For code reading, only `CODESIZE` and `CODECOPY` instructions are affected. They operate directly on the executing code instead of the delegation. For example, when executing a delegated account `EXTCODESIZE` returns `23` (the size of `0xEF0100 || address`) whereas `CODESIZE` returns the size of the code residing at `address` (the actual implementation). *This means during delegated execution `CODESIZE` and `CODECOPY` produce a different result compared to calling `EXTCODESIZE` and `EXTCODECOPY` on the authority (the EOA).*

### Transaction origination

EOA’s with valid delegation indicators i.e. `0xef0100 || address`, can originate transactions. Accounts with any other code values may not originate transactions - continuing the current restriction that transactions cannot originate from contracts. Since EIP-3607, Ethereum  rejects any transaction that originate from accounts that contain code.  EIP-7702 creates an exception to this rule. There is no such such explicit restriction on Rootstock. In Rootstock, we implicitly assume that the origin of a transaction is always an EOA. Therefore, no exception needs to be created. However, if EIP-3607 is introduced in the future, then an exception will have to be carved out for EOAs that have a delegation indicator to originate transactions.

### Gas Costs

The transaction sender will pay for all authorization tuples, regardless of validity or duplication. The cost of a set-code transaction will be at least  `21000 + 16 * non-zero calldata bytes +
4 * zero calldata bytes`  plus the cost of reading accounts and storage cells. 

Additionally, an up front cost of `PER_EMPTY_ACCOUNT_COST * authorization list length` is collected, but some of these will be reimbursed after accounting for actual cost of setting code. This is reflected in `PER_AUTH_BASE_COST` which accounts for the cost to process each authorization tuple and set the delegation destination. The cost for this operation is decomposed into several components such as: 101 bytes of calldata, recovering the `authority` address from their signature, reading the nonce and code of `authority` , storing values in the account, and the cost to deploy code.

The `PER_AUTH_BASE_COST` is the cost to process the authorization tuple and set the delegation destination. To compute a fair cost for this operation, the authors of EIP-7702 reviewed its impact on the system as follows:

- ferry 101 bytes of calldata = `101 * non-zero cost (16) = 1616`
- recovering the `authority` address = `3000`
- reading the nonce and code of `authority` = `2600` (this read operation is `700` on Rootstock)
- storing values in already warm account = `200` (this write operation is about `5000` on Rootstock)
- cost to deploy code = `200 * 23 = 4600`

The authors of EIP-7702 estimate the cumulative impact of the above at `12016` gas, which they rounded up to `PER_AUTH_BASE_COST = 12500`. They then set the upfront cost to be twice of this value `PER_EMPTY_ACCOUNT_COST = 25000` . As noted above, the gas schedule in Rootstock is different (arising mainly from EIP-2929). We set our base cost about 3K higher, at  `PER_AUTH_BASE_COST = 15500` . However, we keep the upfront cost the same as in Ethereum at  `PER_EMPTY_ACCOUNT_COST = 25000` . Once there is an implementation for the proposal, these values can be updated further after some benchmarking.

### No Precompiles

An EOA cannot use a precompile address as a delegate. When a precompile address is the target of a delegation, the retrieved code is considered empty and `CALL`, `CALLCODE`, `STATICCALL`, `DELEGATECALL`instructions targeting this account will execute empty code, and therefore
succeed with no execution when given enough gas to initiate the call.

### No Chained Delegation (no loops)

In case a delegation indicator points to another delegation, creating a potential chain or loop of delegations, clients must retrieve only the first code and then stop following the delegation chain.

## Implementation

Todo: There is currently no implementation for this proposal.

## Rationale

The original EIP has [extensive rationale](https://eips.ethereum.org/EIPS/eip-7702#rationale) section with detailed discussion of several aspects such as no initcode, no contract creation, no precompiles, etc. For the sake of brevity, we do not replicate them here. However, reviewers and implementers should pay particular attention to the the issues mentioned therein. One example is the transaction origin invariant.  Allowing `tx.origin` to set code and execute its own delegated code enables what is called self-sponsoring. It allows users to take advantage of EIP-7702 without relying on any third party infrastructure. However, that means the EIP breaks the invariant that `msg.sender == tx.origin`
only happens in the topmost execution frame of a transaction. This will affect smart contracts containing `require(msg.sender == tx.origin)` style checks.

Reviewers and implementers should pay attention to the discussion on [security considerations](https://eips.ethereum.org/EIPS/eip-7702#security-considerations)  in the  EIP, some of which are listed below.

# Security

- A poorly implemented delegate can *allow a malicious actor to take near complete
control over a signer’s EOA*.
- Attackers can try to front run an EOA’s set-code transaction, so developers must check for signed authorizations using `ecrecover`.
- When upgrading an EOA’s code (delegate address or code therein), care must be taken to manage storage slots since all delegated code (existing and new) will run in the context of the EOA. ****
- Breaks in atomic sandwich protections which rely on `tx.origin` and reentrancy guards of the style `require(tx.origin == msg.sender)`
- Transaction (gas) sponsors may require their users to post some bond or use reputational mechanisms. This is to guard against the possibility of relayers not being reimbursed for gas due to change in a EOA’s authorized code or asset sweeping.
- Transaction propagation: An EOA’s nonce may be incremented more than once per transaction.
- Transaction propagation:  once an EOA has delegated to code, that code can be called by anyone. It becomes possible to cause transactions from other accounts to become stale. The authors of EIP-7702 recommend that clients do not accept more than one pending transaction from any EOA with a non-zero delegation indicator.

## Backward compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated.

## Test cases

Todo.

## References

\[1] [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702): Set Code for EOAs

\[2] [RSKIP167](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP167.md): Install Code Precompile

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
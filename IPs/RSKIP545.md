---
rskip: 545
title: Set Code for EOAs
description: Injects code into an EOA account through a set-code transaction
status: Draft
purpose: Sca, Usa
author: PDG (@patogallaiovlabs), SM (@smishraiov), SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2026-01-06
---

## Abstract

This RSKIP proposes to implement [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) style Account Abstraction in Rootstock to improve user experience and smart contract wallet capabilities. This will allow Externally Owned Accounts (EOAs) to set code in their accounts in the form of delegation pointers (contract addresses) which contain the code to run in the EOA’s own context. As stated in the abstract of EIP-7702, This is done by attaching a list of authorization tuples – individually formatted as `[chain_id, address, nonce, y_parity, r, s]` – to the transaction. For each tuple, a delegation indicator `(0xef0100 || address)` is written to the authorizing account’s code. All code executing operations must load and execute the code pointed to by the delegation.

## Motivation

There have been several proposals for Account Abstraction in Ethereum and also in Rootstock ([RSKIP-167](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP167.md)).  EIP-7702 was the version that finally received very strong support and has seen strong adoption since it was included in Ethereum’s Pectra upgrade. EIP-7702 does not itself provide new wallet features: it lets an EOA set code in its account. Its main anticipated use is account abstraction, where that code provides features such as batched transactions, sponsored transactions, and fine grained access control and permissions for smart-contract wallets. This RSKIP is likewise an enabler for such features rather than an implementation of them. This approach can even work with existing infrastructure and tooling for other account abstraction mechanisms, such as ERC-4337 wallets.

The existence of prior RSKIPs (such as Install Code Precompile, RSKIP-167), the sponsored transaction mechanism RIF Relay, and direct queries from Rootstock community members about the availability of EIP-7702 (and ERC-4337 in the past) provide further motivation to implement this proposal for Account Abstraction.

## Specification

The specification is intended to mirror that of [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702). Large sections of text are copied verbatim from EIP-7702. Points of divergence from the EIP specification are noted. 

### Prerequisites

EIP-7702 has several prerequisites that have not been implemented in Rootstock. Some of them are not strictly necessary for this RSKIP. For example, we do not necessarily require EIP-2929, EIP-2930, EIP-3607, or EIP-4844. This RSKIP requires:

* [RSKIP-543](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP543.md), which introduces [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) style typed transactions.
* [RSKIP-546](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP546.md), which introduces Ethereum's Type 1 and Type 2 transaction formats. The set-code transaction reuses their fields (`access_list`, `max_fee_per_gas` and `max_priority_fee_per_gas`) and the rules that apply to them.
* [RSKIP-701](https://github.com/rsksmart/RSKIPs/pull/701), which specifies `SELFDESTRUCT` for delegated accounts. Without it, a delegated account that executes `SELFDESTRUCT` would be deleted and lose its nonce, so the authorizations it has signed could be replayed.

### Parameters

| Parameter | Value |
| --- | --- |
| `SET_CODE_TX_TYPE` | `0x04` |
| `MAGIC` | `0x05` |
| `PER_AUTH_BASE_COST` | `15500` |
| `PER_EMPTY_ACCOUNT_COST` | `25000`  |
| `DELEGATION_RESOLUTION_COST` | `700` |

A new (EIP-2718 style) transaction known as the “set-code transaction” is introduced, where the `TransactionType` is `SET_CODE_TX_TYPE` . Though not strictly necessary, the transaction type `0x04` is maintained for consistency with Ethereum.

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

and this is described more fully below.  Note that the transaction payload includes the fields `max_priority_fee_per_gas` and `max_fee_per_gas` from EIP-1559 and `access_list` from EIP-2930, which RSKIP-546 introduces on Rootstock. These field names are retained to ensure compatibility for preexisting wallet software and tooling in use in Ethereum. The `access_list` follows the rules of RSKIP-546: it MUST have the shape given there, it has no effect on execution under this RSKIP, and it is charged `80` gas per byte of its complete RLP encoding, with an empty list (`0xc0`) charged nothing. The fee fields also follow the rules of RSKIP-546 for Type 2 transactions: nodes MUST reject a set-code transaction with `max_priority_fee_per_gas > max_fee_per_gas`, and the transaction's effective gas price is `max_priority_fee_per_gas`. Of course, all of the fields have to be used when validating the transaction's signature.

Following EIP-7702, a null destination is not valid for the set-code transaction. Thus, it cannot be used for contract creation. The original EIP also prohibits the use of this transaction for EIP-4844 style blobs. Since Rootstock currently has no implementation for Ethereum’s blob-carrying transactions, this restriction is currently moot. However, we leave this remark here for future reference in case there is a RSKIP to implement a system similar to EIP-4844.

The `signature_y_parity, signature_r, signature_s` elements of this transaction
represent a secp256k1 signature over `keccak256(SET_CODE_TX_TYPE ||
TransactionPayload)`.

The `authorization_list` is a list of tuples that indicate what code the signer of each tuple (signer is called an “authorizer”) desires to execute in the context of their EOA.  A single set-code transaction can therefore be used to set or modify code for multiple EOAs. Each EOA’s authorization is a tuple in the list. While each EOA must sign its own authorization, they need not send their set-code transaction themselves. The owner of an EOA can send their own set code transaction - if they wish. A set-code transaction is invalid if the length of `authorization_list` is zero. 

The set-code transaction is also considered invalid when any field in an authorization tuple cannot fit within the following bounds:

```
assert auth.chain_id < 2**256
assert auth.nonce < 2**64
assert len(auth.address) == 20
assert auth.y_parity < 2**8
assert auth.r < 2**256
assert auth.s < 2**256
``` 

Note that these initial assertions are only to ensure the values fit into appropriately-sized unsigned integers (e.g. `uint64` for `nonce`). Additional checks are performed later, for example step 2 (below) ensures an authority's nonce can still be incremented safely without overflowing.

### Transaction Receipt

The  `ReceiptPayload` (expected to be in EIP-2718 format) for this transaction is
`rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`. So the overall EIP-2718 encoding for the receipt is `04 || rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`

### Behavior

The authorization list is processed before the execution portion of the transaction begins, but after the sender’s nonce is incremented.

For each `[chain_id, address, nonce, y_parity, r, s]` tuple, perform the
following:

1. Verify the chain ID is 0 or the ID of the current chain (which for Rootstock can be 30, 31 or 33).
2. Verify the nonce is less than `2**64 - 1`.
3. Let `authority = ecrecover(msg, y_parity, r, s)`.
    - Where `msg = keccak(MAGIC || rlp([chain_id, address, nonce]))` and  `MAGIC` is defined above in parameters.
    - Verify `s` is less than or equal to `secp256k1n/2`, as specified in [EIP-2](https://eips.ethereum.org/EIPS/eip-2).
    - Verify `authority` is not the zero address `0x0000000000000000000000000000000000000000`.
4. Verify the code of `authority` is empty or `authority` is a delegated account (see *delegation indicator* below).
5. Verify the nonce of `authority` is equal to `nonce`.
    - This uses the current nonce of `authority`. When `authority` is the transaction sender (self-sponsoring), the sender's nonce has already been incremented for the transaction, so the tuple's `nonce` must be the transaction's nonce plus one.
6. Add `PER_EMPTY_ACCOUNT_COST - PER_AUTH_BASE_COST` gas to the global refund
counter if the code of `authority` is not empty, which by step 4 means `authority` is already a delegated account.
    - This differs from EIP-7702, where the refund applies whenever `authority` exists in the state. On Rootstock, an `authority` that has a balance or a nonce but no code receives no refund.
7. Set the code of `authority` to be `0xef0100 || address`. This is a delegation
indicator. Also set the delegation authority flag of `authority` (see *delegation indicator* below).
    - If `address` is `0x0000000000000000000000000000000000000000`, do not write the delegation indicator or set the flag. Clear the account’s code by removing the value at its code key from the Unitrie. Afterwards `EXTCODEHASH` on the account returns the empty code hash `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470` (`keccak256` of the empty string), as for an account that never had code.
    - In the Unitrie, writing the delegation indicator stores it as the value at the account's code key, and also writes the value `0x01` at the account's storage prefix key if it is not already present, as for a contract. Neither step touches any storage cell.
    - A set-code transaction can therefore set a delegation on an EOA, change it to point to a different address, or clear it. These operations change only the account's code (and, when setting a delegation, its delegation authority flag). They never read, write or clear any storage cell of the account.
    - Clearing a delegation therefore does not return the account to its state before it was delegated: storage written while it was delegated remains, and the delegation authority flag stays set. A later delegation runs its code against that storage. As a result, an EOA can now hold persistent storage that no protocol operation clears, and any application logic that assumes an EOA has no storage must account for this. As with a proxy contract, the storage can only be cleared by code running in the account's context: for example, by delegating to a contract that writes zero to the slots in use and then calling the account. `SELFDESTRUCT` does not clear it: under RSKIP-701, a delegated account that executes `SELFDESTRUCT` keeps its storage.
8. Increase the nonce of `authority` by one.

If any step above fails, immediately stop processing the tuple and continue to the next tuple in the list. When multiple tuples from the same authority are present, set the code using the address in the last valid occurrence.

Note, if transaction execution results in failure (e.g. any exceptional condition or code reverting), the processed delegation indicators are *not rolled back*. Neither are the refunds added in step 6: they are still applied at the end of the transaction, subject to the refund cap described in [Gas Costs](#gas-costs), even when execution ends in an exceptional halt that consumes all remaining gas.

### Delegation indicator

Delegation indicators use the undefined EVM opcode `0xEF` from  [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541). Since its adoption in Ethereum, EIP-3541 made it impossible to deploy a contract if the intended deployed bytecode starts with `0xEF`. On Rootstock, the same rule was introduced by [RSKIP-544](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP544.md) in the Vetiver network upgrade, so contracts deployed before Vetiver can still have code that starts with `0xEF`.   EIP-7702 takes advantage of this “banned opcode” and uses it as a delegation marker for an EOA. This indicates that the code must be handled differently than regular code. The complete delegation indicator is a sequence of 3 bytes `0xEF0100`. This is followed by the address of the contract with the actual implementation logic. The 3-byte format starting with `0xEF` follows a pattern introduced in [EIP-3540](https://eips.ethereum.org/EIPS/eip-3540) where the first two bytes represent a “magic” sequence and the next byte represents a version number.

#### Delegation authority flag

A contract deployed before Vetiver may have code that is exactly a delegation indicator, so the presence of a delegation indicator in an account's code is not enough to recognize a delegation. Each account therefore has a **delegation authority flag**. Step 7 of [Behavior](#behavior) sets the flag whenever it writes a delegation indicator, and nothing clears it, including clearing the delegation. The flag is bit 1 (value `2`) of the account's state flags. An account's state is encoded as `rlp([nonce, balance])` when its state flags are zero and `rlp([nonce, balance, stateFlags])` otherwise; bit 0 (value `1`) is the existing hibernation flag.

A **delegated account** is an account that carries the delegation authority flag and whose code is exactly the 23-byte delegation indicator `0xef0100 || address`. Only the delegation of a delegated account is followed. An account whose code is a delegation indicator but that does not carry the flag, such as a contract deployed before Vetiver, is not a delegated account, and its code is executed as is.

#### Delegated execution

The presence of a delegation indicator (`0xef0100 || address`) in a delegated account forces all code-executing operations to follow the address pointer to obtain the code to execute. For example, `CALL` loads the code at `address` and executes it in the context of `authority` i.e. the EOA.

The affected executing operations are:

- `CALL`
- `CALLCODE`
- `DELEGATECALL`
- `STATICCALL`
- any transaction where `destination` is a delegated account

Only these operations follow a delegation. Balance credits that the protocol makes directly, rather than through a call, such as REMASC payouts and Bridge transfers (for example, peg-ins), add to the balance of a delegated account without executing its delegated code, as for any other account.

The code-reading operations `EXTCODESIZE`, `EXTCODECOPY` and `EXTCODEHASH` on a delegated account operate on the delegation indicator itself and do not follow it: they return `23`, copy the 23 bytes `0xef0100 || address`, and return `keccak256(0xef0100 || address)`, respectively.

Of the code-reading instructions, only `CODESIZE` and `CODECOPY` are affected by delegation. They operate directly on the executing code instead of the delegation. For example, when executing a delegated account `EXTCODESIZE` returns `23` (the size of `0xEF0100 || address`) whereas `CODESIZE` returns the size of the code residing at `address` (the actual implementation). *This means during delegated execution `CODESIZE` and `CODECOPY` produce a different result compared to calling `EXTCODESIZE` and `EXTCODECOPY` on the authority (the EOA).*

### Transaction origination

Delegated accounts can originate transactions. Since EIP-3607, Ethereum rejects any transaction whose sender has code, and EIP-7702 makes an exception for delegation indicators. Rootstock has not adopted EIP-3607 and does not, in general, check the code of a transaction's sender, so a delegated account can send legacy, Type 1 and Type 2 transactions without any exception being needed. If EIP-3607 is introduced on Rootstock in the future, it will need the same exception for delegated accounts.

The set-code transaction adds one check of its own: nodes MUST reject a set-code transaction whose sender has code, unless the sender is a delegated account.

As recommended by EIP-7702 (see [Security](#security)), nodes SHOULD accept at most one pending transaction from a delegated account. This is a transaction pool policy, not a consensus rule.

### Gas Costs

The transaction sender will pay for all authorization tuples, regardless of validity or duplication. The cost of a set-code transaction will be at least  `21000 + 16 * non-zero calldata bytes +
4 * zero calldata bytes`, plus the access-list charge of RSKIP-546, plus the cost of reading accounts and storage cells. 

Additionally, an up front cost of `PER_EMPTY_ACCOUNT_COST * authorization list length` is collected, but some of these will be reimbursed after accounting for actual cost of setting code. This is reflected in `PER_AUTH_BASE_COST` which accounts for the cost to process each authorization tuple and set the delegation destination. The cost for this operation is decomposed into several components such as: 101 bytes of calldata, recovering the `authority` address from their signature, reading the nonce and code of `authority` , storing values in the account, and the cost to deploy code.

The `PER_AUTH_BASE_COST` is the cost to process the authorization tuple and set the delegation destination. The authors of EIP-7702 derived it from the components below. The same components priced with Rootstock's gas schedule give the value used here:

| Component | EIP-7702 | Rootstock |
| --- | --- | --- |
| Ferry 101 bytes of calldata (`101 * 16`) | `1616` | `1616` |
| Recover the `authority` address | `3000` | `3000` |
| Read the nonce and code of `authority` | `2600` | `700` |
| Store values in the account | `200` | `5000` |
| Deploy code (`200 * 23`) | `4600` | `4600` |
| **Total** | `12016` | `14916` |

EIP-7702 rounds its total up to `PER_AUTH_BASE_COST = 12500` and sets `PER_EMPTY_ACCOUNT_COST = 25000`, twice that value. Rootstock's total differs mainly because Rootstock does not implement EIP-2929, under which the account is already warm when the values are stored. This RSKIP rounds the Rootstock total up to `PER_AUTH_BASE_COST = 15500` and keeps `PER_EMPTY_ACCOUNT_COST = 25000`, as in Ethereum. These values may be revised after benchmarking the implementation.

The refund in step 6 of [Behavior](#behavior) is therefore `25000 - 15500 = 9500` gas per authorization whose `authority` is already a delegated account. It is added to the same refund counter as other refunds, such as those for `SSTORE` and `SELFDESTRUCT`, and the total refund is capped at half of the gas used by the transaction, as for any other refund. The refund actually received can therefore be less than `9500` per authorization.

**Delegation resolution.** Following a delegation reads a second account, the delegate, in addition to the delegated account itself. In EIP-7702 this read is priced through EIP-2929 (`2600` gas cold, `100` warm). Rootstock does not implement EIP-2929, so the read is charged a flat `DELEGATION_RESOLUTION_COST = 700` gas, Rootstock's price for reading an account (as for `EXTCODESIZE`). The charge applies:

- to a `CALL`, `CALLCODE`, `DELEGATECALL` or `STATICCALL` whose target is a delegated account, in addition to the instruction's own cost;
- to a transaction whose `destination` is a delegated account, at the start of execution, from the gas that remains after the intrinsic cost. If not enough gas remains, execution fails as out of gas.

It is charged whether or not the delegate has code, including when the delegate is a precompile (see [No Precompiles](#no-precompiles)).

### No Precompiles

An EOA cannot use a precompile address as a delegate. When a precompile address is the target of a delegation, the retrieved code is considered empty and `CALL`, `CALLCODE`, `STATICCALL`, `DELEGATECALL` instructions targeting this account will execute empty code, and therefore
succeed with no execution when given enough gas to initiate the call.

### No Chained Delegation (no loops)

In case a delegation indicator points to another delegation, creating a potential chain or loop of delegations, clients must retrieve only the first code and then stop following the delegation chain.

### Activation

This RSKIP is proposed for the Cardamom network upgrade ([RSKIP-694](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP694.md)). Activation block heights are TBD.

This RSKIP MUST NOT activate before RSKIP-543, RSKIP-546 and RSKIP-701. Set-code transactions are valid only when all of them and this RSKIP are active; before that, nodes MUST reject them.

## Rationale

The original EIP has [extensive rationale](https://eips.ethereum.org/EIPS/eip-7702#rationale) section with detailed discussion of several aspects such as no initcode, no contract creation, no precompiles, etc. For the sake of brevity, we do not replicate them here. However, reviewers and implementers should pay particular attention to the issues mentioned therein. One example is the transaction origin invariant.  Allowing `tx.origin` to set code and execute its own delegated code enables what is called self-sponsoring. It allows users to take advantage of EIP-7702 without relying on any third party infrastructure. However, that means the EIP breaks the invariant that `msg.sender == tx.origin`
only happens in the topmost execution frame of a transaction. This will affect smart contracts containing `require(msg.sender == tx.origin)` style checks.

Reviewers and implementers should pay attention to the discussion on [security considerations](https://eips.ethereum.org/EIPS/eip-7702#security-considerations)  in the  EIP, some of which are listed below.

## Security

- A poorly implemented delegate can *allow a malicious actor to take near complete
control over a signer’s EOA*.
- Attackers can try to front run an EOA’s set-code transaction, so developers must check for signed authorizations using `ecrecover`.
- When upgrading an EOA’s code (delegate address or code therein), care must be taken to manage storage slots since all delegated code (existing and new) will run in the context of the EOA.
- Breaks in atomic sandwich protections which rely on `tx.origin` and reentrancy guards of the style `require(tx.origin == msg.sender)`
- Transaction (gas) sponsors may require their users to post some bond or use reputational mechanisms. This is to guard against the possibility of relayers not being reimbursed for gas due to change in a EOA’s authorized code or asset sweeping.
- Transaction propagation: An EOA’s nonce may be incremented more than once per transaction.
- Transaction propagation:  once an EOA has delegated to code, that code can be called by anyone. It becomes possible to cause transactions from other accounts to become stale. The authors of EIP-7702 recommend that clients do not accept more than one pending transaction from any EOA with a non-zero delegation indicator.

## Backward compatibility

This change is a hard fork and therefore all full nodes must be updated. Light clients that only verify block headers do not need to be updated. Clients and tools that decode transactions, receipts or account state must be updated to handle set-code transactions and the delegation authority flag.

As in Ethereum, this change breaks several invariants that applications and tooling may rely on. On Rootstock they apply as follows:

- An account's balance can decrease as a result of a call made by anyone, not only through a transaction the account itself signed, because delegated code runs in the account's context.
- An EOA's nonce can increase during execution, since delegated code can execute `CREATE`, and it can increase more than once in a single transaction, through authorization processing (step 8 of [Behavior](#behavior)). Nodes therefore SHOULD accept at most one pending transaction from a delegated account (see [Transaction origination](#transaction-origination)).
- `tx.origin == msg.sender` can now hold in a nested call frame through self-sponsoring, which breaks guards of the form `require(tx.origin == msg.sender)`.
- An EOA can hold persistent storage (see step 7 of [Behavior](#behavior)).
- Delegated EOAs carry code. Un-upgraded tooling will see `EXTCODESIZE` return `23` and `EXTCODEHASH` return `keccak256(0xef0100 || address)` for accounts that previously looked like plain EOAs, and tooling that treats any account with code as a contract will classify them as contracts.

## Implementation

The implementation is in the RSKJ repository under the consensus rule `RSKIP545`, in [pull request 3558](https://github.com/rsksmart/rskj/pull/3558), which is not yet merged.

The implementation currently differs from this specification in the following points, which need to be resolved before activation:

- **Recognizing a delegation.** The implementation sets and keeps the delegation authority flag as specified, but recognizes a delegation from the account's code alone: when following a delegation, in step 4 of [Behavior](#behavior), and when checking the sender of a set-code transaction. This specification requires the flag as well.
- **Delegation resolution cost.** The implementation does not charge `DELEGATION_RESOLUTION_COST`.
- **RSKIP-701.** The implementation does not yet include RSKIP-701 or require it for activation, so `SELFDESTRUCT` in a delegated account deletes the account, including its nonce.

## Test cases

### Authorization signing

The authorization below is signed with the well-known test private key `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`, whose address is `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`. It delegates to `address = 0x1234567890123456789012345678901234567890` on chain ID `33` (`0x21`) with `nonce = 0`.

Signing preimage, `MAGIC || rlp([chain_id, address, nonce])`:

```
05d72194123456789012345678901234567890123456789080
```

Message hash, `msg = keccak256(MAGIC || rlp([chain_id, address, nonce]))`:

```
0x84697cf20ff2af42b3c4a186e0daab3adcb423ac8c0df0c21d5d48405fa2f23f
```

Signed authorization tuple, `rlp([chain_id, address, nonce, y_parity, r, s])`:

```
f85a219412345678901234567890123456789012345678908001a01ee45ad498fc0c39f6abb56072a4a6fd35a21053ed245b4bafee8f67b2d3dab1a00c4caf636ce61912cb585918862a7d449fbb4c9daebb3c37c57021656f5a491b
```

Here `y_parity = 1`, `r = 0x1ee45ad498fc0c39f6abb56072a4a6fd35a21053ed245b4bafee8f67b2d3dab1` and `s = 0x0c4caf636ce61912cb585918862a7d449fbb4c9daebb3c37c57021656f5a491b`, with `s <= secp256k1n/2`. Step 3 of [Behavior](#behavior) MUST recover `authority = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`.

### Code reading on a delegated account

Let account `A` be a delegated account whose delegation indicator points to a contract `D` at `0x1234567890123456789012345678901234567890`. The code of `D` is the 9 bytes `0x3860005260206000f3` (`CODESIZE PUSH1 0x00 MSTORE PUSH1 0x20 PUSH1 0x00 RETURN`), which returns the size of the code being executed. Then:

| Operation | Result |
| --- | --- |
| Call `A` (executes the code of `D` in the context of `A`) | Returns `9`: `CODESIZE` sees the code of `D` |
| `EXTCODESIZE(A)` | `23` |
| `EXTCODECOPY(A, …)` | The 23 bytes `0xef01001234567890123456789012345678901234567890` |
| `EXTCODEHASH(A)` | `0x50ee8ea5591e28c2efaa0b2d630b17069144a5c286ff271c822335086104b976` |
| `EXTCODESIZE(D)` | `9` |

The call to `A` also costs `DELEGATION_RESOLUTION_COST` (see [Gas Costs](#gas-costs)).

### Gas used by authorizations

In these cases the sender is not an authority, the transaction's `destination` is an EOA with no code, `value` and `data` are empty, and the access list is empty, so no execution gas is used.

| Case | Expected |
| --- | --- |
| Intrinsic gas with one tuple | `21000 + 25000 = 46000` |
| Intrinsic gas with two tuples | `21000 + 2 * 25000 = 71000` |
| One tuple; `authority` has no code (first delegation) | Gas used `46000` (no refund) |
| One tuple; `authority` is already a delegated account (re-delegation, or clearing with `address = 0x0`) | Gas used `46000 - 9500 = 36500` |
| One tuple, gas limit `45999` | Transaction is invalid (gas limit below intrinsic gas) |
| One tuple, gas limit `46000` | Transaction is valid |

### Skipped tuples and invalid transactions

A tuple that fails any step of [Behavior](#behavior) is skipped: the code and nonce of its `authority` are unchanged, the transaction still executes, and the tuple is still charged `PER_EMPTY_ACCOUNT_COST`. Each of the following MUST cause the tuple to be skipped:

| Tuple condition | Step |
| --- | --- |
| `chain_id` is neither `0` nor the chain ID of the network | 1 |
| `nonce >= 2**64 - 1` | 2 |
| `y_parity` is not `0` or `1` | 3 |
| `r` or `s` is `0` or not less than `secp256k1n` | 3 |
| `s > secp256k1n/2` (`s = secp256k1n/2` is accepted) | 3 |
| The signature recovers to the zero address | 3 |
| `authority` has code and is not a delegated account | 4 |
| `nonce` is not equal to the nonce of `authority` | 5 |

By contrast, the following make the whole set-code transaction invalid:

- `destination` is empty (contract creation).
- `authorization_list` is empty.
- A tuple does not have exactly 6 elements, or `address` is not exactly 20 bytes.
- A tuple field is not canonically RLP-encoded, or exceeds the bounds in [Set-code transaction](#set-code-transaction).

### Ordering and nonces

| Case | Expected |
| --- | --- |
| Self-sponsoring: the sender is the `authority`, transaction nonce `0`, tuple nonce `1` | Tuple applied; sender nonce ends at `2` |
| Self-sponsoring with tuple nonce `0` | Tuple skipped; sender nonce ends at `1` |
| External `authority` with nonce `0`, tuple nonce `0` | Tuple applied; `authority` nonce `1`; sender nonce increases by `1` |
| Two tuples from the same `authority`, nonces `0` and `1`, different addresses | Both applied; the second address is the delegation; `authority` nonce `2` |
| Two identical tuples, nonce `0` | First applied, second skipped (stale nonce); both charged, intrinsic gas `71000` |
| An invalid tuple followed by a valid tuple from another `authority` | Second tuple applied |

### Failed execution

`A` is already a delegated account. A set-code transaction with gas limit `200000` and effective gas price `1` carries one valid tuple from `A`, re-delegating it to a new address, and its `destination` is a contract that is not delegated and whose code is `0xfe` (`INVALID`). Then:

- The receipt status is failure, and execution consumes all remaining gas.
- The new delegation of `A` is in place and the nonce of `A` is incremented: the tuple is not rolled back.
- The refund of `9500` still applies, so the gas used is `200000 - 9500 = 190500` and the sender pays `190500`.

## References

\[1] [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702): Set Code for EOAs

\[2] [RSKIP167](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP167.md): Install Code Precompile

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

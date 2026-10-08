---
rskip: 701
title: SELFDESTRUCT Preserves Account Nonces
description: SELFDESTRUCT only moves the balance of an account delegated under RSKIP545, and any other destroyed account that outlived its creating transaction keeps its non-zero nonce
status: Draft
purpose: Sec
author: IS (@italo-sampaio)
layer: Core
complexity: 2
requires: 545
created: 2026-10-05
---
# SELFDESTRUCT Preserves Account Nonces

|RSKIP          | 701 |
| :------------ |:-------------|
|**Title**      |SELFDESTRUCT Preserves Account Nonces|
|**Created**    |05-OCT-26 |
|**Author**     |IS |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP changes what `SELFDESTRUCT` does to the account it destroys. When `SELFDESTRUCT` executes in the context of an account delegated under RSKIP545, only the balance of the account moves to the beneficiary. Its nonce, its delegation indicator and its storage are kept. Any other destroyed account that outlived the transaction that created it and has a non-zero nonce loses its code, its storage and its balance at the end of the transaction, but keeps its nonce.

## Motivation

Currently `SELFDESTRUCT` deletes the account in whose context it executes, at the end of the transaction. The account node and every node under it are removed from the Unitrie, so the nonce of the account is removed with its code, its storage and its balance. A later value transfer or call to the address creates a fresh account with nonce zero.

RSKIP545 [1] lets an externally owned account delegate its code to a contract. The delegated code runs in the context of the account and can execute `SELFDESTRUCT`. At the end of that transaction the account is deleted, and its nonce with it. RSKIP545 checks only the nonce before it processes an authorization, so every authorization the account ever signed becomes valid again, and anyone can replay it in a new set-code transaction. RSKIP545 follows EIP-7702 [2], which was written for a chain where `SELFDESTRUCT` only deletes an account created in the same transaction [3]. RSKIP545 does not mention `SELFDESTRUCT`.

The nonce reset has a second consequence, which does not depend on RSKIP545. A contract deleted by `SELFDESTRUCT` can be replaced in a later block by a contract with different code at the same address. RSKIP131 [4] refuses the redeployment only within the same block. The RSKIP125 [5] collision rule refuses a deployment at an address whose nonce is not zero, and a deleted account has nonce zero. This is the pattern of the Tornado Cash governance attack of May 2023, where a destroyed deployer was redeployed by `CREATE2` and recreated the proposal contract at its original address with different code [6].

This RSKIP keeps the nonce in both cases. A delegated account is not deleted at all. Any other destroyed account keeps its nonce when it outlived the transaction that created it and its nonce is not zero, so the RSKIP125 collision rule refuses any later deployment at its address.

## Specification

### Definitions

The **executing account** of a frame is the account at the address that `ADDRESS` returns in that frame, the account whose balance and storage the frame operates on. Under `DELEGATECALL` and `CALLCODE` that is the executing account of the calling frame, not the account that holds the code.

A **delegated account** is an account that carries the delegation authority flag defined in RSKIP545 and whose code is exactly the 23-byte delegation indicator `0xef0100 || address`. An account whose code is a delegation indicator but that does not carry the flag is not a delegated account.

An account is **created in the current transaction** when the current transaction is a contract creation for its address, or when a `CREATE` or `CREATE2` executed in the current transaction created a contract at its address, including when the init code executes `SELFDESTRUCT`. This holds even when the address held a balance before the transaction. A creation inside a frame that reverts is discarded with the frame, as before activation.

The **marked accounts** of a transaction are the accounts that `SELFDESTRUCT` adds for deletion during the transaction. A frame that reverts or ends in an exceptional halt, and a creation that fails after its init code returns, discard the marks added within them, as before activation.

**Moving the balance** to the beneficiary is the transfer that `SELFDESTRUCT` performs before activation: a zero balance moves nothing and does not create the beneficiary, and a non-zero balance creates the beneficiary when it does not exist.

Where this specification says *as before activation*, the behaviour is the one in force at the activation block, and this RSKIP does not change it.

### SELFDESTRUCT in a delegated account

From the activation of this RSKIP, when `SELFDESTRUCT` executes and the executing account is a delegated account, the following applies.

1. **Balance.** `SELFDESTRUCT` moves the balance of the executing account to the beneficiary. When the beneficiary is the executing account itself, its balance is unchanged.
2. **Account.** `SELFDESTRUCT` does not change the nonce, the code or the storage of the executing account. Writes made earlier in the transaction stand.
3. **Marking.** The executing account is not added to the marked accounts.
4. **Execution and gas.** The frame ends successfully, as it does after `STOP`, and the gas charged for `SELFDESTRUCT` is the same as before activation.

A delegation installed by an authorization of the current transaction makes the account a delegated account for the rest of the transaction.

### SELFDESTRUCT in any other account

When the executing account is not a delegated account, `SELFDESTRUCT` moves the balance to the beneficiary, adds the executing account to the marked accounts, ends the frame successfully and charges gas as before activation. When the beneficiary is the executing account itself, its balance becomes zero, as before activation.

### Marked accounts at the end of the transaction

At the end of the transaction each marked account is processed in one of two ways.

1. **Deleted.** An account created in the current transaction is deleted, as before activation. An account whose nonce is zero is also deleted, as before activation. The account node and every node under it are removed from the Unitrie.
2. **Cleared.** Any other account loses its code, its storage and its balance, and keeps its nonce. Every node under the account node is removed from the Unitrie. That covers the code node, the storage cells and the storage placeholder node, in the layout that RSKIP108 [7] describes. The account node remains with the same nonce, the same flags and a zero balance.

The nonce used in both cases is the nonce of the account at the end of the transaction. A balance the account received after it was marked is removed with the rest of its balance, as before activation.

Every marked account, whether deleted or cleared, is recorded as deleted for the RSKIP131 check of the later transactions of the block.

### What does not change

- The refund of 24,000 gas for each marked account is unchanged, and so is the cap on refunds. A delegated account is never marked, so it earns no refund.
- The RSKIP131 check refuses a `CREATE2` at the address of an account deleted by an earlier transaction of the same block. A delegated account is never marked, so RSKIP131 never treats it as deleted.
- The RSKIP125 collision rule is unchanged. It refuses a deployment at the address of a cleared account because the nonce of that account is not zero.
- `SELFDESTRUCT` inside a static call causes an exceptional halt, as before activation.

### Activation

This change requires a network upgrade. It is proposed for the Cardamom network upgrade (RSKIP694), together with RSKIP545. Before activation `SELFDESTRUCT` and the end-of-transaction processing of marked accounts keep the behaviour in force today, so that historical blocks replay to the same state.

## Rationale

**A delegated account keeps its nonce, code and storage.** The nonce protects the authorizations the account has signed. The code holds the delegation the account chose, and the storage belongs to the account rather than to the delegate. Only the balance moves, so a beneficiary other than the account itself receives the same amount as before activation.

**Delegation is recognised by the flag.** A contract whose code is a delegation indicator could be deployed before RSKIP544 [8]. Under RSKIP545 a call to it runs the code of the named address in its context, but the contract never signed an authorization, so it has no authorization to protect, and it is marked like any other contract.

**The kept nonce blocks redeployment.** The RSKIP125 collision rule already refuses a deployment at an address whose nonce is not zero. Keeping the nonce is therefore enough, and no new marker is needed in the account.

**Some accounts are still deleted.** The address of an account created in the current transaction can be used again in a later block, as on Ethereum after EIP-6780 [3]. No other transaction ever saw its code. A marked account with nonce zero that was not created in the current transaction has one of two origins: a contract creation transaction, or a `CREATE` executed before RSKIP125. Both derive the address from the nonce of the creator, and neither sets the nonce of the new contract to one. Such an account never executed `CREATE`, or its nonce would not be zero. Its creator is an externally owned account, whose nonce never decreases, or a contract whose own address rests on the same argument, so no later creation can derive the same address. Keeping such an account would add state and protect nothing.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated. Blocks produced before activation are validated with the previous behaviour.

RSKIP545 must not activate before this RSKIP. Without it, a delegated account that executes `SELFDESTRUCT` loses its nonce, and its authorizations can be replayed.

Delegated accounts exist only once RSKIP545 is active, so no existing account is affected by the delegated account rule. The following changes apply to a cleared account, that is, an account marked after activation that was not created in the marking transaction and whose nonce at the end of that transaction is not zero.

- `EXTCODEHASH` of the address returns the keccak256 hash of empty data. Before activation it returns zero until a value transfer or a call recreates the account.
- A `CALL` to the address does not pay the 25,000 gas charged for a call to an account that does not exist. A `SELFDESTRUCT` that names the address as beneficiary does not pay the 25,000 gas charged for a beneficiary that does not exist.
- No contract can be deployed at the address again. Before activation a deployment was possible from the next block on. A balance sent to the address after destruction cannot be recovered by redeploying a contract there.
- Explorers, indexers and wallets see the address as an account with a nonce and no code.

Accounts deleted before activation are not affected.

## Test Cases

In every case `D` is a contract whose code stores `42` in slot `0` and then executes `SELFDESTRUCT` with the beneficiary taken from the call data. `C` is a contract with the same code as `D`. `B` is an account that exists before each case. "After activation" refers to blocks after the activation of both RSKIP545 and this RSKIP.

1. Account `A` signs an authorization for `D` with nonce `0`, holds a balance, and a set-code transaction installs the delegation. In a later transaction, an account calls `A` with beneficiary `B`. After the transaction `A` has nonce `1`, the code `0xef0100 || D`, the value `42` in slot `0` and a zero balance. `B` has received the former balance of `A`.
2. After case 1, a set-code transaction carries the authorization of case 1 again. The authorization is skipped, because its nonce `0` differs from the nonce `1` of `A`.
3. As case 1, with `A` itself as the beneficiary. After the transaction the balance of `A` is unchanged.
4. `A` is delegated to a contract `E` whose code forwards its call data by `DELEGATECALL` to `D`. An account calls `A` with beneficiary `B`. The outcome is as in case 1.
5. A single set-code transaction sent by an account `S` other than `A` carries the authorization of `A` for `D` with nonce `0` and calls `A` with beneficiary `B`. After the transaction `A` has nonce `1`, the code `0xef0100 || D`, the value `42` in slot `0` and a zero balance, and `B` has received the former balance of `A`.
6. Case 1 is compared with a transaction in which an account calls `C`, deployed in an earlier block, with the same beneficiary `B`. `SELFDESTRUCT` charges the same gas in both. The transaction that calls `C` receives a refund of 24,000 gas, limited to half of the gas it used, and the transaction of case 1 receives none.
7. Factory `F` deploys `C` with `CREATE2` and salt `s`, so `C` starts with nonce `1`. In a later transaction an account calls `C` with beneficiary `B`. After the transaction `C` has nonce `1`, a zero balance, no code and no storage, and `EXTCODEHASH` of `C` returns the keccak256 hash of empty data. In a later block `F` repeats the `CREATE2` with salt `s` and the same init code. The `CREATE2` pushes zero and consumes its gas. Before activation `C` is deleted and the second creation succeeds.
8. After case 7, a contract calls `C`. The call is not charged the 25,000 gas for an account that does not exist. Before activation it is charged.
9. A transaction makes `F` deploy `C` with `CREATE2` and salt `s`, then call `C` with beneficiary `B`. After the transaction `C` does not exist. The same `CREATE2` by `F` in a later transaction of the same block pushes zero under RSKIP131, and the same `CREATE2` in the next block succeeds.
10. A contract creation transaction deploys `C`, so `C` has nonce `0`. `C` never creates a contract. In a later transaction an account calls `C` with beneficiary `B`. After the transaction `C` does not exist, as before activation.
11. A contract creation transaction deploys `G`, a contract whose code executes `CREATE` and then `SELFDESTRUCT` with the beneficiary taken from the call data, so `G` has nonce `0`. In a later transaction an account calls `G` with beneficiary `B`. The `CREATE` raises the nonce of `G` to `1` before `SELFDESTRUCT` marks it. After the transaction `G` has nonce `1`, no code, no storage and a zero balance. Before activation `G` is deleted.
12. As case 7, but after `C` executes `SELFDESTRUCT`, another contract of the same transaction executes `SELFDESTRUCT` naming `C` as beneficiary, which moves a balance to `C` without running its code. After the transaction the balance of `C` is zero, and `B` has received only the balance `C` held when its own `SELFDESTRUCT` executed.
13. A contract `P` deployed with `CREATE` in a block where RSKIP125 is active and RSKIP544 is not has code of exactly `0xef0100 || D` and nonce `1`. After activation an account calls `P` with beneficiary `B`. The call runs the code of `D` in the context of `P`. `P` does not carry the delegation authority flag, so it is marked and cleared as in case 7.
14. Cases 7 to 12 replayed in blocks before activation produce the same state as before this RSKIP.

## Implementation

TBD

## Security Considerations

This RSKIP closes the replay of RSKIP545 authorizations through `SELFDESTRUCT`. After activation no consensus path lowers the nonce of an account that outlived its creating transaction.

It also closes the replacement of a destroyed contract by different code at the same address, for every contract that is cleared after activation. An address that a user or a contract trusts can no longer receive new code through `SELFDESTRUCT` followed by a redeployment.

`SELFDESTRUCT` still removes code. A contract that calls a library which can be destroyed still loses the library.

## References

[1] RSKIP545 Set Code for EOAs https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP545.md

[2] EIP-7702 Set Code for EOAs https://eips.ethereum.org/EIPS/eip-7702

[3] EIP-6780 SELFDESTRUCT only in same transaction https://eips.ethereum.org/EIPS/eip-6780

[4] RSKIP131 Preventing CREATE2-after-SUICIDE in the same block https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP131.md

[5] RSKIP125 Create2 https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP125.md

[6] Explained: The Tornado Cash Hack (May 2023) https://www.halborn.com/blog/post/explained-the-tornado-cash-hack-may-2023

[7] RSKIP108 More Efficient Unitrie Key Mapping https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP108.md

[8] RSKIP544 Reject new contract code starting with the 0xEF byte https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP544.md

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

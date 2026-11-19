---
rskip: 701
title: SELFDESTRUCT Preserves Account Nonces
description: A destroyed account that outlived its creating transaction keeps its non-zero nonce
status: Draft
purpose: Sec
author: IS (@italo-sampaio)
layer: Core
complexity: 2
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

This RSKIP changes what `SELFDESTRUCT` does to the account it destroys. A destroyed account that outlived the transaction that created it and has a non-zero nonce loses its code, its storage and its balance at the end of the transaction, but keeps its nonce.

## Motivation

Currently `SELFDESTRUCT` deletes the account in whose context it executes, at the end of the transaction. The account node and every node under it are removed from the Unitrie, so the nonce of the account is removed with its code, its storage and its balance. A later value transfer or call to the address creates a fresh account with nonce zero.

A contract deleted by `SELFDESTRUCT` can therefore be replaced in a later block by a contract with different code at the same address. RSKIP131 [1] refuses the redeployment only within the same block. The RSKIP125 [2] collision rule refuses a deployment at an address whose nonce is not zero, and a deleted account has nonce zero. This is the pattern of the Tornado Cash governance attack of May 2023, where a destroyed deployer was redeployed by `CREATE2` and recreated the proposal contract at its original address with different code [3].

This RSKIP keeps the nonce. A destroyed account keeps its nonce when it outlived the transaction that created it and its nonce is not zero, so the RSKIP125 collision rule refuses any later deployment at its address.

## Specification

### Definitions

The **executing account** of a frame is the account at the address that `ADDRESS` returns in that frame, the account whose balance and storage the frame operates on. Under `DELEGATECALL` and `CALLCODE` that is the executing account of the calling frame, not the account that holds the code.

An account is **created in the current transaction** when the current transaction is a contract creation for its address, or when a `CREATE` or `CREATE2` executed in the current transaction created a contract at its address, including when the init code executes `SELFDESTRUCT`. This holds even when the address held a balance before the transaction. A creation inside a frame that reverts is discarded with the frame, as before activation.

The **marked accounts** of a transaction are the accounts that `SELFDESTRUCT` adds for deletion during the transaction. A frame that reverts or ends in an exceptional halt, and a creation that fails after its init code returns, discard the marks added within them, as before activation.

**Moving the balance** to the beneficiary is the transfer that `SELFDESTRUCT` performs before activation: a zero balance moves nothing and does not create the beneficiary, and a non-zero balance creates the beneficiary when it does not exist.

Where this specification says *as before activation*, the behaviour is the one in force at the activation block, and this RSKIP does not change it.

### SELFDESTRUCT marks the executing account

`SELFDESTRUCT` moves the balance to the beneficiary, adds the executing account to the marked accounts, ends the frame successfully and charges gas as before activation. When the beneficiary is the executing account itself, its balance becomes zero, as before activation.

### Marked accounts at the end of the transaction

At the end of the transaction each marked account is processed in one of two ways.

1. **Deleted.** An account created in the current transaction is deleted, as before activation. An account whose nonce is zero is also deleted, as before activation. The account node and every node under it are removed from the Unitrie.
2. **Cleared.** Any other account loses its code, its storage and its balance, and keeps its nonce. Every node under the account node is removed from the Unitrie. That covers the code node, the storage cells and the storage placeholder node, in the layout that RSKIP108 [4] describes. The account node remains with the same nonce, the same flags and a zero balance.

The nonce used in both cases is the nonce of the account at the end of the transaction. A balance the account received after it was marked is removed with the rest of its balance, as before activation.

Every marked account, whether deleted or cleared, is recorded as deleted for the RSKIP131 check of the later transactions of the block.

The following table summarises how these rules apply to each kind of account. It follows from the rules above and adds none.

| Account | Before activation | After activation |
| :--- | :--- | :--- |
| Created in the current transaction | Deleted | Deleted |
| Created in an earlier transaction by `CREATE` or `CREATE2` after RSKIP125, so its nonce is at least `1` | Deleted, and its nonce returns to `0` | Cleared, and it keeps its nonce |
| Created in an earlier transaction by a contract creation transaction or by a `CREATE` before RSKIP125, with nonce `0` at the end of the transaction | Deleted | Deleted |
| Created as in the previous row, with a nonce of at least `1` at the end of the transaction because it executed `CREATE` | Deleted, and its nonce returns to `0` | Cleared, and it keeps its nonce |
| Externally owned account | Never marked, because it has no code to execute `SELFDESTRUCT` | Never marked |

### What does not change

- The refund of 24,000 gas for each marked account is unchanged, and so is the cap on refunds.
- The RSKIP131 check refuses a `CREATE2` at the address of an account deleted by an earlier transaction of the same block.
- The RSKIP125 collision rule is unchanged. It refuses a deployment at the address of a cleared account because the nonce of that account is not zero.
- `SELFDESTRUCT` inside a static call causes an exceptional halt, as before activation.

### Activation

This change requires a network upgrade. Before activation `SELFDESTRUCT` and the end-of-transaction processing of marked accounts keep the behaviour in force today, so that historical blocks replay to the same state.

## Rationale

**The kept nonce blocks redeployment.** The RSKIP125 collision rule already refuses a deployment at an address whose nonce is not zero. Keeping the nonce is therefore enough, and no new marker is needed in the account.

**Some accounts are still deleted.** The address of an account created in the current transaction can be used again in a later block, as on Ethereum after EIP-6780 [5]. No other transaction ever saw its code. A marked account with nonce zero that was not created in the current transaction has one of two origins: a contract creation transaction, or a `CREATE` executed before RSKIP125. Both derive the address from the nonce of the creator, and neither sets the nonce of the new contract to one. Such an account never executed `CREATE`, or its nonce would not be zero. Its creator is an externally owned account, whose nonce never decreases, or a contract whose own address rests on the same argument, so no later creation can derive the same address. Keeping such an account would add state and protect nothing.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated. Blocks produced before activation are validated with the previous behaviour.

The following changes apply to a cleared account, that is, an account marked after activation that was not created in the marking transaction and whose nonce at the end of that transaction is not zero.

- `EXTCODEHASH` of the address returns the keccak256 hash of empty data. Before activation it returns zero until a value transfer or a call recreates the account.
- A `CALL` to the address does not pay the 25,000 gas charged for a call to an account that does not exist. A `SELFDESTRUCT` that names the address as beneficiary does not pay the 25,000 gas charged for a beneficiary that does not exist.
- No contract can be deployed at the address again. Before activation a deployment was possible from the next block on. A balance sent to the address after destruction cannot be recovered by redeploying a contract there.
- Explorers, indexers and wallets see the address as an account with a nonce and no code.

The `suicide` trace is still emitted for every `SELFDESTRUCT`, as on Ethereum after EIP-6780. It records the execution of the opcode and the balance transfer, not the removal of the account. A cleared account remains in the state after it.

Accounts deleted before activation are not affected.

## Test Cases

In every case `C` is a contract whose code stores `42` in slot `0` and then executes `SELFDESTRUCT` with the beneficiary taken from the call data. `B` is an account that exists before each case.

1. Factory `F` deploys `C` with `CREATE2` and salt `s`, so `C` starts with nonce `1`. In a later transaction an account calls `C` with beneficiary `B`. After the transaction `C` has nonce `1`, a zero balance, no code and no storage, and `EXTCODEHASH` of `C` returns the keccak256 hash of empty data. In a later block `F` repeats the `CREATE2` with salt `s` and the same init code. The `CREATE2` pushes zero and consumes its gas. Before activation `C` is deleted and the second creation succeeds.
2. After the transaction of case 1 that calls `C`, and before `F` repeats the `CREATE2`, a contract calls `C`. The call is not charged the 25,000 gas for an account that does not exist. Before activation it is charged.
3. A transaction makes `F` deploy `C` with `CREATE2` and salt `s`, then call `C` with beneficiary `B`. After the transaction `C` does not exist. The same `CREATE2` by `F` in a later transaction of the same block pushes zero under RSKIP131, and the same `CREATE2` in the next block succeeds.
4. A contract creation transaction deploys `C`, so `C` has nonce `0`. `C` never creates a contract. In a later transaction an account calls `C` with beneficiary `B`. After the transaction `C` does not exist, as before activation.
5. A contract creation transaction deploys `G`, a contract whose code executes `CREATE` and then `SELFDESTRUCT` with the beneficiary taken from the call data, so `G` has nonce `0`. In a later transaction an account calls `G` with beneficiary `B`. The `CREATE` raises the nonce of `G` to `1` before `SELFDESTRUCT` marks it. After the transaction `G` has nonce `1`, no code, no storage and a zero balance. Before activation `G` is deleted.
6. As case 1, but after `C` executes `SELFDESTRUCT`, another contract of the same transaction executes `SELFDESTRUCT` naming `C` as beneficiary, which moves a balance to `C` without running its code. After the transaction the balance of `C` is zero, and `B` has received only the balance `C` held when its own `SELFDESTRUCT` executed.
7. Cases 1 to 6 replayed in blocks before activation produce the same state as before this RSKIP.

## Implementation

TBD

## Security Considerations

This RSKIP closes the replacement of a destroyed contract by different code at the same address, for every contract that is cleared after activation. After activation no consensus path lowers the nonce of an account that outlived its creating transaction, so the RSKIP125 collision rule keeps refusing a deployment at the address of a cleared contract. An address that a user or a contract trusts can no longer receive new code through `SELFDESTRUCT` followed by a redeployment.

`SELFDESTRUCT` still removes code. A contract that calls a library which can be destroyed still loses the library.

## References

[1] RSKIP131 Preventing CREATE2-after-SUICIDE in the same block https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP131.md

[2] RSKIP125 Create2 https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP125.md

[3] Explained: The Tornado Cash Hack (May 2023) https://www.halborn.com/blog/post/explained-the-tornado-cash-hack-may-2023

[4] RSKIP108 More Efficient Unitrie Key Mapping https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP108.md

[5] EIP-6780 SELFDESTRUCT only in same transaction https://eips.ethereum.org/EIPS/eip-6780

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

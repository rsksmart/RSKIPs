---
rskip: 692
title: Direct Precompile Call Failure Semantics
description: A transaction whose recipient is a precompiled contract fails as an exceptional halt when the precompiled contract fails
status: Draft
purpose: Sec
author: IS (@italo-sampaio)
layer: Core
complexity: 2
created: 2026-09-15
---
# Direct Precompile Call Failure Semantics

|RSKIP          | 692 |
| :------------ |:-------------|
|**Title**      |Direct Precompile Call Failure Semantics|
|**Created**    |15-SEP-26 |
|**Author**     |IS |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP specifies the outcome of a transaction whose recipient is a precompiled contract when that precompiled contract fails. After activation the transaction ends in an exceptional halt. The receipt reports failure, the transaction consumes its gas limit before refunds, the state changes made by the precompiled contract are discarded, and the logs it emitted are not recorded.

## Motivation

Currently a precompiled contract that fails while executing as the recipient of a transaction is reported as successful. The receipt status is `1`. The gas used in the receipt is the cost declared by the precompiled contract for its input plus the intrinsic cost of the transaction, although the sender is charged the full gas limit. Any state the precompiled contract wrote before failing is committed. Any log it emitted before failing is recorded in the receipt and in the bloom filters.

- A receipt states success and a partial gas figure for a transaction that failed and paid its full gas limit. Wallets, explorers and libraries that read the status field show a failed call as successful.
- The gas used of the block is lower than the gas consumed by its transactions. A following transaction can therefore be included in a block that is already full.
- A Bridge event can appear in the receipt of a call that failed, while the Bridge state the event describes was never committed.
- A state change made by a precompiled contract before it fails persists.

RSKIP197 fixed the same family of defect for a precompiled contract called from a contract. Since the Iris network upgrade such a call behaves as a failed `CALL` for the caller: the caller sees a failure and the nested state changes are rolled back. A precompiled contract called directly by a transaction was not covered by RSKIP197, and the direct path still behaves as described above.

The defect is visible on the Bridge in particular, because the Bridge fails for ordinary reasons such as an unauthorized caller or an unknown method. A call to `updateCollections` from an account that is not a federation member is recorded on testnet with status `1`.

## Specification

### Definitions

A **direct call** is a transaction whose recipient is a precompiled contract. The precompiled contract executes at the top level, with no calling contract. The Remasc transaction that closes each block is a direct call to the Remasc contract and is covered by this definition, although no failure of it is reachable in a valid block. Any other transaction sent to the Remasc address is an ordinary direct call.

A **nested call** is a precompiled contract invoked by a contract through a call opcode. Nested calls are covered by RSKIP197 and are not changed by this RSKIP.

A **precompile failure** is a precompiled contract that ends its execution by raising an error rather than by returning output. A precompiled contract that returns an error code as part of its output has not failed in this sense. A resource failure of the node itself, such as exhausting its memory or its stack, is not a precompile failure.

### Failing direct calls

From the activation of this RSKIP, a direct call whose precompiled contract fails ends in an exceptional halt of the transaction, with the following effects.

1. **Receipt status.** The receipt reports failure. The status field is `0`.
2. **Gas.** The transaction consumes its gas limit before refunds. Refunds apply as for any exceptional halt. The gas used written to the receipt, counted in the gas used of the block and shown in traces is the gas consumed minus the applied refunds.
3. **State.** Every state change made by the precompiled contract during the call is discarded. The nonce increment and the fee paid by the sender remain, as for any exceptional halt. The value sent with the transaction stays with the sender, as it does before activation.
4. **Logs.** No log emitted by the precompiled contract during the call is recorded. It must not appear in the receipt, in the receipt bloom filter or in the block bloom filter.

Any effect of the transaction not listed here is the same as for an exceptional halt of a transaction whose recipient is a contract.

### Direct calls with insufficient gas

A direct call that cannot pay the cost the precompiled contract declares for its input is a precompile failure under this RSKIP, although the precompiled contract does not execute. Before activation such a call already reports status `0`, gas used equal to its gas limit, and no state change or log. After activation it ends in an exceptional halt, so the refunds that apply to an exceptional halt apply to it as stated in the Gas clause.

### What does not change

- Nested calls keep the behaviour specified by RSKIP197.
- A precompiled contract that completes and returns an error code is a successful call.
- A failure raised before the precompiled contract starts executing, such as while it computes the cost it declares for its input, is not a precompile failure under this RSKIP. Its outcome does not change.
- Transactions that end by an exceptional halt of the EVM already report failure and consume their gas as described here. Their outcome does not change.
- Transactions that end by `REVERT` report failure and return their unused gas. Their outcome does not change.

### JSON-RPC interface

Nodes should return an error from `eth_call` and `eth_estimateGas` for a direct call whose precompiled contract fails. They should not return an empty result or a gas estimate for it. The error message should be fixed and should not include the message of the error raised by the precompiled contract. This part does not require a network upgrade, and nodes can adopt it at any time.

### Activation

This change requires a network upgrade. Before activation the behaviour described in the Motivation is kept, so that historical blocks replay to the same state.

## Rationale

**A failing precompiled contract is an exceptional halt.** In Ethereum an error raised by a precompiled contract is an error of the call frame that invoked it. At the top level that means status `0`, all gas consumed, state reverted and no logs. EIP-196 states it for the `alt_bn128` contracts: the call "fails on invalid input and consumes all gas provided". This RSKIP applies the same outcome to a direct call in RSK, so that a precompiled contract and a contract fail in the same way.

**The transaction consumes its gas limit.** Under RSKIP197 a nested call that fails costs the caller only the cost the precompiled contract declared, because the caller continues executing and can act on the failure. A direct call has no caller to continue. The transaction has halted, and a halted transaction consumes its gas limit, as any EVM exceptional halt does. Before activation the sender was already charged the full gas limit for a failing direct call. This RSKIP changes the charge only by the refunds that apply to an exceptional halt. Without such refunds the charge is unchanged. This RSKIP makes the receipt and the block report what was charged, which also closes the gap that let a following transaction fit in a block that was full.

**Refunds are not restated.** The Gas clause defers to the refund rules that apply to any exceptional halt at the time of activation. Stating them here would duplicate rules defined elsewhere and would have to be kept in step with them.

**A call that cannot pay the declared cost is a failure.** The precompiled contract does not execute in that case, and the sender is charged the full gas limit. Leaving that case out of the rule would keep a path that loses the refunds every other exceptional halt keeps.

**Relationship to RSKIP197.** RSKIP197 specified the failure of a nested call and left the direct call as it was. The two paths have differed since Iris: a nested failure is rolled back and reported to the caller, while a direct failure is committed and reported as success. This RSKIP specifies the direct call so that both paths roll back the state changes of a failing precompiled contract. Logs of a failing nested call are outside this RSKIP.

**Bridge state.** On mainnet and testnet, no current Bridge method lets an unprivileged sender commit state before the method fails. The state rollback therefore removes a latent condition rather than a live one, and it protects any future Bridge method that writes state during the call. Bridge events are a different matter. Before activation an event emitted by the Bridge is recorded even when the call fails, so a Bridge event in a receipt does not prove that the Bridge state it describes was committed. After activation events are recorded only for calls that succeed. Indexers that need the committed Bridge state should read it through the Bridge view methods rather than infer it from receipts.

**The JSON-RPC interface reports the failure.** Wallets call `eth_call` and `eth_estimateGas` before they send a transaction. Currently both methods report a failing direct call as successful, and `eth_estimateGas` returns 44,064 for the call in test case 1. After activation a transaction sent with that gas limit fails and consumes it. Therefore the failure should be reported before the transaction is sent.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated. Blocks produced before activation are validated with the previous behaviour.

After activation a block that contains a failing direct call can differ in the following consensus fields.

- The status of the receipt, and its gas used where the receipt encoding carries it.
- The cumulative gas of that receipt and of every later receipt in the block.
- The logs and the bloom filter of the receipt, and the logs bloom of the block.
- The receipts root and the gas used of the block.
- The state root, when the failing call had written state, when the fee differs, or when the set of transactions in the block differs.

For a transaction without refunds the fee is unchanged, because the sender was already charged the full gas limit. The value sent with the transaction was never transferred and still is not.

The following observable changes apply to failing direct calls.

- Receipts report status `0` instead of `1`, a higher gas used, and no logs.
- The gas used of a block that contains such a transaction is higher, so fewer transactions may fit in it.
- Libraries that reject a transaction on status `0` now reject failing direct calls. Explorers show them as failed.
- Consumers that read Bridge events from the receipts of failing calls no longer see them.
- `eth_call` and `eth_estimateGas` return an error for a failing direct call instead of an empty result and a gas estimate. JSON-RPC clients that treat an empty result as success need to handle the error.

Contracts are not affected. Nested calls behave as before.

## Test Cases

Gas price is 1 in every case, so the fee equals the gas used.

1. A transaction with gas limit 200,000 sends the data `0xdeadbeef` to the Bridge. The data matches no Bridge method, so the Bridge fails. Before activation the receipt status is `1` and the gas used is 44,064. That is the 23,000 the Bridge declares for data it cannot parse plus the intrinsic cost of 21,064. After activation the receipt status is `0`, the gas used is 200,000, and the fee paid is 200,000 in both cases.
2. A block has a gas limit of 250,000. The first transaction is the call from case 1 with gas limit 200,000. The second is a value transfer with gas limit 100,000. Before activation both transactions are included and the block gas used is below 250,000. After activation only the first transaction is included and the block gas used is 200,000.
3. A transaction with gas limit 100,000 calls a contract whose code is the invalid opcode `0xfe`. Before and after activation the receipt status is `0`, the gas used is 100,000 and the fee is 100,000.
4. A transaction calls a contract whose code executes `REVERT`. Before and after activation the receipt status is `0` and the gas used and the fee are unchanged.
5. A transaction calls a precompiled contract that writes a storage value and then fails. Before activation the value is present in the state after the block. After activation it is absent.
6. A transaction calls a precompiled contract that emits a log and then fails. Before activation the log is in the receipt and the receipt bloom filter is set. After activation the receipt has no logs and the receipt bloom filter is zero.
7. A transaction calls a precompiled contract that writes a storage value, emits a log and then fails. Before activation the value is present in the state after the block and the log is in the receipt. After activation the value is absent, the receipt has no logs and the receipt bloom filter is zero.
8. A transaction with no authorization list calls a precompiled contract with a gas limit below the cost the contract declares for the input. Before and after activation the receipt status is `0`, the gas used and the fee are the gas limit, and no state change or log is recorded.
9. Where RSKIP545 is active, a set-code transaction with gas limit 100,000 carries one valid authorization whose authority is not empty. RSKIP545 grants a refund of 9,500 for it, which is `PER_EMPTY_ACCOUNT_COST` minus `PER_AUTH_BASE_COST`. The transaction calls a precompiled contract that fails. Before activation the receipt status is `1` and the fee is 100,000. After activation the authorization is processed, the receipt status is `0`, and the gas used and the fee are 90,500.
10. Where RSKIP545 is active, a set-code transaction with gas limit 60,000 carries one authorization that earns the same refund as in case 9 and sends the data `0xdeadbeef` to the Bridge. The limit covers the intrinsic cost of 46,064, which includes the 25,000 charged for the authorization. It does not cover the 23,000 the Bridge declares for the input. Before activation the receipt status is `0` and the gas used and the fee are 60,000. After activation the receipt status is `0`, and the gas used and the fee are 50,500.
11. A node receives `eth_call` and `eth_estimateGas` requests for the call of case 1. Both methods return an error.

## Implementation

TBD

## Security Considerations

This RSKIP closes two conditions. A precompiled contract that writes state and then fails can no longer commit that state through a direct call. The gas used field of a block can no longer understate the gas its transactions paid for, so a failing direct call cannot be used to include a transaction in a full block.

## References

[1] RSKIP197 Fix Precompile Calls Not Conforming With CALL Semantics https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP197.md

[2] RSKIP545 Implement EIP-7702 Account Abstraction in Rootstock https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP545.md

[3] RSKIP197 implementation https://github.com/rsksmart/rskj/pull/1392

[4] EIP-196 Precompiled contracts for addition and scalar multiplication on the elliptic curve alt_bn128 https://eips.ethereum.org/EIPS/eip-196

[5] Ethereum execution specifications, Prague, `process_message_call` https://github.com/ethereum/execution-specs

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---
rskip: 555
title: EVM state access gas costs (EIP-2929 and EIP-2200)
description: Introduce cold/warm state access pricing and net gas metering for SSTORE
status: Draft
purpose: Sec, Sca
author: FML
layer: Core
complexity: 3
created: 2026-04-14
---
# EVM state access gas costs (EIP-2929 and EIP-2200)

|RSKIP          |555        |
| :------------ |:-------------|
|**Title**      |EVM state access gas costs (EIP-2929 and EIP-2200) |
|**Created**    |14-APR-2026 |
|**Author**     |FML |
|**Purpose**    |Sec, Sca|
|**Layer**      |Core|
|**Complexity** |3|
|**Status**     |Draft|

The following RSKIP is an adaptation of [EIP-2929][eip2929] and [EIP-2200][eip2200].

## Abstract

This RSKIP adopts Ethereum's cold/warm pricing for state access ([EIP-2929][eip2929]) together with the net gas metering for `SSTORE` that it builds on ([EIP-2200][eip2200]). The first access to an address or storage slot in a transaction pays a *cold* cost. Later accesses to the same address or slot in that transaction pay a much smaller *warm* cost. `SSTORE` pricing starts to take into account the value the slot held when the transaction began. Both changes activate together under a single consensus rule, `RSKIP555`.

## Motivation

Rootstock's state access prices predate Istanbul: `SLOAD` costs 200 gas, `BALANCE` and `EXTCODEHASH` cost 400. Ethereum repriced them in 2019 and again in 2021, and since Berlin every client, gas estimator, compiler assumption and audited contract pattern has been written against the cold/warm model. Anything that reasons about gas instead of merely spending it has to special-case Rootstock.

The flat price is also wrong as a resource price. The first read of a slot in a transaction walks the unitrie and may hit disk. A repeat read is a memory lookup. At 200 gas, a cold state read is one of the cheapest ways to consume node time per unit of gas, and the imbalance grows with the state.

This bears directly on the block gas limit, which bounds how much state a single block can force a node to read:

| Block gas limit | Cold state reads per block |
| :-------------- | -------------------------: |
| 6.8M, today | `6.8M / 200` = ~34K |
| 12M, current pricing | `12M / 200` = ~60K |
| 12M, with this RSKIP | `12M / 2100` = ~5.7K |

Raising the limit to around 12M without repricing state access would increase the worst-case random disk I/O per block by about 76%. With this RSKIP the same limit lands at roughly a sixth of today's worst case. If both ship, this RSKIP should activate first or at the same height.

## Specification

This RSKIP requires a hard fork. Everything below activates under a single new consensus rule, `RSKIP555`, at the block height set by the network upgrade that includes it. The target upgrade is Cardamom.

[EIP-1884][eip1884] is **not** adopted as an intermediate step. Its repricings of `SLOAD`, `BALANCE` and `EXTCODEHASH` are replaced by the costs below at the same height, and its `SELFBALANCE` opcode already exists on Rootstock under RSKIP-151.

### 1. Access sets

Two sets are maintained for the duration of a single transaction:

- `accessed_addresses`, a set of addresses.
- `accessed_storage_keys`, a set of `(address, storage key)` pairs.

Both are created when transaction execution begins and discarded when it ends. They are **shared across all call frames** of the transaction: a nested call inherits the sets from its caller and adds to them.

### 2. Pre-warmed entries

When a transaction begins, `accessed_addresses` is initialised with:

1. the transaction sender;
2. the transaction recipient, or, for a contract-creation transaction, the address being created;
3. every precompiled contract registered in `PrecompiledContracts` and active at the current block.

On Rootstock, point 3 covers the standard precompiles at `0x01` through `0x09` and the native contracts, which as of today are:

| Address | Contract |
| :------ | :------- |
| `0x0000000000000000000000000000000001000006` | Bridge |
| `0x0000000000000000000000000000000001000008` | REMASC |
| `0x0000000000000000000000000000000001000009` | HDWalletUtils |
| `0x0000000000000000000000000000000001000010` | BlockHeader |
| `0x0000000000000000000000000000000001000011` | Environment |
| `0x0000000000000000000000000000000001000016` | SECP256K1 add |
| `0x0000000000000000000000000000000001000017` | SECP256K1 multiply |

A native contract added by a later RSKIP is pre-warmed from its own activation height. The block `COINBASE` address is **not** pre-warmed; that is [EIP-3651][eip3651] and it's out of scope.

`accessed_storage_keys` starts empty.

### 3. Constants

| **Constant** | **Value** |
| :----------- | --------: |
| `COLD_SLOAD_COST` | `2100` |
| `COLD_ACCOUNT_ACCESS_COST` | `2600` |
| `WARM_STORAGE_READ_COST` | `100` |
| `SSTORE_SET_GAS` | `20000` |
| `SSTORE_RESET_GAS` | `2900` (`5000 - COLD_SLOAD_COST`) |
| `SSTORE_CLEARS_SCHEDULE` | `15000` |
| `SSTORE_SENTRY_GAS` | `2300` |

### 4. `SLOAD` (0x54)

Let `key` be the storage key being read and `addr` the executing contract.

- If `(addr, key)` isn't in `accessed_storage_keys`, charge `COLD_SLOAD_COST` and insert it.
- Otherwise, charge `WARM_STORAGE_READ_COST`.

This replaces the current flat cost of 200.

### 5. Account-accessing opcodes

For `BALANCE` (0x31), `EXTCODESIZE` (0x3B), `EXTCODECOPY` (0x3C), `EXTCODEHASH` (0x3F), `CALL` (0xF1), `CALLCODE` (0xF2), `DELEGATECALL` (0xF4) and `STATICCALL` (0xFA), let `target` be the address argument.

- If `target` isn't in `accessed_addresses`, charge `COLD_ACCOUNT_ACCESS_COST` and insert it.
- Otherwise, charge `WARM_STORAGE_READ_COST`.

This replaces the existing fixed base cost of each opcode: 400 for `BALANCE` and `EXTCODEHASH`, 700 for the others. Every other cost component is unchanged: the per-word copy cost of `EXTCODECOPY`, `VT_CALL` (9000) and `NEW_ACCT_CALL` (25000) in the call family, and the 2300 gas call stipend.

### 6. `SELFDESTRUCT` (0xFF)

If the beneficiary address isn't in `accessed_addresses`, charge `COLD_ACCOUNT_ACCESS_COST` **in addition to** the existing cost, and insert it. The existing `SUICIDE` (5000) and `NEW_ACCT_SUICIDE` (25000) costs and the `SUICIDE_REFUND` (24000) refund are unchanged.

### 7. `CREATE` (0xF0) and `CREATE2` (0xF5)

The address of the contract being created is inserted into `accessed_addresses`. No access charge is made for the insertion, and the costs of both opcodes are otherwise unchanged. The insertion happens at a fixed point:

1. The call depth and endowment balance checks run. If either fails, nothing is inserted. Ethereum clients also reject a sender nonce overflow at this stage; Rootstock has no such check and none is added.
2. The address is inserted into `accessed_addresses`.
3. The collision check runs, together with the same-block destruction check of [RSKIP-131][rskip131] (activated with [RSKIP-125][rskip125] and applied to both opcodes). If either fails, the address **stays** inserted.
4. The initcode runs. If it reverts, runs out of gas, or the creation fails when storing the code, the address **stays** inserted. Entries the initcode itself added are removed per §9.

The insertion belongs to the creating frame, not to the initcode frame. This follows geth and the execution-specs rather than the "immediately" of EIP-2929's text; see [Rationale](#why-client-behaviour-is-the-reference).

### 8. `SSTORE` (0x55)

`SSTORE` moves to the EIP-2200 net metering algorithm with the cold surcharge and constant substitutions of EIP-2929. The steps run in this order and the cost is charged once.

Let `key` be the storage key and `addr` the executing contract. Three values are involved: `original`, the value the slot held at the start of the transaction; `current`, the value it holds now; and `new`, the value being written. Tracking `original` is new to Rootstock.

1. **Sentry.** If the gas remaining *before this opcode is charged* is less than or equal to `SSTORE_SENTRY_GAS` (2300), fail the current call frame with out-of-gas ([EIP-1706][eip1706]).
2. **Cold surcharge.** Set `cost = 0`. If `(addr, key)` isn't in `accessed_storage_keys`, set `cost` to `COLD_SLOAD_COST` (2100) and insert the key.
3. **Net metering.** Add to `cost` and adjust the refund counter as follows, with `SLOAD_GAS` = `WARM_STORAGE_READ_COST` (100) and `SSTORE_RESET_GAS` = 2900.
   1. If `current` equals `new`, add `SLOAD_GAS`.
   2. Otherwise:
      1. If `original` equals `current` (the slot is clean):
         - If `original` is 0, add `SSTORE_SET_GAS`.
         - Otherwise, add `SSTORE_RESET_GAS`. If `new` is 0, add `SSTORE_CLEARS_SCHEDULE` to the refund counter.
      2. If `original` doesn't equal `current` (the slot is dirty), add `SLOAD_GAS`, then apply both:
         - If `original` isn't 0: if `current` is 0, subtract `SSTORE_CLEARS_SCHEDULE` from the refund counter; if `new` is 0, add it.
         - If `original` equals `new`: if `original` is 0, add `SSTORE_SET_GAS - SLOAD_GAS` (19900) to the refund counter; otherwise add `SSTORE_RESET_GAS - SLOAD_GAS` (2800).
4. **Charge.** Charge `cost` once. If the frame can't afford it, it fails with out-of-gas and the key inserted in step 2 is removed with the rest of the frame's effects (§9).

Because the sentry is evaluated before any cost is taken, the cold surcharge can't push a frame under it: a frame that reaches a cold `SSTORE` with 4400 gas and `current` equal to `new` succeeds and pays 2200, as in geth and the execution-specs.

**The per-frame refund counter is signed.** The subtraction in step 3.2.2 can reverse an addition made in a different call frame, so a frame's counter can go negative while the transaction total stays at or above zero. EIP-2200 states it directly: "if the implementation uses call-frame refund counter, the counter can go negative. If the implementation uses transaction-wise refund counter, the counter always stays positive." Rootstock accumulates refunds per frame and merges them into the parent when the frame succeeds, so the per-frame value must be allowed to go negative. The existing cap of half the gas used applies to the transaction total only.

Refund policy is otherwise unchanged; Ethereum's later reduction of refunds ([EIP-3529][eip3529]) isn't adopted here.

### 9. Reversion and exceptional halts

`accessed_addresses` and `accessed_storage_keys` are transaction-scoped constructs, implemented identically to the self-destruct list and the refund counter. If a call frame **reverts or halts exceptionally** (out of gas, invalid opcode, stack underflow or overflow, state modification under `STATICCALL`), both sets are restored to the contents they had when that frame began. Running out of gas while paying a cold cost is included: the entry inserted by that opcode is removed with the frame.

Entries added by the caller before the call, including the created address of §7, are unaffected.

### 10. Ethereum EIP coverage

| EIP | Status under this RSKIP |
| :-- | :---------------------- |
| [EIP-2929][eip2929], gas cost increases for state access opcodes | **Adopted**, with the pre-warm set extended per §2 |
| [EIP-2200][eip2200], structured definitions for net gas metering | **Adopted** |
| [EIP-1283][eip1283], net gas metering for `SSTORE` | **Adopted** as a component of EIP-2200 |
| [EIP-1706][eip1706], disable `SSTORE` with gasleft below the stipend | **Adopted** as a component of EIP-2200 |
| [EIP-1884][eip1884], repricing for trie-size-dependent opcodes | **Not adopted.** Superseded by EIP-2929. `SELFBALANCE` already exists under RSKIP-151 |
| [EIP-1087][eip1087], net gas metering with an in-transaction dirty map | **Not adopted.** Never accepted on Ethereum; superseded by EIP-1283 |
| [EIP-2930][eip2930], optional access lists | **Not in this RSKIP.** Separate proposal |
| [EIP-2565][eip2565], ModExp gas cost reduction | **Not in this RSKIP.** Separate proposal |
| [EIP-3529][eip3529], reduction in refunds | **Not in this RSKIP.** Separate proposal, overlaps with [RSKIP-243][rskip243] |
| [EIP-3651][eip3651], warm `COINBASE` | **Not adopted** |

## Rationale

### Why EIP-2200 and EIP-2929 ship as one rule

EIP-2929 doesn't define `SSTORE` pricing from scratch. It adds a cold surcharge and two constant substitutions on top of EIP-2200's algorithm. Rootstock has neither EIP-2200 nor EIP-1283, since `SSTORE` today compares only `current` and `new`, so adopting EIP-2929 requires introducing `original` first. Two separate rules would either be pinned to the same height, making one of them dead weight, or expose EIP-2200's standalone constants for a window nobody intends to ship.

### Why the Rootstock native contracts are pre-warmed

The cold surcharge prices a trie read. Native contracts are dispatched in client code at a fixed address and aren't in the unitrie, so charging 2600 on the first Bridge or REMASC call of a transaction would price a read that never happens.

### Why Ethereum's constants

The values 2100, 2600 and 100 were calibrated for a hexary Merkle-Patricia trie under a block gas limit on the order of 30M, not for the unitrie. They're adopted anyway because compatibility is the point of the change. A Rootstock-specific schedule would keep every estimator and ported contract special-casing Rootstock, which is the problem this RSKIP exists to remove. If one is ever wanted, it should be a separate proposal that starts from measurements.

### Why client behaviour is the reference

Two places in the Specification follow geth and the execution-specs rather than the EIP prose.

EIP-2929 says the created address is added "immediately (ie. before checks are done to determine whether or not the address is unclaimed)". Both clients insert it after the depth and balance checks and before the collision check ([geth `create`][gethevm], [execution-specs `generic_create`][essystem]). Read literally, "immediately" would warm the address when `CREATE` fails on depth or balance, and Ethereum doesn't do that.

EIP-2200 lists the sentry as step one of the `SSTORE` algorithm and EIP-2929 adds the cold surcharge "in addition", without saying which comes first. Both clients evaluate the sentry against the gas remaining before the opcode and charge the whole cost once ([geth `makeGasSStoreFunc`][gethacl], [execution-specs `sstore`][esstorage]).

Where the prose and the clients disagree, the clients are what Ethereum's consensus actually is, and a contract that behaves identically on both chains is the goal.

### Why the 2300 gas stipend isn't raised

Raising the stipend to absorb the new cold costs would be a Rootstock-specific divergence in exactly the interface this RSKIP aligns. It would also defeat EIP-1706, adopted here as part of EIP-2200: the stipend is 2300 precisely so that a callee can log an event and can't modify state.

## Backward Compatibility

This RSKIP requires a network upgrade hardfork, so all full nodes have to be updated.

### Costs

Every transaction that reads state pays more. A cold `SLOAD` rises from 200 to 2100 gas, a cold `BALANCE` or `EXTCODEHASH` from 400 to 2600, a cold `CALL` base from 700 to 2600. Repeated access to the same slot or address pays 100, so contracts that reuse state may end up cheaper, while contracts that touch many distinct slots once become significantly more expensive.

EIP-2200 moves cost downward for repeated writes. Today every `SSTORE` to a non-zero slot costs 5000; under §8 the second and later writes to a dirty slot cost 100. A reentrancy guard that sets and clears a flag in one transaction drops from roughly 10K gas net of refunds to roughly 2.3K.

`eth_estimateGas` has to account for the access sets, since the cost of a call now depends on what the transaction has already touched.

### The 2300 gas call stipend

A value-transferring `CALL` passes a 2300 gas stipend to the recipient, and Solidity's `address.transfer()` and `address.send()` forward exactly that amount. Today it buys about eleven storage reads. After this change a single cold `SLOAD` costs 2100 and a single cold account access costs 2600, so a fallback that reads an implementation slot and then delegates (every EIP-1967 proxy), or delegates to a fixed address (every EIP-1167 proxy), no longer completes within the stipend. Those proxies are immutable bytecode and can't be repaired in place. Ethereum took the same break at Istanbul ([safe-contracts issue #149][safe149]) and again at Berlin ([folia-app/eip-2929][folia2929]).

A failure needs a **pair**: a payer that forwards exactly the stipend, and a recipient whose fallback no longer fits in it. Unaffected are plain value transfers from an externally-owned account (a top-level transaction isn't a `CALL` and forwards all of its gas), any call made with adequate gas, `call{value: n}("")` forwarding all remaining gas, and payouts back to `msg.sender`, where the recipient is already warm because the access sets are transaction-wide. What breaks is **a contract paying with `transfer()` or `send()` to a second contract that the transaction hasn't touched yet**: a splitter, an escrow release, a router refunding a third party.

Every contract on both networks was surveyed for this RSKIP (mainnet at block 9,216,000, testnet at block 7,615,296). Each side of the pair was sized separately:

| | Mainnet | Testnet |
| :-- | --: | --: |
| Recipients that accept a bare `transfer()` today and stop fitting the stipend | 14,361 of 16,183 | 18,670 of 26,611 |
| of which proxy fallbacks | 99.2% | 96.4% |
| Payers using the `transfer()`/`send()` idiom (upper bound) | 945 | 10,058 |

The breakage rate is the intersection of the two, which wasn't measured. Since the recipients are immutable, the fix belongs to the payers: contracts still using `transfer()` or `send()` should move to `call{value: n}("")` with an explicit gas budget and a checked return value. The survey method, the breakdown by cause and the payer review are in the discussion thread for this RSKIP.

## Security Considerations

For client implementations, this RSKIP reduces the worst-case state access a single block can force, as described in [Motivation](#motivation). Denial of service based on cheap state reads becomes considerably less effective, and the block gas limit can be raised without raising worst-case block validation time along with it.

For deployed contracts, this RSKIP introduces one failure mode where there previously was none: a contract paying with `transfer()` or `send()` to a cold contract recipient now reverts. Its scope is in [Backward Compatibility](#the-2300-gas-call-stipend).

EIP-1706 closes a reentrancy surface rather than opening one. `SSTORE` is rejected outright when the remaining gas is at or below the stipend, instead of being left to fail partway through.

## Test Cases

An implementation should verify at least the following, stated as required behaviours rather than as a specific test suite.

**Access sets and warming**

1. Two `SLOAD` operations on the same slot in one transaction cost 2100 then 100.
2. Two `SLOAD` operations on different slots of the same contract each cost 2100.
3. An address warmed inside a nested call stays warm in the caller after that call returns.
4. An address warmed inside a call frame that **reverts** is cold again in the caller. The same holds when the frame **halts exceptionally**: out of gas, an invalid opcode, or a write attempted under `STATICCALL`.
5. A storage slot warmed inside a call frame that reverts or runs out of gas is cold again in the caller, including when the frame runs out of gas while paying the cold cost of that very slot.
6. The access sets don't persist between two transactions in the same block.

**Pre-warmed entries**

7. The first `BALANCE` of the transaction sender costs 100, not 2600.
8. The first `CALL` to the transaction recipient costs 100.
9. The first `CALL` to each precompile at `0x01` through `0x09` costs 100.
10. The first `CALL` to the Bridge (`0x01000006`) and to REMASC (`0x01000008`) costs 100.
11. The first `BALANCE` of an arbitrary externally-owned account costs 2600.

**Account-accessing opcodes**

12. Each of `BALANCE`, `EXTCODESIZE`, `EXTCODECOPY`, `EXTCODEHASH`, `CALL`, `CALLCODE`, `DELEGATECALL` and `STATICCALL` charges 2600 on first access to a target and 100 thereafter.
13. `EXTCODECOPY`'s per-word copy cost is applied on top of the access cost, unchanged.
14. A value-transferring `CALL` still adds `VT_CALL`, and a call creating an account still adds `NEW_ACCT_CALL`.
15. `SELFDESTRUCT` to a cold beneficiary costs 2600 more than to a warm one.
16. An address created by `CREATE` or `CREATE2` is warm immediately afterwards, and stays warm when the creation fails after the pre-checks (initcode reverts, initcode runs out of gas, address collides). It isn't inserted when the depth or balance check fails.

**`SSTORE`**

17. The full EIP-2200 transition table, covering every combination of `original`, `current` and `new` being zero or non-zero, including both dirty-slot branches and both reset-refund cases (19900 and 2800).
18. A cold slot adds 2100 to whichever EIP-2200 cost applies, and a warm slot doesn't.
19. An `SSTORE` with 2300 or less gas remaining *before the opcode is charged* fails with out-of-gas regardless of the values involved. An `SSTORE` on a cold slot with `current` equal to `new` and 4400 gas remaining succeeds, costs 2200 and leaves 2200.
20. Refunds accumulate and are subtracted correctly across multiple writes to the same slot, including across call frames. A slot originally holding `X` is written to 0 in the outer frame, then to `Y` in a reentrant inner frame that succeeds: the net refund is 0 and the inner frame doesn't fail even though its own counter reaches -15000.

**Activation**

21. In the block immediately before the activation height, all costs are the pre-activation ones. In the activation block itself, all costs are the new ones.

Ethereum's execution-spec fixtures at `tests/berlin/eip2929_gas_cost_increases` in [`ethereum/execution-specs`][execspecs] cover a useful subset of the EIP-2929 cases and can be reused with the pre-warm set adjusted for §2. [Pull request 649 in `ethereum/tests`][tests649] covers the EIP-1706 and EIP-2200 out-of-gas cases. Neither suite covers EIP-2200 exhaustively, so cases 17 through 20 need purpose-written tests.

## References

[1] [EIP-2929: Gas cost increases for state access opcodes][eip2929]

[2] [EIP-2200: Structured definitions for net gas metering][eip2200]

[3] [EIP-1283: Net gas metering for SSTORE without dirty maps][eip1283]

[4] [EIP-1706: Disable SSTORE with gasleft lower than call stipend][eip1706]

[5] [EIP-1884: Repricing for trie-size-dependent opcodes][eip1884]

[6] [EIP-2930: Optional access lists][eip2930]

[7] [EIP-2565: ModExp gas cost][eip2565]

[8] [EIP-3529: Reduction in refunds][eip3529]

[9] [RSKIP-243: Intra-transaction Gas Refunds][rskip243]

[10] [RSKIP-131: Preventing CREATE2-after-SUICIDE in the same block][rskip131]

[11] [RSKIP-125: Create2][rskip125]

[12] [safe-contracts issue #149, "With istanbul it is not possible to use `send` or `transfer` to send funds to a Safe"][safe149]

[13] [folia-app/eip-2929, fixing `.transfer()` to Gnosis Safe with an access list][folia2929]

[14] [go-ethereum, `core/vm/operations_acl.go` (`makeGasSStoreFunc`)][gethacl] and [`core/vm/evm.go` (`create`)][gethevm]

[15] [ethereum/execution-specs, Berlin `vm/instructions/storage.py` (`sstore`)][esstorage] and [`system.py` (`generic_create`)][essystem]

[eip1087]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1087.md
[eip1283]: https://eips.ethereum.org/EIPS/eip-1283
[eip1706]: https://eips.ethereum.org/EIPS/eip-1706
[eip1884]: https://eips.ethereum.org/EIPS/eip-1884
[eip2200]: https://eips.ethereum.org/EIPS/eip-2200
[eip2565]: https://eips.ethereum.org/EIPS/eip-2565
[eip2929]: https://eips.ethereum.org/EIPS/eip-2929
[eip2930]: https://eips.ethereum.org/EIPS/eip-2930
[eip3529]: https://eips.ethereum.org/EIPS/eip-3529
[eip3651]: https://eips.ethereum.org/EIPS/eip-3651
[rskip125]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP125.md
[rskip131]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP131.md
[rskip243]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP243.md
[safe149]: https://github.com/safe-global/safe-contracts/issues/149
[folia2929]: https://github.com/folia-app/eip-2929
[execspecs]: https://github.com/ethereum/execution-specs/tree/master/tests/berlin/eip2929_gas_cost_increases
[tests649]: https://github.com/ethereum/tests/pull/649
[gethacl]: https://github.com/ethereum/go-ethereum/blob/master/core/vm/operations_acl.go
[gethevm]: https://github.com/ethereum/go-ethereum/blob/master/core/vm/evm.go
[esstorage]: https://github.com/ethereum/execution-specs/blob/master/src/ethereum/forks/berlin/vm/instructions/storage.py
[essystem]: https://github.com/ethereum/execution-specs/blob/master/src/ethereum/forks/berlin/vm/instructions/system.py

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

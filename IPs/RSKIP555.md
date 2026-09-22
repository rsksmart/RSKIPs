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

This RSKIP brings Ethereum's cold/warm state access gas model ([EIP-2929][eip2929]) to Rootstock, together with the net gas metering for `SSTORE` that EIP-2929 builds on ([EIP-2200][eip2200]).

State-accessing opcodes stop having a single fixed price. The first access to an address or a storage slot inside a transaction is charged a *cold* cost that reflects a trie read. Every later access to the same address or slot in that transaction is charged a much smaller *warm* cost. `SSTORE` moves from Rootstock's current pricing, which compares only the current and the new value, to the EIP-2200 model, which also considers the value the slot held when the transaction started.

Both changes activate together under a single consensus rule, `RSKIP555`.

## Motivation

### Compatibility

Rootstock aims to run the same contracts and the same tooling as Ethereum, and gas is part of that interface. Since Berlin, every Ethereum client, every gas estimator, the Solidity compiler's assumptions about worst-case costs and a large body of audited contract patterns have been written against the cold/warm model.

Rootstock's state access prices predate Istanbul. `SLOAD` costs 200 gas flat, and `BALANCE` and `EXTCODEHASH` cost 400. Ethereum moved away from those values in 2019 and again in 2021. Anything that reasons about gas instead of merely spending it has to special-case Rootstock as a result: estimators, benchmarks and ported contracts with hardcoded gas budgets.

### Resource pricing

A single price for `SLOAD` can't be right for both of the cases it covers. Reading a slot for the first time in a transaction means walking the unitrie and, when the node isn't cached, hitting disk. Reading the same slot again is a lookup in memory. The first costs orders of magnitude more node time than the second, so a flat price is either too high for the repeat case or too low for the first one.

Rootstock's current price resolves that in the wrong direction. At 200 gas, a state read is one of the cheapest ways to consume node time per unit of gas. This is the same imbalance Ethereum repriced against after the 2016 Shanghai attacks, and it gets worse as the state grows, because deeper tries and lower cache hit rates raise the real cost of the first read while the price stays fixed.

### Effect on the block gas limit

The block gas limit bounds how much state a single block can force a node to read. The figure that matters is cold reads per unit of gas, and it doesn't depend on the limit: repricing `SLOAD` from 200 to 2100 divides it by about 10.

Applying that to the current limit and to the planned one:

| Block gas limit | Cold state reads per block |
| :-------------- | -------------------------: |
| 6.8M, today | `6.8M / 200` = ~34K |
| 12M, current pricing | `12M / 200` = ~60K |
| 12M, with this RSKIP | `12M / 2100` = ~5.7K |

Raising the limit to around 12M without repricing state access would increase the worst-case random disk I/O per block by about 76%. With this RSKIP, the same 12M limit lands at roughly a sixth of today's worst case.

Therefore this RSKIP shouldn't be read as an independent change that happens to sit alongside the gas limit increase. It's what makes that increase safe. If both ship, this one should activate first or at the same height.

### On the constants

The values adopted here (2100, 2600 and 100) are Ethereum's. They were calibrated against a hexary Merkle-Patricia trie under a block gas limit on the order of 30M. Rootstock uses a unitrie under a smaller limit, so the calibration doesn't transfer exactly.

They're adopted anyway, because compatibility is the point of the change. A schedule closer to Rootstock's measured costs but different from every other EVM would give up the benefit this RSKIP exists to obtain. See [Rationale](#why-ethereums-constants).

## Specification

This RSKIP requires a hard fork, since it modifies consensus rules. Everything below activates under a single new consensus rule, `RSKIP555`, at the block height set by the network upgrade that includes it. The target upgrade is Cardamom. Before activation, all behaviour is unchanged.

[EIP-1884][eip1884] is deliberately **not** adopted as an intermediate step. Its repricings of `SLOAD`, `BALANCE` and `EXTCODEHASH` are replaced by the cold/warm costs below at the same height, so they'd never be observable. Its other component, the `SELFBALANCE` opcode, already exists on Rootstock under RSKIP-151.

### 1. Access sets

Two sets are maintained for the duration of a single transaction:

- `accessed_addresses`, a set of addresses.
- `accessed_storage_keys`, a set of `(address, storage key)` pairs.

Both are created when transaction execution begins and discarded when it ends. They are **shared across all call frames** of the transaction. A nested call inherits the accumulated sets from its caller and adds to them, so warming an address in an inner frame keeps it warm in the outer frame after the inner frame returns.

### 2. Pre-warmed entries

When a transaction begins, `accessed_addresses` is initialised with:

1. the transaction sender;
2. the transaction recipient, or, for a contract-creation transaction, the address being created;
3. every precompiled contract registered in `PrecompiledContracts` and active at the current block.

Point 3 covers more addresses on Rootstock than on Ethereum. It includes the standard precompiles at `0x01` through `0x09`, and additionally the Rootstock native contracts. Which as of today, are:

| Address | Contract |
| :------ | :------- |
| `0x0000000000000000000000000000000001000006` | Bridge |
| `0x0000000000000000000000000000000001000008` | REMASC |
| `0x0000000000000000000000000000000001000009` | HDWalletUtils |
| `0x0000000000000000000000000000000001000010` | BlockHeader |
| `0x0000000000000000000000000000000001000011` | Environment |
| `0x0000000000000000000000000000000001000016` | SECP256K1 add |
| `0x0000000000000000000000000000000001000017` | SECP256K1 multiply |

Native contracts are pre-warmed because they live in client code, not in the unitrie. Charging a cold surcharge to reach one would price a trie read that never happens.

> **Note**: any native contract added by a later RSKIP is pre-warmed from its own activation height.

The block `COINBASE` address is **not** pre-warmed. That's a separate Ethereum change ([EIP-3651][eip3651]) and it's out of scope here.

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

The following opcodes take a target address as an argument: `BALANCE` (0x31), `EXTCODESIZE` (0x3B), `EXTCODECOPY` (0x3C), `EXTCODEHASH` (0x3F), `CALL` (0xF1), `CALLCODE` (0xF2), `DELEGATECALL` (0xF4) and `STATICCALL` (0xFA).

For each of them, let `target` be that address.

- If `target` isn't in `accessed_addresses`, charge `COLD_ACCOUNT_ACCESS_COST` and insert it.
- Otherwise, charge `WARM_STORAGE_READ_COST`.

The existing fixed base costs of these opcodes are **removed** and replaced by the above. Today those are 400 for `BALANCE` and `EXTCODEHASH`, and 700 for `EXTCODESIZE`, `EXTCODECOPY` and the call family.

Every other component of these opcodes' costs is unchanged. `EXTCODECOPY` still pays its per-word copy cost. The call family still pays `VT_CALL` (9000) for value-transferring calls and `NEW_ACCT_CALL` (25000) when the call creates an account. The 2300 gas call stipend is unchanged; see [Backward Compatibility](#the-2300-gas-call-stipend).

### 6. `SELFDESTRUCT` (0xFF)

If the beneficiary address isn't in `accessed_addresses`, charge `COLD_ACCOUNT_ACCESS_COST` **in addition to** the existing cost, and insert it. If it's already present, no additional charge is made.

The existing `SUICIDE` (5000) and `NEW_ACCT_SUICIDE` (25000) costs and the `SUICIDE_REFUND` (24000) refund are unchanged.

### 7. `CREATE` (0xF0) and `CREATE2` (0xF5)

The address of the newly created contract is inserted into `accessed_addresses`. No access charge is made for the insertion.

### 8. `SSTORE` (0x55)

`SSTORE` pricing changes in two parts, both applied at the same activation height.

**First, the cold surcharge.** Let `key` be the storage key and `addr` the executing contract. If `(addr, key)` isn't in `accessed_storage_keys`, charge `COLD_SLOAD_COST` and insert it. This charge is made *in addition to* the cost computed below.

**Second, the EIP-2200 net metering algorithm**, evaluated with `SLOAD_GAS` equal to `WARM_STORAGE_READ_COST` (100) and `SSTORE_RESET_GAS` equal to 2900.

Three values are involved:

- `original`, the value the slot held at the start of the transaction;
- `current`, the value the slot holds now;
- `new`, the value being written.

Tracking `original` is new to Rootstock. The current implementation compares only `current` and `new` (`VM.java:1321`-`1355`).

The algorithm:

1. If remaining gas is less than or equal to `SSTORE_SENTRY_GAS` (2300), fail the current call frame with an out-of-gas exception. This is [EIP-1706][eip1706], a component of EIP-2200.
2. If `current` equals `new`, charge `SLOAD_GAS` (100).
3. If `current` doesn't equal `new`:
   1. If `original` equals `current`, so the slot hasn't been modified yet in this transaction:
      - If `original` is 0, charge `SSTORE_SET_GAS` (20000).
      - Otherwise, charge `SSTORE_RESET_GAS` (2900). If `new` is 0, add `SSTORE_CLEARS_SCHEDULE` (15000) to the refund counter.
   2. If `original` doesn't equal `current`, so the slot is dirty, charge `SLOAD_GAS` (100), then apply both of the following:
      - If `original` isn't 0:
        - If `current` is 0, subtract `SSTORE_CLEARS_SCHEDULE` (15000) from the refund counter.
        - If `new` is 0, add `SSTORE_CLEARS_SCHEDULE` (15000) to the refund counter.
      - If `original` equals `new`, so the slot has been reset to where it started:
        - If `original` is 0, add `SSTORE_SET_GAS - SLOAD_GAS` (19900) to the refund counter.
        - Otherwise, add `SSTORE_RESET_GAS - SLOAD_GAS` (2800) to the refund counter.

Refund policy is otherwise unchanged. `SSTORE_CLEARS_SCHEDULE` stays at 15000 and the `SELFDESTRUCT` refund is retained. Ethereum's later reform of both ([EIP-3529][eip3529]) isn't adopted here.

### 9. Reversion

If a call frame reverts, `accessed_addresses` and `accessed_storage_keys` are restored to the contents they had when that frame began. Entries added by a frame that reverts don't stay warm for later frames.

### 10. Ethereum EIP coverage

| EIP | Status under this RSKIP |
| :-- | :---------------------- |
| [EIP-2929][eip2929], gas cost increases for state access opcodes | **Adopted**, with the pre-warm set extended per §2 |
| [EIP-2200][eip2200], structured definitions for net gas metering | **Adopted** |
| [EIP-1283][eip1283], net gas metering for `SSTORE` | **Adopted** as a component of EIP-2200 |
| [EIP-1706][eip1706], disable `SSTORE` with gasleft below the stipend | **Adopted** as a component of EIP-2200 |
| [EIP-1884][eip1884], repricing for trie-size-dependent opcodes | **Not adopted.** Superseded by EIP-2929 for `SLOAD`, `BALANCE` and `EXTCODEHASH`. Its `SELFBALANCE` opcode already exists under RSKIP-151 |
| [EIP-1087][eip1087], net gas metering with an in-transaction dirty map | **Not adopted**, on Rootstock or on Ethereum. It was never accepted, and EIP-1283 supersedes it |
| [EIP-2930][eip2930], optional access lists | **Not in this RSKIP.** Companion, see below |
| [EIP-2565][eip2565], ModExp gas cost reduction | **Not in this RSKIP.** Follow-up, see below |
| [EIP-3529][eip3529], reduction in refunds | **Not in this RSKIP.** Follow-up, see below |
| [EIP-3651][eip3651], warm `COINBASE` | **Not adopted** |

#### Companion proposal: EIP-2930 access lists

On Ethereum, EIP-2929 and EIP-2930 shipped in the same hard fork, and the pairing was intentional. EIP-2929 makes the first touch of an address or a slot expensive, and EIP-2930 lets the transaction sender declare those addresses and slots up front and pay the warm rate for them. More details on the pairing can be found in [EIP-2930][eip2930].

A separate RSKIP should specify EIP-2930 for Rootstock. It's kept separate because it depends on a typed transaction envelope, the subject of [RSKIP-543][rskip543], while this RSKIP has no such dependency. That dependency chain, and not any question of desirability, is what determines the ordering.

Whether EIP-2930 has to activate no later than this RSKIP is left open here, because it depends on a measurement that hasn't been done yet. Access lists are the only mechanism that can repair the one case with no other remedy: an immutable payer making a cold third-party `transfer()` to a recipient that no longer fits the stipend. This is not theoretical, it's how the same breakage was handled on Ethereum after Berlin (see [Precedent on Ethereum](#precedent-on-ethereum)). Without EIP-2930 that pair is permanently broken. With it the pair becomes repairable, though only when the transaction sender supplies the list, so it also depends on wallet and library support.

The size of that set is unknown. The recipient side counts 14,361 mainnet contracts, but that's the wrong side of the pair and it's an upper bound. The payer side is 945 mainnet contracts, and how many of those make cold third-party payouts hasn't been established. Reviewing those 945 is what should settle whether EIP-2930 is a hard co-requisite or a strong recommendation. See [Backward Compatibility](#what-a-failure-requires).

#### Follow-up proposals

Two further Berlin-era changes are recorded here and left to their own proposals:

- **EIP-2565**, which reprices the ModExp precompile. Rootstock still uses the original EIP-198 formula. It's a self-contained precompile repricing, it has no dependency on the machinery in this RSKIP, and it moves cost downward rather than upward.
- **EIP-3529**, which reduces `SSTORE` refunds and removes the `SELFDESTRUCT` refund. It's a London change rather than a Berlin one, and it overlaps with ground covered by [RSKIP-243][rskip243].

## Rationale

### Why net metering for `SSTORE` is required, not optional

EIP-2929 doesn't define `SSTORE` pricing from scratch. It defines a cold surcharge and two constant substitutions applied *on top of* EIP-2200's algorithm. Rootstock has neither EIP-2200 nor EIP-1283, since `SSTORE` compares `current` against `new` and nothing else. Adopting EIP-2929 for `SSTORE` therefore requires first introducing the notion of an original value, which is EIP-2200. The two are specified together because they have to activate together.

### Why one consensus rule

For the same reason. Gating EIP-2200 and EIP-2929 separately leaves two options, and neither is useful. Pin both rules to the same height and the second rule is dead weight. Pin them to different heights and there's a window in which EIP-2200's standalone constants are consensus-visible, which is a gas schedule nobody intends to ship and every client would nonetheless have to implement correctly. A single rule removes the choice.

### Why the Rootstock native contracts are pre-warmed

EIP-2929's pre-warm set exists because the cold surcharge is meant to price a trie read, and precompiles aren't in the trie. Rootstock's native contracts are in the same position: they're dispatched in client code at a fixed address. Not pre-warming them would add 2600 gas to the first Bridge interaction in every transaction that makes one, charging for a unitrie access that never occurs. Peg-in and peg-out flows are among the most consequential transactions on the network, so the exemption carries more weight here than it does on Ethereum.

### Why Ethereum's constants

Rootstock could derive its own schedule from measurements of the unitrie. Doing so would produce numbers better fitted to Rootstock's storage layout and block gas limit, and it would defeat the purpose of the change, because contracts and tooling would still need Rootstock-specific gas assumptions. That's the problem this RSKIP sets out to remove.

Adopting 2100, 2600 and 100 accepts a calibration performed for a different trie. That's a real cost, and it's the smaller one. If a Rootstock-specific schedule is ever wanted, it should be a separate proposal that starts from measurements, and it should be weighed against the compatibility it would give up.

### Why the 2300 gas stipend isn't raised instead

Raising the stipend is the obvious alternative to EIP-2930, and it has one clear advantage: it needs no tooling support at all. Around 4800 gas would cover both proxy shapes, since an EIP-1967 proxy needs 4700 and an EIP-1167 one needs 2600.

It's rejected for two reasons. The first is that it's a Rootstock-specific divergence in exactly the interface this RSKIP exists to align, so it trades the whole benefit for a partial mitigation.

The second is more specific. EIP-1706, adopted here as a component of EIP-2200, exists to reject `SSTORE` when remaining gas is at or below 2300. The stipend is set at 2300 so that a callee can log an event and deliberately can't modify state. Forwarding 4800 gas to an untrusted callee re-opens the reentrancy surface that EIP-1706 closes. This RSKIP would be adopting a safety rule and defeating it in the same hard fork.

## Backward Compatibility

This RSKIP requires a network upgrade hardfork, so all full nodes have to be updated.

### Transactions become more expensive

Every transaction that reads state pays more. A cold `SLOAD` rises from 200 to 2100 gas. A cold `BALANCE` or `EXTCODEHASH` rises from 400 to 2600. A cold `CALL` base rises from 700 to 2600. Contracts that touch the same slots repeatedly benefit from the warm rate of 100 and may end up cheaper overall. Contracts that touch many distinct slots or addresses once each become significantly more expensive.

Not every part of this RSKIP moves cost upward. EIP-2200 makes repeated writes to the same slot much cheaper: today `SSTORE` charges 5000 gas for every write to a slot that's already non-zero, and under the algorithm in §8 the second and later writes to a dirty slot cost 100. A reentrancy guard, which sets a flag and clears it in the same transaction, goes from roughly 10K gas net of refunds to roughly 2.3K.

Raising the gas limit covers the increase in most cases, and `eth_estimateGas` gives correct guidance once it accounts for the access sets. Rootstock's block gas limit is lower than Ethereum's, so a transaction that touches a large amount of distinct state consumes a proportionally larger share of a block, and the ceiling on how much state one transaction can reach is correspondingly lower.

### The 2300 gas call stipend

This is the sharpest break in the RSKIP, and it's also the part most easily overstated. It's worth being precise about what has to be true for a payment to fail.

A value-transferring `CALL` passes a stipend of 2300 gas to the recipient. Solidity's `address.transfer()` and `address.send()` forward exactly this amount. The stipend is meant to be enough for the recipient to log the transfer, and deliberately not enough to modify state.

Today, at 200 gas per `SLOAD`, a recipient fallback can perform about eleven storage reads inside the stipend. After this change, one cold `SLOAD` costs 2100, which still fits, but it leaves only 200 gas. That's enough to finish a fallback that does nothing else, and not enough for a second read, a cold account access, or an event whose arguments require a read. The threshold drops from eleven reads to one, with almost no room after it.

`SSTORE` inside the stipend was already impossible on cost grounds. With `SSTORE_SENTRY_GAS` it now fails explicitly with an out-of-gas exception instead of by exhausting gas.

Contracts that call out with an explicitly limited gas budget can fail for the same reason. This pattern has been discouraged since EIP-150 precisely because gas schedules change, and Ethereum accepted the identical break at Berlin.

#### What a failure requires

A failure needs a **pair**: a payer that forwards exactly the stipend, and a recipient whose fallback no longer fits inside it. Neither side is enough on its own, and most of the ways coin moves on Rootstock never form the pair.

The change is confined to the stipend, which is added only by the `CALL` opcode when it carries value. The following are unaffected:

- A plain value transfer from an externally-owned account to a contract. A top-level transaction isn't a `CALL` and it forwards all of its gas, so a 4700 gas proxy fallback is trivially affordable.
- Any contract call made with adequate gas, including every ordinary function call to the affected proxies. Those contracts keep working in full.
- `call{value: n}("")` with all remaining gas forwarded, which is the pattern Solidity has recommended since Istanbul.
- A payout back to the caller, which is the common `msg.sender.transfer(amount)` shape. The access sets are transaction-wide, so a proxy that has already served a call in this transaction holds its implementation slot and implementation address warm. The re-entrant fallback then costs roughly 200 gas and fits. Wrapped-coin `withdraw()` is this case.

What breaks is narrower: **a contract paying coin with `transfer()` or `send()` to a second contract that the transaction hasn't touched yet.** The recipient's fallback is charged at the cold rate and the stipend no longer covers it. `transfer()` propagates the failure and reverts the whole transaction, and `send()` returns `false` and leaves the caller to handle a result that's often unhandled. Third-party payouts are the shape at risk, such as a splitter, an escrow release, or a router refunding an address other than the caller.

The two subsections below size each side of the pair separately. Neither one is the breakage rate. The breakage rate is the intersection, and this survey didn't measure it.

#### The recipient side

Every contract on both networks was surveyed for this RSKIP, on mainnet at block 9,216,000 and on testnet at block 7,615,296. Each contract was executed symbolically from its entry point with empty calldata, a non-zero `CALLVALUE` and a 2300 gas budget, first under the current schedule and then under the schedule specified here. A contract counts as affected only when a completing path exists today and none exists afterwards, so contracts that already reject a bare transfer are excluded instead of being counted as damage.

The question this answers is narrow: **if this contract were sent a bare `transfer()` as the first thing to touch it in a transaction, would its fallback still complete?** The answer sizes a population at risk. It isn't a rate of breakage, because a recipient only fails when some payer actually pays it that way, and every contract counted below keeps working for every caller that forwards adequate gas.

| Outcome | Mainnet | | Testnet | |
| :------ | ------: | ---: | ------: | ---: |
| Already rejects a bare transfer | 8,184 | 33.6% | 86,587 | 76.4% |
| Still completes within the stipend | 1,822 | 7.5% | 7,941 | 7.0% |
| **Stops completing within the stipend** | **14,361** | **58.9%** | **18,670** | **16.5%** |
| Inconclusive | 2 | < 0.1% | 67 | 0.1% |
| Total | 24,369 | | 113,265 | |

Measured against the contracts that accept a bare transfer today:

- **Mainnet: 14,361 of 16,183, or 88.7%.**
- Testnet: 18,670 of 26,611, or 70.2%.

Mainnet is higher because two thirds of its contracts are payable, against one quarter on testnet. Restricting to contracts that hold a balance, 184 of 259 are affected on mainnet (71.0%), and those hold **559 RBTC** against 6 RBTC in the ones that keep working. The two largest are an OpenZeppelin `TransparentUpgradeableProxy` holding 278 RBTC and a custom upgradeable proxy holding 253 RBTC.

> **Note**: those 559 RBTC are not value at risk. They sit in recipients, and they stay withdrawable through ordinary calls, which forward gas normally. The figure says how economically live the affected contracts are, and nothing beyond that. What can trap value is the payer side, covered next.

Causes, by what the contract does on the path it takes today:

| Cause | Mainnet | Testnet |
| :---- | ------: | ------: |
| Proxy: cold `SLOAD` of an implementation slot, then `DELEGATECALL` | 12,031 | 15,262 |
| Proxy: `DELEGATECALL` to an address fixed in the bytecode (EIP-1167) | 2,213 | 2,736 |
| Storage read together with a cold account access | 88 | 439 |
| Cold account access alone | 18 | 46 |
| Storage read alone | 11 | 187 |

**This is overwhelmingly a proxy problem.** 14,244 of the 14,361 mainnet contracts (99.2%) and 17,998 of the 18,670 testnet ones (96.4%) are proxy fallbacks, and the arithmetic isn't marginal. An EIP-1967 proxy pays 2100 for the cold `SLOAD` of the implementation slot and 2600 for the cold `DELEGATECALL`, so `2100 + 2600 = 4700` against a 2300 stipend. It exceeds the budget at the `SLOAD` alone and never reaches the call. An EIP-1167 minimal proxy performs no `SLOAD` and still fails, because the `DELEGATECALL` by itself costs 2600. Neither can be repaired in place, since both are immutable deployed bytecode.

Two limits on these figures. The analysis is a static approximation: storage values are treated as unknown, so a fallback whose cost depends on what it reads is judged only on the paths the analysis can prove, which is why 2 mainnet and 67 testnet contracts came back inconclusive. And it models a bare `transfer()` into a cold frame, so a contract reached later in a transaction that has already touched its slots pays the warm rate and survives.

#### The payer side

Searching the same bytecode corpus for the Solidity `transfer()` and `send()` idiom, which is a `PUSH2 0x08fc` with the gas argument computed as `mul(iszero(value), 2300)` ahead of a `CALL`, finds 945 contracts on mainnet and 10,058 on testnet. Of the mainnet payers, 59 hold a balance totalling 209 RBTC, and a single WRBTC deployment accounts for 204 of that. WRBTC pays only to `msg.sender`, so it takes the warm re-entrant path and is unaffected.

945 is an upper bound in turn. It counts the bytecode idiom, not whether a given payout is cold or goes to a third party, and a payer that only ever pays `msg.sender` is safe. Set against 14,361 on the recipient side, the two sides of the pair differ by more than an order of magnitude, and what actually breaks is the intersection.

One shape does make value unreachable: an immutable payer whose only payout path is a cold third-party `transfer()`, paired with a recipient that no longer fits the stipend. Neither side can be changed, so there's no in-place remedy.

> **Note**: reviewing the 945 mainnet payers is what turns this from an estimate into a number, and it should be a precondition of activation. Each payer classifies as one of three things: it pays only `msg.sender`, so it's safe; it pays third parties but is upgradeable, so it's fixable; or it's an immutable third-party payer, and it belongs to the real set. This RSKIP doesn't claim that set is empty.

Because the recipients are immutable, the fix belongs to the payers and to tooling. Contracts still using `transfer()` or `send()` should move to `call{value: n}("")` with an explicit gas budget and a checked return value. Where the payer is itself immutable, an EIP-2930 access list naming the recipient and its implementation slot brings both to the warm rate, which drops the proxy fallback from 4700 to roughly 200 gas and fits the stipend again. That path only works if the transaction sender supplies the list, so it depends on wallet and library support as much as on the protocol.

#### Precedent on Ethereum

Ethereum absorbed this same break twice, and the ecosystem adapted both times.

[EIP-1884][eip1884] raised `SLOAD` from 200 to 800 at Istanbul in 2019. Gnosis Safe was the well-publicised casualty: its proxy needs 800 for the mastercopy `SLOAD`, 700 for the call into it and roughly 900 for the event, which doesn't fit in 2300 ([safe-contracts issue #149][safe149]). That fork is what turned "stop using `transfer()`" into standard advice ([Consensys Diligence, 2019][stoptransfer]).

EIP-2929 then moved the same read to 2100 cold at Berlin in 2021, with proxies far more common than they had been two years earlier, and it hit the identical shape again. The remedy used in production was EIP-2930: naming the recipient in an access list prepays the cold cost and lets the 2300 gas call succeed ([folia-app/eip-2929][folia2929]).

Two differences are worth stating rather than glossing over. Rootstock doesn't adopt EIP-1884 (§10), so the move Ethereum made in two steps arrives here in one, from 200 straight to 2100. And Ethereum's own pre-Istanbul survey flagged roughly 200 mainnet contracts ([holiman/eip-1884-security][eip1884sec]), against 14,361 here, but the two numbers don't compare: that survey ran in 2019 when proxies were rare, and it worked from observed transaction traffic rather than from every deployed contract. The precedent supports two claims only, that the break is survivable and that EIP-2930 is the remedy that works in practice. It says nothing about the size of Rootstock's affected set.

### Gas estimation and tooling

`eth_estimateGas` has to account for the access sets, because the cost of a call now depends on what the transaction has already touched. Estimating a call in isolation can overstate its cost relative to the same call made later in a transaction.

## Security Considerations

For client implementations, this RSKIP reduces the worst-case state access a single block can force, as described in [Motivation](#effect-on-the-block-gas-limit). That makes denial of service attacks based on cheap state reads considerably less effective, and it's what allows the block gas limit to be raised without raising the worst-case block validation time along with it.

For deployed contracts, this RSKIP introduces a failure mode where there previously was none. A contract paying coin with `transfer()` or `send()` to a cold contract recipient now reverts. The measured scope is in [Backward Compatibility](#the-recipient-side). The residual risk with no remedy is an immutable payer paired with an immutable recipient, and closing it depends on EIP-2930 being available.

EIP-1706, adopted as a component of EIP-2200, closes a reentrancy surface rather than opening one. `SSTORE` is rejected outright when remaining gas is at or below the stipend, instead of being left to fail by running out of gas partway through.

## Test Cases

An implementation should verify at least the following. These are stated as required behaviours rather than as a specific test suite.

**Access sets and warming**

1. Two `SLOAD` operations on the same slot in one transaction cost 2100 then 100.
2. Two `SLOAD` operations on different slots of the same contract each cost 2100.
3. An address warmed inside a nested call stays warm in the caller after that call returns.
4. An address warmed inside a call frame that **reverts** is cold again in the caller.
5. A storage slot warmed inside a call frame that reverts is cold again in the caller.
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
16. An address created by `CREATE` or `CREATE2` is warm immediately afterwards.

**`SSTORE`**

17. The full EIP-2200 transition table, covering every combination of `original`, `current` and `new` being zero or non-zero, including both dirty-slot branches and both reset-refund cases (19900 and 2800).
18. A cold slot adds 2100 to whichever EIP-2200 cost applies, and a warm slot doesn't.
19. An `SSTORE` attempted with 2300 or less gas remaining fails with out-of-gas, regardless of the values involved.
20. Refunds accumulate and are subtracted correctly across multiple writes to the same slot inside one transaction.

**Activation**

21. In the block immediately before the activation height, all costs are the pre-activation ones. In the activation block itself, all costs are the new ones.

Ethereum's execution-spec fixtures at `tests/berlin/eip2929_gas_cost_increases` in [`ethereum/execution-specs`][execspecs] cover a useful subset of the EIP-2929 cases and can be reused with the pre-warm set adjusted for §2. [Pull request 649 in `ethereum/tests`][tests649] adds coverage for the EIP-1706 and EIP-2200 out-of-gas cases. Neither suite covers EIP-2200 exhaustively, so cases 17 through 20 need purpose-written tests.

## References

[1] [EIP-2929: Gas cost increases for state access opcodes][eip2929]

[2] [EIP-2200: Structured definitions for net gas metering][eip2200]

[3] [EIP-1283: Net gas metering for SSTORE without dirty maps][eip1283]

[4] [EIP-1706: Disable SSTORE with gasleft lower than call stipend][eip1706]

[5] [EIP-1884: Repricing for trie-size-dependent opcodes][eip1884]

[6] [EIP-2930: Optional access lists][eip2930]

[7] [EIP-2565: ModExp gas cost][eip2565]

[8] [EIP-3529: Reduction in refunds][eip3529]

[9] [RSKIP-543: Implement EIP-2718 Style Typed Transactions in Rootstock][rskip543]

[10] [RSKIP-243: Intra-transaction Gas Refunds][rskip243]

[11] [Martin Holst Swende, "Analysing the effects of EIP-2929"](https://github.com/holiman/eip2929-stats)

[12] [Security implications of EIP-1884][eip1884sec]

[13] [safe-contracts issue #149, "With istanbul it is not possible to use `send` or `transfer` to send funds to a Safe"][safe149]

[14] [Consensys Diligence, "Stop Using Solidity's transfer() Now"][stoptransfer]

[15] [folia-app/eip-2929, fixing `.transfer()` to Gnosis Safe with an access list][folia2929]

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
[rskip243]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP243.md
[rskip543]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP543.md
[eip1884sec]: https://github.com/holiman/eip-1884-security
[safe149]: https://github.com/safe-global/safe-contracts/issues/149
[stoptransfer]: https://diligence.security/blog/2019/09/stop-using-soliditys-transfer-now/
[folia2929]: https://github.com/folia-app/eip-2929
[execspecs]: https://github.com/ethereum/execution-specs/tree/master/tests/berlin/eip2929_gas_cost_increases
[tests649]: https://github.com/ethereum/tests/pull/649

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

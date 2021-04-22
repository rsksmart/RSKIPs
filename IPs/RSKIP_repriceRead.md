# Reprice Trie Read Opcodes

|RSKIP          |pull_request_number_here           |
| :------------ |:-------------|
|**Title**      |Reprice Trie Read Opcodes |
|**Created**    |20-APR-21 |
|**Author**     |SDL, SM|
|**Purpose**    |Sca, Sec, Fair|
|**Layer**      |Core|
|**Complexity** |1|
|**Status**     |Draft|

## Abstract
This RSKIP proposes increasing the `gasCost` for some trie read opcodes to better reflect actual use of computing resources.

## Motivation

Executing transactions requires accessing (and modifying) the latest snapshot of *blockchain state*: this includes account balances, nonces, contract code, and contract storage. In RSK, state is stored using a single binary prefix tree called the *unitrie*. In the reference client imlementation (RSKJ), the unitrie is backed up on disk using a LevelDB key-value datastore. Only some portions of the unitrie are loaded on to RAM as needed to execute transactions.

Recent experiments [0] indicate that some EVM trie opcodes are *underpriced* relative to actual resource use. For example, the current state of the RSK network can be read from disk  at the rate of about 400 unitrie nodes per second.  Caching some nodes in memory leads to a 10X increase in *lookup speed*  to about 3000 nodes/sec. Loading the *entire current state* of the RSK mainnet (in a single binary file) increases the lookup speed by 100X, relative to reading from disk, to around 35000 nodes/sec. The tradeoff is that this requires an additional 250MB of RAM to store the entire state snapshot in memory.

There is no unique way to link execution resource use and resource pricing (gas costs). So we have to approximate this relationship. Suppose that `1 gas` is about `30 nanoseconds` of VM execution time. Naturally, this is hardware dependent, see Rationale section below. Even with the most favorable scenario of keeping the entire trie in memory (35K lookups/sec),  each lookup takes nearly 28500 nanoseconds. Thus, a trie node lookup has an *impilicit cost* of about 950 gas (using 1 gas = 30 ns). However, the current cost of `BAL` is 400 gas while a `SLOAD` costs just 200 gas. Both of these codes are used quite frequently. Recall that reading from disk is much slower, about 400 nodes per second. That implies an implicit cost of 83000 gas for a lookup.

Thus, depending on how much state is cached in RAM, the implict cost of an `SLOAD` (priced at 200 gas) can vary from 1000 gas to 83000 gas. Again, this is assuming a 1:30 ratio between gas and execution time in nanoseconds. Correcting underpriced operations to reflect their actual resource consumption improves security (from attacks), scalabity and fairness.

## Specification

This proposal requires a hard fork and needs to be a part of a future network upgrade - activation block height to be determined.

|OpCode         |Current cost | New Cost |
| :------------ |:------------|:----     |
| `SLOAD (0x04)`  | 200 gas | 800 gas |
| `BAL (0x31)` | 400 gas | 700 gas |
| `EXTCODEHASH (0x3F`) |400 gas | 700 gas |


The revised gas cost structure will resemble that of Ethereum post Istanbul, see [EIP-1884](https://eips.ethereum.org/EIPS/eip-1884) [2]. Note that one component of EIP-1884, the `SELF-BALANCE` opcode, has previously been implemented in RSK. 

It is not our goal to match Ethereum's gas structure. Indeed, the current gas schedule in Ethereum has now evolved even further post EIP-2929, which increased some of the above costs by 3X (of what we are proposing now). Rather, we propose these changes now because they make sense for RSK, especially in light of recent trie experiments and as the RSK community starts planning for the next network upgrade.

## Rationale

### Gas and resource use
We can use experiments to *calibrate* gas for various operations in terms of some resource metric - such as execution time. One way to track resource use is through `VMPerformanceTest` which is part of RSKJ reference implementation [1]. For example, running this test on an Amazon AWS EC2 `M5a.large` instance (with 2 vCPUs and 8GB RAM) provides an average ratio of about `50 ns  per gas`. This estimate changes to `1 gas = 30ns` on faster (but still "consumer grade") machines.

Another way to think about resource use is to take a ratio of `blockGasLimit` (currently 6.8M gas) to some maximum `maxBlockExecutionTime` (e.g. 500 milliseconds). This provides a ratio of around `1 gas = 75ns`.

As node software and network usage evolve over time, the gas schedule can be periodically updated using new measurements. Further increases in trie read costs may be needed in the future with increase in state size and disk IO overhead.

## Backwards Compatibility

As already mentioned, this change needs a hard fork. In most cases, increasing the `gaslimit` is enough to account for repricing. RPC method to `estimate_gas` will provide reasonable guidance. However, increasing opcode costs can lead to some breaking changes.

*Value-transferring* `CALL`s cost 9000 gas. From this 9000, an amount 2300 gas is subracted as a *call stipend* and passed to the *receiving* contract. Solidity automatically adds the stipend as a gaslimit for the internal CALL.

The call stipend is not high enough to modify state e.g. no `SSTORE`. This helps avoid re-entrancy type attacks. The stipend is intended to cover the cost for logging the value transfer. This RSKIP does not change any logging costs and most transfers should not be impacted. However, if the receiving contract performs *three or more*  `BAL` or `SLOAD` operations as part of logging, then such CALLs will fail.

Similarly, contracts that make CALLs to external contacts with specific gas limits can also fail. Such patterns have been discouraged for quite some time now, at least since EIP-150, which increased awareness that gas costs can change  and developers should not make their execution logic rely on the gas schedule. 

We think such breaks will be rare in RSK. We also note that the Ethereum community has deliberated and implemented potentially breaking changes multiple times (EIP-150, EIP-1884, EIP-2929 [3]).

## Implementations

None.

## Test Cases

Should verify that the gascost of `SLOAD`, `BAL` and `EXTCODEHASH` are updated as proposed at the correct activation block height (to be determined)

## References

[0] Sergio Demian Lerner, IOV Labs, reported April 2021.

[1] See [https://github.com/rsksmart/rskj/blob/master/rskj-core/src/test/java/co/rsk/vm/VMPerformanceTest.java](https://github.com/rsksmart/rskj/blob/9a56ee28c8aae31c8bda1334e44124f83d135956/rskj-core/src/test/java/co/rsk/vm/VMPerformanceTest.java)

[2] Martin Holst Swende, "EIP-1884: Repricing for trie-size-dependent opcodes," Ethereum Improvement Proposals, no. 1884, March 2019. Available: https://eips.ethereum.org/EIPS/eip-1884.

[3] Martin Holst Swende, "Analysing the effects of EIP-2929", Available: https://github.com/holiman/eip2929-stats

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
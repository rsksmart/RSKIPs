---
rskip: 61
title: Cache Oriented Storage Rent (collect at EOT version) 
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2018-05-03
---

# Cache Oriented Storage Rent (collect at EOT version)

|RSKIP          |61           |
| :------------ |:-------------|
|**Title**      |Cache Oriented Storage Rent (collect at EOT version) |
|**Created**    |03-MAY-2018 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft* |

# **Abstract**

This RSKIP proposes that contracts should pay storage rent, to reduce the risk of storage spam and to make storage payments more fair. At the same time this RSKIP discusses the limitations of storage rent due to the additional complexity and overhead that, in some cases, overweight the benefits.

# **Motivation**

One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever. There are almost no examples in real-world commerce where users acquire with a single non-recurring payment eternal rights over a property that requires continued maintenance and therefore implies a periodic maintenance cost to a third party. The cost of maintenance is low but non-negligible, as persistent data must be stored in SSD so access cost matches real cost. That is the case of blockchain state storage, The cost is multiplied by the number of state replicas in the network. In some cases space is given for free (e.g. google drive space), but this is because space is subsidized by other services the google user consumes. Also there is no guarantee Google will offer free space forever. It can be argued that full nodes are altruistic, and therefore they are willing to incur in any storage cost network demands. While this may have been partially true for Bitcoin nodes in the past, this altruistic behaviour can decrease. The number of Bitcoin nodes has been declining, while the number of Bitcoin users has increased considerably, meaning that new users are not willing to run full nodes more than old users. It is expected that block pruning and sharding techniques enable users to commit certain partial amount of storage, but not for the full blockchain. However, the verification of new blocks, more than the historic storage, is what defines a full node. To verify a block, a node needs the full state, or receive Inclusion proofs for all state data used. The sharding factor must be inversely proportional to the number of honest host a peer connects to, so if the state size grows, and other factors remain constant, the local storage must also grow. Therefore, in principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.  both in terms of monetary effort and the fact that no single user may have the incentive to carry out the task, whatever the cost is.

A well designed crowd-contract should have a revenue generation method for paying for the storage rent. For example, each crowd-contract operation should be accompanied by a payment in bitcoins to a special rent sub-account where the crowd-contract collects all rent oriented income. However, as most crowd-contracts are immutable, such revenue collecting method must be defined at day zero, and at that stage it will be unclear if the revenue model can sustain the memory rent. RSKIP21 presents the problems in depth with storage rent. The main problem is efficiency: most rent payments are micro-payments and therefore the cost to pay (the overhead) is excessive. Another problem is scalability: the system to collect rent may introduce new inefficiencies (such as repeated state writes) that interfere with the scalability plan.

Three approaches have been devised:

1. Contracts pay rent once every period using the DEPOSIT opcode.

2. An hibernation deadline is automatically postponed each time a contract is called. A call consumes gas to pay for the rent. The amount of rent gas is proportional to the unused delta time (current time minus last time the contract was accessed) and the contract persistent memory size (code and storage). If the call does not provide enough gas, then the call fails. If the contract is called and the delta time is greater than one year, then the contract is hibernated.

3. Same as before, but there is no hibernation. Nodes will have a multi-level cache to store contract data (account state, code and storage), so that when the pending rent is higher than two times the access cost to a slower storage, the data is moved, and when data is accessed, it is brought into the fastest cache. The exact thresholds when moving from one cache to another will be specified in another RSKIP, and are not subject to consensus rules because this is transparent to the network. A multi-level cache can consist of DRAM, an SSD drive and a HDD drive.

This RSKIP specifies the third approach, as it is the less interfering and can be made more easily compatible with pre-built Ethereum application. A previous attempt to implement this functionality (RSKIP52) collected rent after each CALL, however this method failed when combined with low rent exceptions to possible attacks involving the use of recursive CALLs to prevent rent from being paid, by moving persistent data to auxiliary location. This version collects rents when transaction ends processing. 

# **Specification**

When a transaction is executed, the address of every contract called is stored in a "called" map. Every contract that is created is stored in a "created" map. After the transaction has been fully processed, the called map is iterated, and a storage rent is collected for every call. In case the call does not change the storage or account state of the called contract (including the balance) then the rent will only be paid if the rent is higher than 10000 gas (see later for the formula to compute the rent). If the state of the called contract is changed, then the rent will be paid if the rent is higher than 1000. This protects the network from performing costly micro-transactions. 
This RSKIP does not interfere with the plan to add parallel transaction execution if the transactions are scheduled properly. 

The rent is paid by extending the transaction to add a new field "rentGas".  The total gas consumed by a transaction will be equal to the normal gas consumed plus the rent gas consumed. If the rent gas to consume becomes higher than rentGas, the transaction is aborted and reverted. In this case, the remaining standard gas is not consumed (compared with the OOG exception). The storage rent is however consumed in full.  If the transaction ends because of OOG exception, then only 25% of the storage rent is paid, but called contract states are not modified. The 25% of storage rent is paid to compensate the cost of accessing the contracts from the caches. As with normal gas, the full rentGas amount is deducted from the origin address and then the remaining is reimbursed at the end of the transaction processing. 

Each account/contract has a new field lastRentPaidTime. Let d be the timestamp of the block in which the call is executed. Both fields are given in seconds. 

The following pseudo-code illustrates how rent is computed and paid for each called address "dest".
```
if (d>lastRentPaidTime) {
    callRentGas =  (storageSize+codeSize+256)*(d-lastRentPaidTime)/2^21
    if ((dest modified its state) && (callRentGas>=1000)) || 
       ((dest NOT modified its state) && (callRentGas>=10000)) {
        dest.lastRentPaidTime = now
        consumeRentGas(callRentGas);
      }
}
```

There are 31536000 seconds in a year.  2^32 =4294967296. SecondsAYear/2^32= 0.00734. A byte pays 14.68 gas units a year. A simple account (without code) pays 3750 gas units/year. A simple account cannot consume rent more than four times a year.

This callRentGas is consumed from the transaction rentGas. If this becomes negative, the transaction is reverted. If it's still zero or positive, the value of lastRentPaidTime is updated.

The cost of a byte is 15.625 gas per byte a year.

The storage size (storageSize) is computed as 128*N where N is the number of entries in the storage trie.

If there are several calls in the same transaction or the same block, only the first call will pay, because the remainder will have (d==lastRentPaidTime)

When a contract is created, the lastRentPaidTime is set 6 months in the future. This means that some rent is prepaid. 

The opcodes EXTCODECOPY, EXTCODESIZE and BALANCE opcodes also must pay rent (therefore the addresses are stored in the called map), because they access other contracts.

The block gas limit does not apply to rents: the amount of rents paid in gas may be higher than the gas limit. Therefore the rent is an additional uncapped revenue stream for the miners.

The created map is scanned, and the lastRentPaidTime value of each contract is set to 6 months in the future (considering 30-day months).

## New Transaction Format

The transaction format is modified. Currently the transaction contains the following fields:
1. Nonce
2. GasPrice
3. GasLimit
4. ReceiveAddress
5. Value
6. Data
7. v
8. r
9. s

If the transaction has 10 fields or more, then field at index 10 (starting from 1) will correspond to the field rentGasLimit. The same size restrictions on the field gasLimit will apply to rentGasLimit. Also the rentGasLimit is subtracted in full from the sender's balance, and then the amount unspent is reimbursed. If the transaction does not specify a rentGasLimit, then rentGasLimit is assumed to be equal to the gasLimit. If the rent cannot be paid because the rentGasLimit is reached, then the transaction is reverted and all the gas consumed so far is deducted (not reimbursed) as if a REVERT opcode had been executed. 

## AccountState changes

Two fields are added to the account state. The first is *flags* (currently always zero) and the second is the *lastRentPaidTime*. If the lastRentPaidTime is zero, the field is not serialized in RLP. If the flags is zero and the lastRentPaidTime  is zero, neither flags nor lastRentPaidTime fields are serialized.

## New Receipt status values

If a transaction is reverted manually (REVERT), a new status of (-1) is recorded in the transaction receipt.
If a transaction is reverted because of standard OOG, the old empty-vector status is still used.
If a transaction is reverted because of rent OOG, a new status of (-2) is recorded in the transaction receipt.

If an VM instruction would generate simultaneously a standard OOG and a rent OOG execption, the standard OOG is reported.

## Future Impromenets

If a contract unpaid rent becomes higher than a certain very high threshold, the contract could be hibernated. 

Also this RSKIP can be combined with the SPV Compressed block propagation using state trie update batch (COBLOP)  method.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
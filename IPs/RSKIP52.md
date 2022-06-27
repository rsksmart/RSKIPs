---
rskip: 52
title: Cache Oriented Storage Rent
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-12-12
---

# Cache Oriented Storage Rent

|RSKIP          |52           |
| :------------ |:-------------|
|**Title**      |Cache Oriented Storage Rent |
|**Created**    |12-DIC-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft* |

# **Abstract**

This RSKIP proposes that contracts should pay storage rent, to reduce the risk of storage spam and to make storage payments more fair. At the same time this RSKIP discusses the limitations of storage rent due to the additional complexity and overhead that, in some cases, overweigh the benefits.

# **Motivation**

One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever. There almost no examples in real-world commerce where users acquire with a single non-recurring payment eternal rights over a property that requires continued maintenance and therefore implies a periodic maintenance cost to a third party. The cost of maintenance is low but non-negligible, as persistent data must be stored in SSD so access cost matches real cost. That is the case of blockchain state storage, The cost is multiplied by the number of state replicas in the network. In some cases space is given for free (e.g. google drive space), but this is because space is subsidized by other services the google user consumes. Also there is no guarantee Google will offer free space forever. It can be argued that full nodes are altruistic, and therefore they are willing to incur any storage cost the network demands. While this may have been partially true for Bitcoin nodes in the past, this altruistic behaviour can decrease. The number of Bitcoin nodes has been declining, while the number of Bitcoin users has increased considerably, meaning that new users are not willing to run full nodes more than old users. It is expected that block pruning and sharding techniques enable users to commit certain partial amount of storage, but not for the full blockchain. However, the verification of new blocks, more than the historic storage, is what defines a full node. To verify a block, a node needs the full state, or receive inclusion proofs for all state data used. The sharding factor must be inversely proportional to the number of honest hosts a peer connects to, so if the state size grows, and other factors remain constant, the local storage must also grow. Therefore, in principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.  both in terms of monetary effort and the fact that no single user may have the incentive to carry out the task, whatever the cost is.

A well designed crowd-contract should have a revenue generation method for paying for the storage rent. For example, each crowd-contract operation should be accompanied by a payment in bitcoins to a special rent sub-account where the crowd-contract collects all rent oriented income. However, as most crowd-contract are immutable, such revenue collecting methods must be defined at day zero, and at that stage it will be unclear if the revenue model can sustain the memory rent. RSKIP21 presents the problems in depth with storage rent. The main problem is efficiency: most rent payments are micro-payments and therefore the cost to pay (the overhead) is excessive. Another problem is scalability: the system to collect rent may introduce new inefficiencies (such as repeated state writes) that interfere with the scalability plan.

Three approaches have been devised:

1. Contracts pay rent once every period using the DEPOSIT opcode.

2. An hibernation deadline is automatically postponed each time a contract is called. A call consumes gas to pay for the rent. The amount of rent gas is proportional to the unused delta time (current time minus last time the contract was accessed) and the contract persistent memory size (code and storage). If the call does not provide enough gas, then the call fails. If the contract is called and the delta time is greater than one year, then the contract is hibernated.

3. Same as before, but there is no hibernation. Nodes will have a multi-level cache to store contract data (account state, code and storage), so that when the pending rent is higher than two times the access cost to a slower storage, the data is moved, and when data is accessed, it is brought into the fastest cache. The exact thresholds when moving from one cache to another will be specified in another RSKIP, and are not subject to consensus rules because this is transparent to the network. A multi-level cache can consist of DRAM, an SSD drive and a HDD drive.

This RSKIP specifies the third approach, as it is less interfering and can be easily compatible with pre-built Ethereum applications.

# **Specification**

In case the call do not change the storage or account state of the target contract (including the balance) then the rent will only be paid if the rent is higher than 10000 gas. If the state of the called contract is changed, then the rent will be paid if the rent is higher than 1000. This protects from costly micro-transactions. 
This RSKIP does not interfere with the plan to add parallel transaction execution if the transactions are scheduled properly. 

The rent is paid by extending the transaction to add a new field "maximumRentGas".  The total gas consumed by a transaction will be equal to the normal gas consumed plus the rent gas consumed. If the rent gas to consume becomes higher than maximumRentGas, the transaction is aborted with OOG. As with normal gas, the full maximumRentGas amount is deducted from the origin address and then the remaining is reimbursed. 

Each account/contract has a new field lastRentPaidTime. Let d be the timestamp of the block in which the call is executed. Both fields are given in seconds. 

The following code ilustrates how rent is prePaid in full when transaction starts execution and when it finishes:
```
// called on Start
void prepayAllRentGas(transactionMaximumRentGas) {
 sourceAccount.subtract(transactionMaximumRentGas*gasPrice);
 availableRentGas = transactionMaximumRentGas
}

// Called during execution
void consumeRentGas(gas) {
 rentGasConsumed +=gas;
 availableRentGas -=gas;
}
// Called during execution
void reimburseRentGas(gas) {
rentGasConsumed -=gas;
 availableRentGas +=gas;
}

// Called on finish
void finalizeRentGas() {
 sourceAccount.add(availableRentGas*gasPrice);
 minerAccount.add(rentGasConsumed *gasPrice);
}
```

The following pseudo-code ilustrates how rent is computed and paid.

```
if (d>lastRentPaidTime)
  rentGas =  (storageSize+codeSize+256)*(d-lastRentPaidTime)*2000/2^32
else
  rentGas = 0
  
consumeRentGas(rentGas)
dest_contract.call()
if (dest_contract modified its state) {
  if (rentGas<1000) 
    reimburseRentGas(rentGas)
   else
    dest_contract.lastRentPaidTime = now

} else {
  if (rentGas<10000)
    reimburseRentGas(rentGas)
  else
   dest_contract.lastRentPaidTime = now
}
```

There are 31536000 seconds in a year.  2^32 =4294967296. SecondsAYear/2^32= 0.00734. A byte pays 14.68 gas units a year. An simple account (without code) pays 3750 gas units/year. An simple account cannot consume rent more than four times a year.

This rentGas is consumed from the maximumRentGas. If this becomes zero, the out-of-gas exception is raised. On success, the value of d-lastRentPaidTime is updated).

The cost of a byte is 15.625 gas per byte a year.

The storage size (storageSize) is computed as 128*N where N is the number of entries in the storage trie.

If there are several calls in the same transaction or the same block, only the first call will pay, because the remainder will have (d==lastRentPaidTime)

When a contract is created, the lastRentPaidTime is set 6 months in the future. This means that some rent is prepaid. 

The opcodes EXTCODECOPY, EXTCODESIZE and BALANCE opcodes also must pay rent, because they access other contracts.

The block gas limit does not apply to rents: the amount of rent paid in gas may be higher than the gas limit. Therefore rent is an additional uncapped revenue stream for the miners.

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

If the transaction has 10 fields or more, then field at index 10 (starting from 1) will correspond to the field maximumRentGas. The same size restrictions on the field gasLimit will apply to maximumRentGas. Also the maximumRentGas is subtracted in full from the sender's balance, and then the amount unspent is reimbursed. If the transaction does not specify a maximumRentGas, then maximumRentGas is assumed to consume all remaining sender's balance: the account balance minus the value transferred, divided by the GasPrice specified. 
If the rent cannot be paid because the maximumRentGas is reached, then the transaction is reverted and all the gas consumed so far is deducted (not reimbursed) as if a REVERT opcode had been executed. 

## New Receipt status values

If a transaction is reverted manually (REVERT), a new status of (-1) is recorded in the transaction receipt.
If a transaction is reverted because of standard OOG, the old empty-vector status is still used.
If a transaction is reverted because of rent OOG, a new status of (-2) is recorded in the transaction receipt.

If an VM instruction would generate simultaneously a standard OOG and a rent OOG execption, the standard OOG is reported.

## Future Improvements

If a contract's unpaid rent becomes higher than a certain threshold, the contract could be hibernated. 

[RSKIP21]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP21.md# 

**Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
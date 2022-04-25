---
rskip: 21
title: Efficient Persistent Storage Rent
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-12-02
---

# Efficient Persistent Storage Rent

|RSKIP          |21           |
| :------------ |:-------------|
|**Title**      |Efficient Persistent Storage Rent|
|**Created**    |02-DIC-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes that contracts should pay storage rent, to reduce the risk of storage spam and to make storage payments more fair. At the same time this RSKIP discusses the limitations of storage rent due to the additional complexity and overhead that, in some cases, overweight the benefits.

# **Motivation**

One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever. There almost no examples in real-world commerce where users acquire with a single non-recurring payment eternal rights over a property that requires continued maintenance and therefore implies a periodic maintenance cost to a third party. The cost of maintenance is low but non-negligible, as persistent data must be stored in SSD so access cost matches real cost. That is the case of blockchain state storage, The cost is multiplied by the number of state replicas in the network. In some cases space is given for free (e.g. google drive space), but this is because space is subsidized by other services the google user consumes. Also there is no guarantee Google will offer free space forever. It can be argued that full nodes are altruistic, and therefore they are willing to incur in any storage cost network demands. While this may have been partially true for Bitcoin nodes in the past, this altruistic behaviour can decrease. The number of Bitcoin nodes has been declining, while the number of Bitcoin users has increased considerably, meaning that new users are not willing to run full nodes more than old users. It is expected that block pruning and sharding techniques enable users to commit certain partial amount of storage, but not for the full blockchain. However, the verification of new blocks, more than the historic storage, is what defines a full node. To verify a block, a node needs the full state, or receive Inclusion proofs for all state data used. The sharding factor must be inversely proportional to the number of honest host a peer connects to, so if the state size grows, and other factors remain constant, the local storage must also grow. Therefore, in principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.  both in terms of monetary effort and the fact that no single user may have the incentive to carry out the task, whatever the cost is.

Ideally, a well designed crowd-contract should have a revenue generation method for paying for the storage rent. For example, each crowd-contract operation should be accompanied by a payment in bitcoins to a special rent sub-account where the crowd-contract collects all rent oriented income. However, as most crowd-contract are immutable, such revenue collecting method must be defined at day zero, and at that stage it will be unclear if the revenue model can sustain the memory rent. The simplest case is that every user pays rent independently for the memory it consumes. For example, a DNS-like contract will attach an account balance to every name registered, and users would be free to send money to pay the rent for their registered names. When the time comes to pay the rent, the contract would see which names have enough balance, remove the names that cannot pay for the rent, subtract a fixed amount from each name balance, and pay the rent using LIFE_EXTEND-like opcode  (see [RSKIP08]). However, this approach may be highly inefficient as each payment may represent a hundredth of a cent. As long as individual rent-payment transactions are bundled with other (more important) transactions, the system works. However, users not frequently using the crowd-contract will be forced to send transactions with the sole motive to pay the rent-share. Therefore a probabilistic approach, where 1% of the users are pseudo-randomly chosen to pay the rent for all the users at a given period, seems more adequate.


# **Specification**

See discussion [here](https://github.com/rsksmart/RSKIPs/issues/79)

Every contract (not accounts) has four new fields:  **rentPreDeposit, rentDue, rentNext, rentEndTimestamp.**. The first three fields hold values in gas. The last field is a block number.

The rent payment algorithm is as follow: 

-Let t be the current date.

-Let d be the date of the last message sent to a contract.

-Let s be the amount of memory consumed by persistent memory

-Let payTime be equivalent to 4 months in blocks. During this period the owner has time to pre-deposit the the due rent, and if the pre-deposit is higher than the due amount, the due rent is paid immediately. 

-Let graceTime is equivalent to 2 months. This period starts just after rentEndTimestamp.

After the graceTime is over, the contract can be hibernated with the HIBERNATE opcode. If self or external hibernation is not requested, the contract behaves normally.

After the the payTime, the contract will hibernate either by external hibernation or by any message sent to it.

The following diagram depicts these intervals:

<img src="../RSKIP21/intervalDiagram_1RSKIP21.png">


Then when a message or call at time t arrives to a contract X and needs to be processed,  the following procedure is executed.
```
1. If (t<rentEndTimestamp) 
or  ((d>=rentEndTimestamp) and (d<rentEndTimestamp+payTime))
    1. r = s*rentCost/(t-d)
    2. rentNext = rentNext+r
2. else if (t>=rentEndTimestamp)
    3. r1 = s*rentCost/(rentEndTimestamp-d) 
    4. rentNext  = rentNext  + r1
    5. rentDue = rentNext
    6. rentNext =0.
    7. r2 = s*rentCost/(t-rentEndTimestamp)
    8. rentNext  = rentNext  + r2
3. if (t>=rentEndTimestamp+payTime) then 
    9. rentDue = rentDue+nextRent
    10. nextRent = 0
4. hibernate contract X 
5. else if  (t>=rentEndTimestamp)
    11. if (rentPreDeposit>=rentDue) 
        1. rentPreDeposit =0
        2. rentDue =0
        3. rentEndTimestamp = rentEndTimestamp + period
```
The cases considered by step 1 are depicted below. The light blue bar represents the amount of rent that is paid.

<img src="../RSKIP21/case1aRSKIP21.png">


<img src="../RSKIP21/case1bRSKIP21.png">

The case considered by step 2 is depicted below. 

<img src="../RSKIP21/case2RSKIP21.png">

The case considered by step 3 is depicted below. 

<img src="../RSKIP21/case3RSKIP21.png">


In this last case, the storage rent is split into two amounts: the first amount corresponds to the interval painted in light blue and ending with the rentEndTimestamp that is added to the previous payment period, the second amount painted in yellow correspond to the interval that starts afterwards. 

It must be noted that the case considered by step 4 is not exclusive of other cases, and can be entered after case 2b, 2 or 3 has been entered.

Any contract can execute HIBERNATE <dstcontract>. When dstcontract is not the self contract, and is not an account, the following actions happen:

* Let x  be the time the rentEndTimestamp has been surpassed.

* if x is negative no further action is taken

* If (x>=graceTime) then 

    * rentDue = rentDue - rentPreDeposit

    * rentPreDeposit =0

    * dstcontract is hibernated 

The HIBERNATE opcode returns 1 if hibernation was successful and 0 if not.

When a dstcontract is hibernated, the cost of hibernation is set to zero. If not, then the opcode HIBERNATE pays a cost for the performed checks, equal to at least 300 gas units.

On each hibernation the pre-deposit MUST be emptied, even if it contains more than the rentDue amount. To reduce the uncertainty of the amount of gas that should be pre-deposited, the rentDue is "locked" on the previous deadline.

It is suggested that the deadline interval is 6 months, so the user has at most 6 months to pay from the moment of the memory usage, and 6 months from the moment he knows exactly how much he has to pay. 

At any time the contract can make a pre deposit in gas using the opcode DEPOSIT_RENT <gasAmount>.

If hibernate is called for an account, the following actions are taken:

- if (t-lastSendTime>OneYear) hibernate account, and return 1

- else, return 0 

If a contract self-hibernates, no rent needs to be paid, so the contract hibernates and HIBERNATE returns 1.

Generally the contract will implemement a method such as:
```
public PayMyRent(int amount){

 DEPOSIT_RENT(this,amount)

}
```
When the contract is hibernated, both the code, memory and balance are hashed and only a single hash digest is stored in the contract address. While the contract is hibernated it cannot receive nor send payments or messages except a special WAKEUP message. It can receive payments into a new account with the same name, and when the contract is woke up both balances are added. The user can awake the contract by sending the WAKEUP message containing the full code and persistent data. If the code and data does not fit into a message then the user needs to create a proxy contract that composes the code/data in chunks, append the chunks, and sends the contract a wakeup WAKEUP message using the WAKEUP opcode. The WAKEUP opcode has several arguments: the code, the code size, the data, the data size. If data size is set to zero, but data pointer is non-zero, it is interpreted as the address of a storage cell where to take to code from. If WAKEUP returns an error code if the contract could not be woken up: the code 2 means that the code was invalid, 3 means that the data was invalid, and 4 means that the rent is too low for paying the re-hibernation cost.

## Split Storage

Contract storage is divided into two subspaces. A first subspace persists hibernations. All addresses in this subspace begin with 0x00 following 256 bits. The second subspace begins with 0x01, following 256 bits. 

To select the subspace to write to, the new SSPACE opcode is introduced. By default, if no SSPACE opcode is executed, the subspace selected is 0x00.

## HIBERATION cost 

The cost of hibernation does not depend on the size of the memory, since this RSKIP will be implemented on top of the new Trie structure (persistent memory below account address on the Trie). Therefore the root hash of the memory subtree need not be computed.

If hibernate is called and it returns 1 (meaning an hibernation has taken place) HIBERNATE opcode consumes no gas at all. If HIBERNATE is called but it returns 0 (hibernation not reched) the opcode cost is 300 gas units. Internally, the 300 gas cost is first deducted, and then given back in case the hibernation succeds. 

## The problem with Ethereum SSTORE costs

The following table show previous SSTORE costs and new costs:

<table>
  <tr>
    <td>Identifier</td>
    <td>previous cost</td>
    <td>new net cost</td>
    <td>when is paid</td>
  </tr>
  <tr>
    <td>SET_SSTORE </td>
    <td>20000 </td>
    <td>2000</td>
    <td>from null to non-zero</td>
  </tr>
  <tr>
    <td>RESET_SSTORE </td>
    <td>5000</td>
    <td>300 or 600</td>
    <td>from zero to zero, or from non-zero to non-zero. 
</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE </td>
    <td>15000</td>
    <td>--</td>
    <td>from non-zero to zero
(refunded in the future)</td>
  </tr>
  <tr>
    <td>CLEAR_SSTORE </td>
    <td>5000 </td>
    <td>50</td>
    <td>from non-zero to zero
</td>
  </tr>
  <tr>
    <td>CODEBYTE</td>
    <td>200</td>
    <td>20</td>
    <td></td>
  </tr>
</table>


First, the rationale behind CLEAR_SSTORE is that by zero-ing a cell it is actually deleted, so space is saved. However, to determine that a cell has been deleted, it must be first read, and reading pertain a high cost of disk access. All current SSTORE actions require the cell value to be previously read, which is in most cases unnecessary. Reads are blocking: the code cannot proceed execution until the address is fetched. However writes are non-blocking: writes can be cached and executed at the end of the contract.

The underlying problem is that Ethereum does not clearly states if contract storage is pre-loaded when the contract is called, or storage cells are loaded on-demand. Pre-loading does not seems the right approach, as to do this in an optimized fashion requires compacting all contract storage space into a single consecutive disk data chunk. Compacting storage is expensive, and that cost is not paid by SSTORE. 

In RSK platform the premise is that each cell read may require a disk access. To prevent unnecessary pre-reads, all SSTORE operations consume a constant gas cost, and when the contract finishes some refunds are made depending on the existence of pre-existing cells.

<table>
  <tr>
    <td>Identifier</td>
    <td>value</td>
    <td>net cost</td>
    <td>when is paid</td>
  </tr>
  <tr>
    <td>SSTORE </td>
    <td>2000</td>
    <td></td>
    <td>at execution</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_Z_NZ</td>
    <td>0</td>
    <td>2000</td>
    <td>from null to non-zero (refund)</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_Z_Z</td>
    <td>1700</td>
    <td>300</td>
    <td>from zero to zero
(refund)</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_NZ_NZ</td>
    <td>1400</td>
    <td>600</td>
    <td>from non-zero to non-zero
(refund)</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_NZ_Z</td>
    <td>1950</td>
    <td>50</td>
    <td>from non-zero to zero
(refund)</td>
  </tr>
  <tr>
    <td>SSTORE_YEARLY </td>
    <td>200</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>CODEBYTE</td>
    <td>20</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>CODEBYTE_YEARLY </td>
    <td>20</td>
    <td></td>
    <td></td>
  </tr>
</table>


It is important that the implementation checks if a cell address exists in the trie without retrieving its value. The RESET_SSTORE cost is split into two new costs: ZERO_ZERO y NZERO_NZERO.

The costs of SSTORE is modified. When SSTORE adds a value to memory, the cost of space is pre-deducted, along with an additional cost. If the same cell is later set to zero, the cost of space is returned back to the gas pool. However, if a cell is set to zero by a different transaction, then the pre-deducted gas is not returned: the SSTORE cost is set to zero, but it does not pay back gas.

## CREATE_DATA (CODEBYTE) cost

Currently every byte of code added pays 200 gas units. This is too high as code stays together and can be stored in a single chunk of disk space. A  cell of 32 bytes of SSTORE data is being priced at 2000 units of gas, but it actually occupies in memory approximately 256 bytes (address+data+child_pointers_overhead+hash=~256). So SSTORE data is priced at 7 units of gas per byte. Therefore CREATE_DATA will be reduced 10-fold, to 20 gas units per byte. The recurrent price will be lower: 10 gas units per byte.

A different problem is the cost of loading the code when a contract is called. Loading from SDD/HDD a single byte or a chunk of 36 Kbytes takes the same for a modern OS, since data is stored in blocks. Therefore reducing the code cost per byte does not open a new vector of attack unless there is no limit on program size on CALLs. The problem of program size can be solved in several ways: 

- establishing an upper bound (as Ethereum has done)

- paginating program memory.

The first approach forces the compiler to split a long program into libraries, and use DELEGATECALL as a means to call between them. The problem with this approach is that DELEGATECALL costs 700 gas, while a normal call costs only 8 gas. This means that the split into libraries must obey a structural rule, such that there are much more inter-library calls than cross-library calls. 

RSK does memory pagination. When a contract is loaded, only the first page (up to 8 KBytes) is loaded. Jumps that are outside the current loaded page are not checked beforehand, but all addresses that are the destinations of a jump are collected in the address list.  A page cannot end in the middle of data, each page must end with a valid instruction. Only when the program counter wraps around a page or when a JUMP that points out of a page is executed then the missing page is loaded. The first time a page is loaded the VM performs the static checks. It first scans the code collecting all destination addresses into the same address list, and then scans finds the addresses that belong to the loaded page and check that each destination address has a JUMPDEST opcode. If an address does not have such opcode, an out-of-gas exception is generated

In RSK all the static checks are performed when a contract is created. At that time two checks are performed: first the initialization code (containing the payload) and then the payload itself (when it is returned). Since the cost of each code byte is 20 gas, if there is a 4M gas limit, the maximum code size is 200K bytes. 

Afterwards, on calls, the contract can be freely executed without checks. When the program counter wraps around a page or when a JUMP that points out of a page is executed then the missing page is loaded and a cost of 300 gas units is consumed. The page remains in memory until the call ends. A 4M block gas limit allows up to 500 pages to be loaded, totaling 4M bytes of code. The user must however take into account that if the gas limit is reduced, a large contract may be prevented from being called.

## Library problems

One of the problems of storage rent is that is discourage the creation of libraries. Who is going to pay for a library ?

The short answer is that a library must implement a rent collection method based on estimating the number of calls per rent period (NC) (at design or run time) and dividing the rent cost in the first NC calls to the library. This housekeeping consumes at least writing to a storage cell, or 2000 gas units, and therefore is only suitable for libraries  where rent share > 2000 (or 100 bytes of code). As an example, a library with 100 Kbytes, that is used 100 times a year minimum, prices each one of the first 100 calls at 50 gas units. 

In case a library goes to hibernation, this is highly inconvenient for a decentralized user, since he may not know exactly why the library fails, or which other libraries are called that are not available anymore. The whole idea of libraries that call libraries, where the system liveness depends on the rate of calls received,  seems flimsy, to say the least.

One possibility is to create a special entity for libraries. Libraries do not posses storage nor balance, and have a byte cost of 1000 (5 times more than in Ethereum and 50 times more than RSK initial byte cost). With a 4M gas limit, the maximum size of a library would be 4 Kilobytes.

The same approach could be used to support eternal contracts, by increasing the SSTORAGE cost to 50K (25 times more than RSK initial cost). 

## WAKEUP cost

To wake up a contract, the user must provide the spv path and the required hash pre-image. Transferring that data in a transaction has a cost of x per byte. The WAKE UP opcode itself also has a costs that comprises the following costs:

* Code byte cost  (20 gas units per byte)

* Storage cell cost (2000 units per cell)

* fixed cost to recover balance and other contract internal fields (2000 units)

Also the memory considered by the rent always have a fixed initial cost of 64 bytes, or 128 units of gas.

## Can Self-Hibernation save money? Short answer: No.

To see how hibernation can save money in certain cases, imagine a contract with the following properties:

- Code size: 1024 bytes

- Storage size: 4 cells

- Total bytes to transfer (without cell addresses): ~640 bytes

The contract is programmed so it self-hibernates after each operation. This means that the rentCost is always zero. The cost of maintaining this contract active for 1 year is : 1024*2+8*256*2+128=6272 units of gas.

The cost of awakening is 20*1024+4*2000+2000=30480.

The cost of transfer every non-zero byte is 68 gas units. Therefore transferring 640 bytes in every transaction costs 43520. Clearly there is no benefit to do it unless the contract is going to be inactive for 12 years.

Also it’s clear than the penalty for not paying is 12 times more than the amount to be paid, which seem exaggerated.

Self-hibernation could save money if data could be transferred at lower cost. One of such methods would be ephemeral segwit data, which can only be used by contracts two blocks after inclusion. Blocks would be transferred in two phases, the first without segwit, and the second only segwit data. This allows a miner do validate a block while the segwit data is being received. If validation takes 1 second, then data can be received though that one second at no cost. The cost of that data would be the cost of temporary blockchain storage. If that data is kept in a RAM cache, then the cost per byte could be as low as 1 units of gas.

This means that a block with a 4M gas limit can contain  4M bytes of data, and the data must be transmitted in 1 second, which exceeds the normal capacity of links. Therefore we must set a price higher: e.g. 5 gas units per byte.

The cost of a gas unit is 1.4*10^-7 USD.

When we compute the cost of storage, we must multiply by 1000 (the desired number of nodes in the newtok).

Then the cost of permanent blockchain storage for each byte in HDD is 2.4 * 10^-11. Multiplied by 1K nodes is 2.4 *10^-8  (7  times less)

The cost of permanent storage in SDD is 5.5 * 10^-11.  Multiplied by 1K nodes is 5.5 * 10^-8 (3 times less)

The cost of storing in a RAM cache is 1.6 * 10^-10 (for 4 years). If it’s going to be in cache for just 1 day, then the cost is 1.6 /1460 * 10^-10  is 1.09*10^-13.   Multiplied by 1K nodes is    1.09*10^-10 (1000 times less).

Therefore the cost of storage is much lower than the cost of transfer in segwit. The cost that prevails is always the cost of disk access: if the contract data must be retrieved from disk, it costs 1 msec to access, then for a 4M gas block limit, it should cost 4000 units of gas. Clearly the segwit data MUST be held in memory until it is used or disposed.It can be written to disk, but this must not block mining and transaction processing.

It seems that the segwit system can be used to restore state, HOWEVER it requires two transactions instead of one: the first to provide the segwit data (and pay for it) and the second to execute the wakeup command using the segwit data. Because each transaction costs  21K gas, the whole process costs 21K gas more, only to save 6272 units of gas!

One way to reduce the cost of the second transaction is by allowing a transaction to be pre-paid. This is done by including in a transaction a hash of the following transaction.

Therefore, if those hashes can be kept for some time in memory, following transactions can be very cheap.

Adding this functionality is complex, unless we change the signature system so that a contract can verify it. Therefore we could change the contract with the first transaction to accept the second or many more without signature (only by its hash). But the signature would not include the gasPrice, since this can change from block to block.  The problem is that if a miner rejects it, the transaction will be always valid ? A transaction would also need an increasing sequence number.

Clearly all this complex stuff is super interesting, but useless.



[comment]: <> (## Changes to nonZeroDataBytes( Cost Instead of counting non-zero data bytes, the platform will count zero-blocks of data. Each zero-block must (CHECK, UNFINISHED)


## Security

This design is only is secure if:

* There is always a reasonable minGasPrice. No contract can pay lower than this price. This is to protect a miner from buying eternal memory at no cost.

* To prevent services that offer memory to avoid rent, the cost of memory should not be higher than the cost of transfer it from one contract to another. 

* As times rent are measured by block number, the costs remain acceptable if the block interval does not suffer a high and persistent deviation.

## New opcodes

Let m be the amount of memory persisted by the contract in 32 byte words.

### HIBERNATE

Arguments: <address> 

GasCost: provided by the caller (not taken from the hibernation deposit) 

returns: error_code

### WAKEUP

Arguments: contract_address spv_path_address spv_path_length code_address code_size trie_address trie_size

Returns: error_code

### DEPOSIT_RENT 

Arguments: <address>

Value: amount to deposit. It must be equal or higher than m*f

Value accepted: m*f

### SSPACE

Arguments: <space>

Changes the memory space.

Possible values for space: { 0,1}.

All SLOAD and SSTORE operations following SSPACE apply to the selected space.

### GASDEPOSITED

Returns the amount of gas in the deposit.

### RENTDUE

Returns the amount of gas that must be payed in the next deadline.

### RENTNEXT

Returns the amount of gas that is accumulating to be added to the rent due in the next deadline. 

### GASCOST

arguments: <gasIndex>

Returns: the cost of a gasIndex, such as an opcode.

### SMEMSIZE

Returns: the size of persistent memory.

Due to the number of opcodes required, if may be preferable to implement them by one or several native contracts. In this case, it must be evaluated if the cost of call exceeds or not the desired cost of the operation.

### Unowned Contracts 

A contracts that does not have an owner must make sure the users that interact with the contract pay the rent. One way to achieve this is by tracking the time each user has made a partial rent payment and detect when too much time has elapsed since the last payment. Contracts are able to known how much gas is in the deposit with the opcode GASDEPOSITED, and also they can know how much memory is being held with the SMEMSIZE opcode.

The following pseudo-code collects rent shares from each method call:

SMEMSIZE			(cost 2)

BLOCK.TIMESTAMP		(cost 2)

SLOAD LastCallTimeStamp  	(cost 200)

SUB			           	(cost 3)

MUL				(cost 5)

DEPOSIT 			(cost 5)

BLOCK.TIMESTAMP		(cost 2)

SSTORE LastCallTimeStamp (20K in Ethereum, 2K in RSK)

The cost of executing this code snippet in RSK is : 2220 units.

This is equivalent to the storage of:

<table>
  <tr>
    <td>Size</td>
    <td>Equivalent cost of 2200 gas units in rent Time </td>
    <td>Type of contract</td>
    <td>Frequency of use expected</td>
  </tr>
  <tr>
    <td>1.1 Kbyte</td>
    <td>1 year</td>
    <td>Very simple wallet</td>
    <td>1 per month</td>
  </tr>
  <tr>
    <td>11 Kbyte</td>
    <td>36 days</td>
    <td>Multi-sig wallet
(not taking into account code reuse)</td>
    <td>1 per 10 days</td>
  </tr>
  <tr>
    <td>110 Kbyte</td>
    <td>3.6 days</td>
    <td>DAO</td>
    <td>1 per day</td>
  </tr>
  <tr>
    <td>1 Mbyte </td>
    <td>8.6 hours</td>
    <td>Registry for 3500 users</td>
    <td>1 per hour</td>
  </tr>
  <tr>
    <td>10 Mbytes</td>
    <td>51 minutes</td>
    <td>Registry for 35K users </td>
    <td>1 per minute</td>
  </tr>
  <tr>
    <td>100 MBytes </td>
    <td>5.1 minutes</td>
    <td>Crypto-asset for 390K users</td>
    <td>1 per 10 seconds</td>
  </tr>
  <tr>
    <td>1 GB</td>
    <td>30 seconds</td>
    <td>Crypto-asset for 3.9M users</td>
    <td>1 per second</td>
  </tr>
</table>


 

The table shows that the execution of the code snippet for paying rent in every incoming payment has a cost one order of magnitude higher than the amount paid. There are two solutions: either the rentCost is increased 10 times, or the platforms automatically provides the value LastCallWriteTimeStamp which indicates the last time the contract was called resulting in the contract state being changed. The platform would store this value along the changed account, so there is no cost in updating it. Also the platform will provide access to a flag contractModified which becomes true if the contract was modified in any way (balance, memory, etc.). Therefore this code snipped would be added at the end of every contract call:

ContractModified		(cost 2)

JUMPI exit			(cost 10)

SMEMSIZE			(cost 2)

BLOCK.TIMESTAMP		(cost 2)

LastCallWriteTimeStamp  	(cost 2)

SUB			           	(cost 3)

MUL				(cost 5)

DEPOSIT 			(cost 5)

BLOCK.TIMESTAMP		(cost 2)

exit:

JUMPDEST			(cost 1)

This has a total cost of 33.

<table>
  <tr>
    <td>Aprox Size</td>
    <td>Equivalent cost of 33 gas units in rent Time </td>
    <td>Type of contract</td>
    <td>Frequency of use expected</td>
  </tr>
  <tr>
    <td>165 bytes</td>
    <td>1 month</td>
    <td>Tiny wallet with embedded rules for anonymization</td>
    <td>1 per month</td>
  </tr>
  <tr>
    <td>1.65 Kbyte</td>
    <td>same as expected</td>
    <td>Very simple wallet with rate limits</td>
    <td>1 per month</td>
  </tr>
  <tr>
    <td>16.5 Kbyte</td>
    <td>same as expected</td>
    <td>Multi-sig wallet</td>
    <td>1 per 10 days</td>
  </tr>
  <tr>
    <td>165 Kbyte</td>
    <td>same as expected</td>
    <td>DAO</td>
    <td>1 per day</td>
  </tr>
  <tr>
    <td>1.6 Mbyte </td>
    <td>same as expected</td>
    <td>Registry for 3500 users</td>
    <td>1 per hour</td>
  </tr>
  <tr>
    <td>16 Mbytes</td>
    <td>same as expected</td>
    <td>Registry for 35K users </td>
    <td>1 per minute</td>
  </tr>
  <tr>
    <td>165 MBytes </td>
    <td>same as expected</td>
    <td>Crypto-asset for 390K users</td>
    <td>1 per 10 seconds</td>
  </tr>
  <tr>
    <td>1.6 GB</td>
    <td>same as expected</td>
    <td>Crypto-asset for 3.9M users</td>
    <td>1 per second</td>
  </tr>
</table>


With this pricing the cost of paying partial rents is similar to the cost of the rent paid.

## Sample Use Cases

### User-asset contract using child contracts

This is an ASSET contract where child contracts are used to increase parallelization factor. The ASSET contract will not store anything in its own persistent memory, but still needs to pay the cost of maintaining the ASSET contract alive.

Each child contract will be created with the address HASH ( parent-address |  user-address). Each child contract could have the following method:

 public payRent()

{

   if (gas<rentGas) thow;

   DEPOSIT_RENT(this,rentGas*5) // pays for code + data 

   DEPOSIT_RENT(parent,rentGas) // overpays for some constant code/data

}

Another option is that the parent contract implements the following method :

public payRent() {

  address a = msg.sender;

  address childContract = SHA3( this.address , a);

  DEPOSIT_RENT(childContract,rentGas);

  DEPOSIT_RENT(this,rentGas);

}

## Cost of CALL

Should the cost of a CALL account for the cost of restoring all persistent contract data into RAM? To answer this question it is required to analyze how contracts use storage and how the system would cache contract storage. On one side the cost of CALL is currently fixed, so it cannot account restoring all contract storage from disk into RAM. On the other side, paying for a disk access for every 32 bytes readed from memory is overkill. Using an unified tree for accounts and contract storage means than storage elements get stored on random places on disk, so there is no way all elements can be retrieved at the same time at lower cost. However, a storage cache is present, so after a first access subsequent accesses in the same transaction are executed much faster. Writing or reading to a storage cell a hundred times costs the same as accessing it a single time. If access cost in gas is constant, then contracts should use volatile memory as a cache to perform several operations until a single  SSTORE operation is made last for every cell accessed. Changing the cost of SSTORE/SLOAD depending on if it is the first time or not seems overly complex and prevents static code analyzers from easily inferring gas cost of a code block. 

We conclude that CALL should have a fixed cost, and that cost should not take into account contract memory retrieval. Each storage cell access should account for the cost of disk access.

## New Gas Prices of SSTORE, CREATE and CALL

To find the right prices for SSTORE, CREATE and CALL we must first analyze what is being prices by these opcodes, because processing these opcodes involves more than reading from disk.

### The cost of Storage

Since storage costs must be paid periodically, the initial cost of storage acquisition can be lowered. We’ll assume each entry in the trie consumes in memory 128 bytes (32 bytes key, 32 bytes data, and 64 bytes of overhead), and the same amount in disk, and we’ll try to compute what is the actual cost of the network storing this value, and the cost to retrieve each value. 

Some costs that will be used later:

<table>
  <tr>
    <td>Medium</td>
    <td>Cost</td>
    <td>1 TB cost /year 
(4 years amortization)</td>
    <td>Cost of 1 byte</td>
  </tr>
  <tr>
    <td>HDD</td>
    <td>Internal 
50 USD for 1 TB</td>
    <td>12.5 USD</td>
    <td>12.5 * 10^-12</td>
  </tr>
  <tr>
    <td>SDD</td>
    <td>Internal 
70 USD for 240 GB</td>
    <td>72 USD</td>
    <td>72 * 10^-12</td>
  </tr>
  <tr>
    <td>RAM</td>
    <td>50 USD for 8 GB</td>
    <td>1600 USD</td>
    <td>1600 * 10^-12 =
1.6 * 10^-9
</td>
  </tr>
</table>


We want to compare these cost to current RSK/Ethereum costs for storage.

The opcode SSTORE costs 20K gas (but 15K is refunded on data removal). So the net cost is 5K gas. Considering the average gas price as 20 gwei, and a 7 ETH/USD, then the cost for persisting data in contract storage is 7^10^-4. 

The following table compares what will cost to store the same in the different kinds of storage. The table takes into account that disks and computers have to be periodically replaced because they malfunction or become obsolete. The Moore's law prediction that worked well in the past says that computing power doubles every 18 months. Currently that grow has slowed down to a doubling every 30 months [https://en.wikipedia.org/wiki/Moore's_law]. 

A similar law exist for disk storage density (Kryder's law) that density increases 40% per year, but this prediction has fallen short in the last 5 years. The actual improvement is about 15% a year [[https://en.wikipedia.org/wiki/Mark_Kryder](https://en.wikipedia.org/wiki/Mark_Kryder)]. 

There is no precise measure of the average lifetime of a computer (and it varies depending on if it’s a laptop, a desktop PC or a tablet). But we can assume safely an average of 5 years. Therefore, following the current trend in HDD storage densities,  in 5 years new hard drives have doubled their capacity for the same price. This means that the price per byte has halved. If the prices keeps halving every 5 years forever, this means that the cost of storing a byte forever is just two times the cost of storing a byte for five years.

In case of SSD disks and RAM, the law that governs the increase in memory density is Moore’s law, so every 2.5 years (not 5) we see the memory size doubled. In the following table we very conservatively assume SSD and RAM price per byte decreases every 5 years.

<table>
  <tr>
    <td>Medium</td>
    <td>Cost [USD]</td>
    <td>Cost of 1 byte for 5 years</td>
    <td>Cost of 1 byte forever</td>
    <td>Ratio</td>
    <td>Ratio considering 1K replications</td>
  </tr>
  <tr>
    <td>ETH SSTORE</td>
    <td>7^10^-4</td>
    <td></td>
    <td></td>
    <td>1</td>
    <td></td>
  </tr>
  <tr>
    <td>HDD</td>
    <td>1.6 * 10 ^-9</td>
    <td>12.5 * 10^-12</td>
    <td>25* 10^-12</td>
    <td>218750</td>
    <td>218</td>
  </tr>
  <tr>
    <td>SDD</td>
    <td>9.2 * 10^-9</td>
    <td>72 * 10^-12</td>
    <td>144* 10^-12</td>
    <td>38043</td>
    <td>38</td>
  </tr>
  <tr>
    <td>RAM</td>
    <td>2.0* 10^-7</td>
    <td>1.6 * 10^-9 
</td>
    <td>3.2 * 10^-9</td>
    <td>1750</td>
    <td>1.75</td>
  </tr>
</table>


We cannot draw a definite conclusion from this numbers, but we can find several possible explanations. 

- The price for SSTORE is related to the storage in RAM

- Ethereum was made to have 10K full nodes, not 1K.

- There is a 10x safe margin.

- What is been priced is not storage. Two other resources are being considered by SSTORE price: the access time, and the cost for new nodes to download the state.

### The cost of disk access

Ethereum current block gas limit is 4M gas. The limitation of the gas has two uses in the short term: limit the block size, and limit the amount of computation the block requires. As executing an arithmetic loop in RSK VM takes 200 ns per instruction on average (5M instructions per second), and assuming all other opcode costs are set according to the real time required to execute them, we can roughly estimate that a block topping 4M gas requires 800 msec to be executed (since 4M/5M=0.8). Suppose that we want to keep an one second limit for block execution, the cost in gas of each storage operation must not exceed the cost of time it takes. The following table shows typical storage access times. 

<table>
  <tr>
    <td>Medium</td>
    <td></td>
    <td>Maximum number of operations per second</td>
    <td>Minimum cost in gas relative to the time it consumes</td>
    <td>Cost [USD]</td>
  </tr>
  <tr>
    <td>HDD</td>
    <td>10 ms</td>
    <td>100</td>
    <td>40K</td>
    <td>~2 cents</td>
  </tr>
  <tr>
    <td>SDD</td>
    <td>0.1 ms</td>
    <td>10K</td>
    <td>400</td>
    <td>0.02 cents</td>
  </tr>
  <tr>
    <td>RAM</td>
    <td>10 ns</td>
    <td>10^8</td>
    <td>~0</td>
    <td>~0</td>
  </tr>
</table>


Since Ethereum cost for SSTORE is 5K, we can infer that Ethereum design is tailored for SSD disks. The existence of RAM caches for data (and the fact that currently Ethereum state data can fit in RAM) means that miners can use computers with HDD without risk of being attacked until the state exceeds the RAM size or the process space. However, at the current pace of Ethereum state growth, this limit may be reached in a year. However, since attacking by spamming the state still requires the attacker to fill the state, and at current storage initial costs (20K for 128 bytes), the cost of filling 4 GB of RAM is 335K USD. We conclude that use of HDD disks for Ethereum miners would be highly discouraged in the forthcoming years.

Assuming this is true also for RSK, the above chart shows us that no contract storage operation should be priced less than 400 gas units. The new RSK gas price of 2K per SSTORE of non-zero data verifies this lower bound.

### The cost to future users

The blockchain is an ever growing data structure and nothing can change that. During the first years of Bitcoin, the experience of connecting to the network for the first time was smooth and the whole blockchain was downloaded in hours. Several improvements and optimizations had to be made to keep the initial download time short: for example, Bitcoin does not verify signatures before the last checkpoint embedded in the code. Ethereum opted for a more radical approach and new nodes can be bootstrapped in fast mode. In this mode they can receive a "signed" or trusted checkpoint of the last block in the blockchain, where they can find a hash digest of the current state. Then they can download the state from peers and they an verify the state against the hash digest. In the Ethereum node Parity, allows to download the full snapshot of the state from peer-to-peer filesharing networks, and import it. The import process took (as of August 2016) only 5 minutes. Therefore the only real limit to setup a full node is the download time for the state. [[https://github.com/ethcore/parity/releases/tag/v1.3.0](https://github.com/ethcore/parity/releases/tag/v1.3.0)]

Also the trie snapshot can be compressed, occupying much less space than in RAM (live objects) or in an indexed database in disk. The parity snapshot as of Ago-16 occupies 150 Mb. Downloading 150 Mb with a 5 Mbps link takes 50 minutes. [https://en.wikipedia.org/wiki/List_of_countries_by_Internet_connection_speeds]. The time a new user is willing to wait to get in sync is a subjective matter, and depends in many other factors. However, if state size increases at a lower rate than the rate the average bandwidth increases, then bootstrapping a node will always take the same time. The size of the state should be related to the number of users and the number of active services the users take part. Both of them can eventually reach a maximum. However, this maximum can be very high. In the third quarter of 2016, Paypal had more than 192 million active accounts [[https://www.statista.com/statistics/218493/paypals-total-active-registered-accounts-from-2010](https://www.statista.com/statistics/218493/paypals-total-active-registered-accounts-from-2010)]. I took 16 years for Paypal to reach this volume of user. So we could predict that there is a chance that a blockchain such as RSK can get 250M in ten years. 

We can assume that applications will be standardized so 250M users does not imply 250M copies of a wallet contract code, but a single copy called by all of them, and we focus our analysis on the contract storage.  We assume each user uses 8 applications (e.g. crypto-assets) and each use consumes 256 bytes, therefore each user consumes 2 Kbytes. If the blockchain targets 250M users, then the state would be as large as 500 GB. Transferring 500 GB today will take 115 days. [Nielsen's Law](https://en.wikipedia.org/wiki/Nielsen%27s_Law) says that the bandwidth available to users increases by 50% annually, so the average bandwidth in ten years will be 57 times higher than today. This means that downloading the state will still take 2 days.

It seems that, even in a very optimistic scenario of growth, the full nodes will be able to cope with the blockchain weight. However we cannot predict that miners will behave correctly to set the minimum price of each gas unit so that users do spam the state. In fact they have incentives to lower the minimum gas price both to get fees by  side-channels and to make the blockchain more popular by subsidizing its use. And they may be right in doing so!

Storage rent must be implemented not as a protection from spam, but to protect the blockchain users of the future from market driven measures that miners may take for a certain period of time to increase user adoption and to outperform the competition. Without storage rent, these market driven decisions turn into populism: sacrificing resources for short-term gains and preventing long term success. Storage rent also protects the blockchain against miscalculations and erroneous predictions on technology and blockchain adoption rate. However, implementing storage rent is complex, as many rent payments are micro-transactions, the system implemented must make sure the rent collection cost does introduce a new limiting factor to scaling.

  

### Can RSK launch without warning user to use full nodes with SSD?

A block should be processed in less than 1 second, as stated before. Assuming hard disk technology (not SSD), a disk access takes approximately 10 msec, which means that there can be only 400 disk operations in a block. Processing a simple account to account transaction transaction takes at least 2 disk accesses, so using a hard-drive sets an upper limit of 20 tps. Since we don’t expect RSK to reach 20 tps during the first year, this figure is ok to start with. 

Some experts estimate that SSD may completely replace HDDs in the sub 1GB space market in the coming 4 years. Also, since the state can be cached in RAM when it fits, we won’t need external storage during block execution during the first year.

Even without implementing storage rent, RSK could set a 10K initial SSTORE price, smoothly decreasing so that in 4 years it becomes close to ~400 as SSD drives outperform HDD.

### Are all operations prices for SSDs?

SSD's access time is around 0.1 msec, so using the same reasoning (4M gas limit / 10K operations = 400 ) the gas cost of SSTORE should be as low as 400 units, very close to the current value in Ethereum for a single disk access (such as what the BALANCE opcode requires).

A CALL operation costs 700 units of gas, which is in line with the cost of SLOAD. So it seems that Ethereum assumes state is stored in SSD. 

A standard SSD can store 512 Gbytes, allowing up to 2 billion simple accounts (256 bytes each), or 100 million multi-sig wallet contracts, so there seems to be enough space for growing globally.

Currently Ethereum holds approximately 600K active accounts (empty accounts are being pruned). 600K accounts also fit in RAM memory (about 150 Mbytes).

### Could the state fit entirely in RAM forever?

A standard machine RAM size is 8 GBytes. Assuming 6 Gigabytes are left to the OS and other applications, 2 GBytes can be used by the full node. That space can store 8M simple accounts, or 390K multi-sig wallets,  which seems low for global coverage. As a comparison, Bitcoin currently has 44M UTXOs (occupying more than 1.3 Gb compressed on disk).

## Conclusion

To be able to choose an adequate SSTORE initial and recurrent prices we must assume external conditions:

* During the first 4 years, computers having HDD will be allowed. Those computers will need to store the state in RAM. Therefore there can’t be more than 16M account in the first 4 years.

* Afterwards, all state will be stored in SSD, and no computer having HDD will be allowed.

As a conclusion, SSTORE recurrent price can be safely reduced 100 times (200 gas), while initial price only 10 times (2000 gas). 

Therefore rent cost (cost per byte) is set to 2 units of gas per year, and each storage cell is counted as 256 bytes. Therefore rentCost = 2/365/86400 (seconds a day).

Even if initial cost could be reduced further, is preferably be conservative, as there are many hidden costs related to write operations such as: 

* Most types of memories writing consumes more energy than reading. 

* Writing involves invalidating caches and merging caches, while it is not the case for reads.

* Writes may interfere with transaction serialization, while only reads do not.

[RSKIP08]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP08.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
---
rskip: 17
title: Simpler Persistent Storage Rent
description: 
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2016-09-27
---
# Simpler Persistent Storage Rent

|RSKIP          |17           |
| :------------ |:-------------|
|**Title**      |Simpler Persistent Storage Rent|
|**Created**    |27-SEP-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Rejected |


# **Abstract**

This RSKIP proposes that certain contracts should pay storage rent, to reduce the risk of storage spam and to make storage payments more fair. At the same time this RSKIP discusses the limitations of storage rent due to the additional complexity and overhead that, in some cases, overweight the benefits.

# **Motivation**

One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever. In principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.

A well designed crowd-contract should add a revenue method for paying for the memory rent of persistent storage. For example, each operation should be accompanied by a payment in bitcoins, and payments should be collected by the contract. However, such revenue collecting method must be defined at day 0, and at that stage it will be unclear if the revenue model can sustain the memory rent. The simplest case is that every user pays rent independently for the memory it consumes. For example, a DNS-like contract will attach an account balance to every name registered, and users would be free to send money to the registered name account.

When the time comes to pay the rent, the contract would see which names have enough balance, remove the names that cannot pay for the rent, subtract a fixed amount from each name balance, and pay the rent using LIFE_EXTEND-like opcode  (see RSKIP-08).

# DISCUSSION

Several problems arise:

1. A contract cannot schedule an action at a specific time, so triggering the rent-paying code would need to be done from a message coming from the outside world, before the rent deadline comes.

2. The payed amount would be specified in gas, so the gasPrice applies. A contract cannot easily determine the adequate gasPrice to be paid unless the price is semi-fixed. Even if the minimum gas price is  published by miners, miners are not forced to accept the rent-paying contract execution, not any other transaction. Therefore the the rent-paying contract execution should be scheduled using a crontab-like method. But one of the design choices of RSK/Ethereum is to avoid crontab-like scheduling because the CPU consumed by contract execution has a direct effect on block propagation, so the crontab system can be used to perform a DoS on certain miners (e.g. by using an expensive computation which the attacker knows the result, but not the honest miner)

3. Still one more possibility to go through this path is to limit contract execution scheduling to short executions (gas limited). Still another problem arises: how to prevent massive number of contracts requesting scheduling at the same time. One possibility is that rent-deadline events are chosen randomly at contract creation time, so the event is not previsible, and an attacker cannot create thousands of contracts with the same planned deadline).

4. Suppose that contracts must pay for memory rent once a month. An owner of a name in the DNS contract would prefer to pay an annual fee, rather than worrying about a monthly fee. If names must reserve bitcoins to pay for gas, then they must know the gasPrice one year ahead of time. This adds unnecessary uncertainty.

5. In case of short contracts and accounts, does the contract rent code introduces computation that is more expensive that the micro-rent that is required to be paid?

6. If the rent is a micro-transaction, the fees for a full transaction (21K gas) is an huge overhead, therefore it would be preferable to pay the rent re-using another transaction, saving the space and computation time of a single signature. One way to allow this is using a transaction with embedded contract code: in this way several rents can be paid. Another approach is by using multi-output transactions: a transaction that has a single input but different outputs. Still another option is that a transaction from sender x can pay the rent for address x using a single new field in the transaction.

There are several questions that must be answered:

1. When to rise the Not-enough-Rent  (NER) "exception".

2. Rent payment is direct or via a pre-deposit.

The first question admits several possibilities:

a. forced NER checking and processing at certain blocks heights (deadlines) for each contract

b. forced NER checking of random contracts on each block, the contracts to check being a subset pseudo-randomly derived from the previous block hash and current block timestamp.

c. unforced NER. Each account has a deadline and a pre-deposited bounties for hibernating the contract on NER.

d. forced NER on posponed deadline.

The option (a) may suffer from too many processing at certain blocks, even if the NER checking deadlines are somehow randomized. If deadlines are randomized, then there exists the problem of storing the deadlines in a data structure that allows easily retrieving the closest deadlines. A heap or priority queue may do this, but this structure must be reorganized after each deadline arrives and a new deadline is created. It would be better if one could use a cyclic data structure: one which allows the detection of each deadline without modification. Such data structure is a closed double-linked-list. Each node stores the contract creation block number and the contract address. A head pointer points to the next block number such in NODE(head).creationHeigh is the closest to the next block number modulo 777600 (corresponding to 90 days at 10/seconds blocks). When the linked list is scanned, if a node points to a contract address that does not exists, then the node in the list is automatically removed. All this list handling can be done in a native smart contract NERCHECKER, so the list is part of the contract memory. The NERCHECKER implementation works if a new consensus rule is created that anyone can hibernate a NES-raised contract: then the NERCHECKER only scans the list, issuing HIBERNATE calls for each NER contract, and removing from the list any inexistent contract. NERCHECKER also has to interact with contract creation: each time a contract is created somehow NERCHECKER must be executed to register the deadline.

One possibility is that NERCHECKER is special in the sense that the NERCHECKER memory is updated automatically on CREATE/CALL calls. Another option is that any contract recently created contract must call NERCHECKER register method in its initialization code, but this seems odd, since the consensus rules would need to enforce it anyway. Also CALLs can create accounts, but there is no code execution associated with account creation. Another complication is that transaction execution parallelization [RSKIP04] prevents several threads accessing writing to the same data structure in the NESCHECKER contract. One option is that contract creations are logged in a list per execution thread, and after all threads have finished, all list are concatenated and all contract addresses are added to the NERCHECKER memory. This could be done by an implicit or explicit transaction added by the miner that contains the list of contracts as data, but this involves a huge overhead of information duplication. 

The  best option is that all native contracts have memory that the consensus engine is free to modify, so the NESCHECKER storage memory is automatically updated on the finalization of the execution of the block.

Note that the new "collapsing" system for hibernation does not allow a contract to detect if another contract exists. This means if NESCHECKER detects that an account does not exists, it will remove it from the list. When the contract is awaken with the AWAKE opcode, 

it must be re-entered on the list.

The option (b) seems good, but the effectiveness depends on the ratio between accounts checked (C) per block and total number of accounts (T). If the total number of accounts is T=16M, and every block checks C=512 accounts, then the probability of a NER-account to be detected per block is 2^(T/C) = 2^-15 ~= 32K. Every day there are 8K blocks, so the probability of a contract to resist 3 days while being in NER state is (1-2^-15)^24000 ~= 0.48. So there is a 50% chance for an unscrupulous speculator to go undetected for 3 days.One possible variant is to make C dependant on T, such as C=Max(T/V,64).   It must be taken into account that checking C accounts has a cost of retrieving the account from disk. If it takes 5 msec to do it, then checking such block will take 2.5 seconds more, which seems too high.Therefore it seems better that each contract has a clear deadline. 

Regarding the question 2, direct rent paying is undesired because it puts the responsibility of the payment on a new user that may not have any stake in the contract.

This is a list of the ideas that were evaluated (and most of them discarded) in the process of selecting a solution.

The option "d" is interesting because it’s simpler. Every time a contract processes a message, the gas from the pre-deposit is consumed equivalent to the memory consumed in the last inactivity interval, and the deadline is postponed certain amount of time. However there are two problems with this approach: it allows memory gas arbitration (one can pre-deposit rent for 100 years), and the deadline data structure (which is central) must be updated every time a contract receive a message, adding a lot of work to transaction processing. 

## Other solutions with similar effects

In a following section we show that that space consumption assuming the Ethereum post-EIP150 fork cost for storage operations is generally not a problem. This is true for all possible assumptions for a medium to store contract memory, such as HDD, SSD and RAM. However it is true that future parties that become full nodes must pay the price to store the state in order to be able to verify transactions. It seems unfair that previous miners/full nodes collect fees for future miners. One way to achieve a similar (although not exact same) result is by setting apart a percentage of mining fees for miners in a distant future. For instance every fee collected could be split in 50% for this year miners, 25% for second year miners, 12.5% for third year  miners, etc. If there are N blocks per year, and p percentage is taken from the revenue por for miners in each block, then p must be such (1-p)^N=0.5. For blocks every 10 seconds, N=3153600, then (1-p)=0,9999997802. This however means that until the first year elapses, miners will receive less than 50% of what they would expect. Also this is not entirely fair, since future miners can mine only empty blocks to collect the storage subsidy, but do not actually store the state. A variant is that during the first year, the  percentage p is replaced by a higher percentage p’, and p’ decreases very block until it reaches p. For instance, p’ starts with p(0)=p’=1, and p(n)=p(n-1)*p.  

The difference between delaying fees and storage rent are the following:

- Delaying fees does not account for variations in memory pricings. Variation in pricing allows the network adapt to real costs. Because transaction fees are always tied to current fee price, and the platform does not allow users to acquire gas in advance, the future usability of the platform already depend on the future prices, even if contract storage did not. Therefore, making contract storage still dependent on future prices does bring a fundamental economic change.

- Storage rent can recover space from lost keys or spam attacks, where delaying transactions fees cannot.

## Rent payment period

There are two types of deadlines that could be used: storage size dependant deadlines or periodic deadlines, or an hybrid of the two. An example of the first is fixing an amount of gas use for memory (rentNext maximum) and forcing the deadline if a certain amount is surpassed. This means that the period depends on the memory consumed. This has the benefit that contracts that use very little memory are not forced to pay micro-fees.

For example, a contract using 3x32-byte records, would need to pay 900 gas units for a lifetime use, or 90 per year. But 90 gas units represents 0.000018225 USD, or 233 times less than the cost of the transaction that pays that micropayment. Therefore the overhead is overkill. For a rent payment per year, a platform storing 50M accounts would use 1 payment per second just to keep accounts active.

Therefore the storage rent system cannot be used for period micropayments. Periodic payments have the advantage that the data structure required to handle deadlines is a simple double-linked list, while data size dependant payments would require a more complex priority queue which must be updated for every transaction call. 

One solution is that account/contracts (AOCs) are marked as private or public. Private contracts pay the rents automatically from each transaction that interacts with it. Accounts would be always private. Private AOCs would have a periodic deadline checking code, once a year, but would not require the use of DEPOSIT_RENT. The opcode NEXTRENT would return the rent required to pay up to that transaction. 

Public AOCs would have the periodic deadline, but would use DEPOSIT_RENT system described.

A variant is to make accounts the only private AOCs, and contracts always public. This can reduce additional space used for accounts: one can set a single constant GasPerBlock, and consume GasPerBlock*(blockTime-lastUseTime) gas for each transaction. The only additional field required is lastUseTime. Note that the amount to be paid if there is a single transaction per day would be around 2 units of gas. Therefore the whole computation seems a high overhead for such micropayment. GasPerBlock being a power of two will slightly reduce the cost.

Still another possibility is to add a single bit per account yearRentPaid, set to false at start, and set a deadline every year to check which accounts have not paid a constant value per year, and clear all bits for the following year. But scanning all accounts (e.g. 30M accounts) may require loading the whole world state into RAM to prevent so many disk accesses. 30M accounts would require 8 GBytes of data to be loaded, so it’s again overkill.

One possibility is that accounts (or AOCs consuming less than 1 Kbyte of memory) will not pay storage rent. Therefore storage rent prevents an attacker from requesting a high amount of memory for a single contract, but does not prevent an attacker acquiring many small-sized contracts.

# Chosen solution

Contract persistent memory can be of any of three types:

- eternal, paying 10 years in pre-deposit for each cell.

- hibernable on unpaid-rent

- killed on unpaid-rent.

If it is hibernable, if rent is not enough, all the memory (contract+persistent mem) collapses into a single hash. To bring the contract alive again, a message paying the wakeup fee, containing all missing data must be sent.

A new opcode HIBERNATE is added for self-inflicted or hibernation of 3rd party contracts. In the first proposal of the HIBERNATE opcode, the opcode could accept an argument (flags) of whether to hibernate code, data or both, but this was discarded for simplicity data hibernation can always be achieved programmatically, by destroying data but keeping a hash in persistent memory. For simplicity it was decided that hibernation will always remove all contract code and memory. 

Hibernation freezes the contract until external wake up is performed. Self-hibernation can be used by contracts to sleep an amount of time, since no rent is paid during hibernation time.

Even if this RSKIP describes immortal contracts, this involves additional risks and it can be safely excluded.

# **Specification**

Every contract has three new fields:  **rentPreDeposit, rentDue** and **rentNext.**. All fields hold values in gas.

A contract can be of one of three types: immortal, mortal and ephemeral. Mortal and ephemeral contracts must pay a rent. 

The rent payment algorithm is as follow: 

-Let t be the current date.

-Let d be the date of the last message sent to a contract.

-Let s be the amount of memory consumed by persistent memory

Then when a message at time t is arrives to a contract X and needs to be processed ,  then contract X must pay rent for r = s*rentCost/(t-d) gas. The amount of gas is added to rentNext., When the deadline is detected by NESCHECK, the following actions happen:

- the rentDue value is compared against rentPreDeposit: if there is not enough funds in the pre-depost, the contract is hibernated (and all rent fields are cleared)

- the rentPreDeposit field is cleared.

- the rentNext value is copied to rentDue.

- the rentNext field is cleared.

On each deadline, the pre-deposit MUST be emptied, even if it contains more than the rentDue amount. To reduce the uncertainty of the amount of gas that should be pre-deposited, the rentDue is "locked" on the previous deadline.

It is suggested that the deadline interval is 3 months, so the user has at most 6 months to pay from the moment of the memory usage, and 3 months from the moment he knows exactly how much he has to pay. 

At any time the contract can make a pre deposit in gas using the opcode DEPOSIT_RENT <gasAmount>.

A problem is that if blocks are not saturated of gas use, then miners may use the gas left to pay rents for contracts (and offer this a service). The fact that RSK has a minimal gasPrice and we are planning to forward 90% of the fees collected in a block to the next blocks partially prevents this, since the miner will be having a 10% discount on the minimum price, but not zero cost.

When r >  rentPreDeposit the contract has become a debtor. When a message is to a contract, it is processed, after the processing, if the the debtor contract is a debtor, it is killed or hibernated.

Generally the contract will include a method such that:

public PayMyRent(int amount)

{

 DEPOSIT_RENT(this,amount)

}

Mortal and Ephemeral contracts have different rent costs. Ephemeral contracts pay a little less, but if the rent is consumed, the contract is destroyed (and the contract bitcoins vanish). 

The cost of hibernation does not depend on the size of the memory, since this RSKIP will be implemented on top of the new Trie structure (persistent memory below account address on the Trie). Therefore the root hash of the memory subtree need not be computed.

When the contract is hibernated, both the code, memory and balance are hashed and only a single hash digest is stored in the contract address. While the contract is hibernated it cannot receive nor send payments or messages except a special WAKEUP message. The user can awake the contract by sending the WAKEUP message containing the full code and persistent data. If the code and data does not fit into a message then the user needs to create a proxy contract that composes the code/data in chunks, append the chunks, and sends the contract a wakeup WAKEUP message using the WAKEUP opcode. The WAKEUP opcode has several arguments arguments: the code, the code size, the data, the data size, the contract address and the initial pre-deposit for the rent and pre-deposit for the bounty. WAKEUP returns an error code if the contract could not be woken up: the code 2 means that the code was invalid, 3 means that the data was invalid, and 4 means that the rent is too low for paying the re-hibernation cost.

**Immortal contracts**

Contracts can also become immortal by calling IMMORTALIZE and paying 10 years*rentCost*s where s is the current memory consumed. The bitcoin bounty is paid back and moved to the contract normal balance. Immortal contracts may offer long term storage service to other contracts, and how this affects the market should be analyzed. Memory requested by immortal contracts pay a special immortalizeCost gas cost per SSTORE, but do not require a bitcoin bounty.

## Security

This design only is secure if:

1. There is always a reasonable minGasPrice. No contract can pay lower than this price. This is to protect a miner from buying eternal memory at no cost.

2. Some fes percentage should be burned to prevent miners forming a coalition to buy eternal memory at low cost (however they could collude to put the minGasPrice low enough, so buring does not apply).

To prevent services that offer memory to avoid rent, the cost of memory should not be higher than the cost of transfer it from one contract to another. 

## New opcodes

Let m be the amount of memory persisted by the contract in 32 byte words.

### HIBERNATE

Arguments: <address> 

GasCost: provided by the caller (not taken from the hibernation deposit) 

### WAKEUP

Arguments: contract_address code_address code_size trie_address trie_size

Value: equal or higher to m*f+c

Value accepted: 6 month of rent.

Returns: error_code

### SET_MORTALITY

Arguments: <mortaity_kind>

GasCost: 

if switching to immortal from mortal/ephemeral: m*immortalizeCost

	if switching to ephemeral/mortal from immortal: -m*immortalizeCost/2 (refund in gas)

Possible mortality_kind values:

0. Ephemeral

1. Mortal

2. Immortal

Changes the contract mortality type 

### DEPOSIT_RENT 

Arguments: <address>

Value: amount to deposit. It must be equal or higher than m*f

Value accepted: m*f

If the rent overflows the maximum accepted rent (6 months), the remaining is not deposited.

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

### SET_FLAGS

 new-bit ForbidExternalRentPayments (default 0 = false)

Due to the number of opcodes required, if may be preferable to implement them by one or several native contracts. In this case, it must be evaluated if the cost of call exceeds or not the desired cost of the operation.

## A discarded variant

One possibility is that rent is automatically deposited when new persistent memory cell is requested with the SSTORE opcode. The the cost d to execute the SSTORE opcode is split in the following way:

- x = (a+b) 

- "a" is automatically pre-deposited for rent, 

- "b" is payed to the miner (persistCost) 

This has the effect that the user requesting the memory pays not only for the grace period of 3-6 months, but for an additional period of 3 months. Since the effect is subtle, and it can always be emulated by execution a DEPOSIT_RENT opcode along with SSTORE, I don’t  think it add much value.

Unowned Contracts 

A contracts that does not have an owner must make sure the users that interact with the contract pay the rent. One way to achieve this is by tracking the time each user has made a partial rent payment and detect when too much time has elapsed since the last payment. Contracts are able to known how much gas is in the deposit with the opcode GASDEPOSITED, and also they can know how much memory is being held with the SMEMSIZE opcode.

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

Should the cost of a CALL account for the cost of restoring all persistent contract data into RAM? To answer this question it is required to analyze how contracts use storage and how the system would cache contract storage. On one side the cost of CALL is currently fixed, so it cannot account restoring all contract storage from disk into RAM. On the other side, paying for a disk access for every 32 bytes readed from memory is overkill. Using an unified tree for accounts and contract storage means than storage elements get stored on random places on disk, so there is no way all elements can be retrieved at the same time at lower cost. However, a storage cache is present, so after a first access subsequent accesses in the same transaction are executed much faster. Writing or reading  to a storage cell a hundred times costs the same as accessing it a single time. If access cost in gas is constant, then contracts should use volatile memory as a cache to perform several operations until a single  SSTORE operation is made last for every cell accessed. Changing the cost of SSTORE/SLOAD depending on if it is the first time or not seems overly complex and prevents static code analyzers from easily inferring gas cost of a code block. 

We conclude that CALL should have a fixed cost, and that cost should not take into account contract memory retrieval. Each storage cell access should account for the cost of disk access.

## New Gas Prices of SSTORE, CREATE and CALL

Since storage costs must be paid periodically, the initial cost of storage acquisition can be lowered. We’ll assume each entry in the trie consumes in memory 128 bytes (32 bytes key, 32 bytes data, and 64 bytes of overhead), and the same amount in disk, and we’ll try to compute what is the actual cost of the network storing this value, and the cost to retrieve each value. 

Some costs that will be used later:

<table>
  <tr>
    <td>Medium</td>
    <td>1 TB cost /year (4 years amortization)</td>
  </tr>
  <tr>
    <td>HDD</td>
    <td>24 USD</td>
  </tr>
  <tr>
    <td>SDD</td>
    <td>55 USD</td>
  </tr>
  <tr>
    <td>RAM</td>
    <td>1600 USD
(50 USD per 8 Gbytes)</td>
  </tr>
</table>


Assumption:

- Accounts and contract storage is stored in HDD disk.

The storage of the entry in RAM is temporarily stored in a cache, and this lasts only during the execution of the contract, so as long as the RAM is big enough to temporarily store all cache records, there is no problem. Assuming a block gas limit of 8M, and a SLOAD cost of 200 units, a contract can load 40K entries, occupying 5 Mbytes. This cost is therefore unimportant. As of 2016 the cost of storing 1 TB on an external hard drive can be as low as USD 24. A hard-drive has a lifetime of 4 years, therefore the cost of storing 1 TB for 3 months is 1.5 USD.  1 TB can store 2^(40-7)=2^33 entries. With the current ETH/USD price of 9 USD, the cost in Ethereum to store 2^13 entries is 0.497664 USD (0.0000000225*300*9*2^13). The 2^13~=16K entries occupy 2^20 bytes on disk (1 Mbyte). We’ll assume the Ethereum blockchain will last 50 years, and afterwards the state will be manually cleaned. Then Ethereum real price of hard disk storage of 16K entries is 24*50/4/1000/1000=0.0003 USD. But that is the storage cost in a single computer, while the blockchain is replicated hundreds if not thousands of times. Let’s assume there are 1000 nodes, and that is enough to support a decentralized network. Therefore the cost of storing 2^13 entries is 0.30 USD. Therefore the current cost of SSTORE is approximately equal to the actual cost. But the actual cost computed does not take into account that hard disk storage costs will decrease every year. Assuming the storage costs will half every 4 years, then the cost is  24*4*2/1000=0.192, or half of the cost in Ethereum. Therefore assuming HDD storage RSK could reduce the cost by 12.5 (payment every 3 months). Let’s now focus on disk access cost.

A block should be processed in less than 2 seconds to prevent excessive empty blocks being generated. Assuming hard disk technology (not SSD), a disk access takes approximately 10 msec, which means that there can be only 400 disk operations in a block, or a maximum of 40 tps. Since we don’t expect RSK to reach 40 tps fast, this is ok to start with. However,  with a 4M gas limit, each access should cost 10K gas units.

Some people estimate that SSD may completely replace HDDs in the sub 1GB space market in the coming 4 years. Without storage rent, RSK could set a 10K price at the beginning for 4 years, and then decrease it to ~300 as SSD drives reach HDD drives parity.

Assumption: 

- State is stored on SSD

SSD's access time is around 0.1 msec, so the gas cost of SSTORE should be 200 units, very close to the current value in Ethereum.

A CALL operation costs 700 units of gas, which is in line with the cost of SLOAD/SSTORE.

So it seems that Ethereum assumes state is stored in SSD. The cost of SSD per gigabyte approximately doubles the cost of HDD, therefore, as previously discovered, the cost of storage is close to the gas consumed..

A standard SSD stores 512 Gbytes, allowing up to 2 billion accounts (256 bytes each), so there seems to be enough space for growing.

If accounts are stored in SSD, it means that in 2 seconds of block processing there can be 2000 account accesses, which is more than what is needed. Currently Ethereum holds approximately 600K active accounts (empty accounts are being pruned). 600K accounts also fit in RAM memory (about 150 Mbytes).

Assumption:

- State is stored in RAM

A standard machine RAM size is 8 GBytes. Assuming 4 Gigabytes are left to the OS and other applications, 4 GBytes can be used by the full node. That space can store 32M accounts, which seems low for global coverage. As a comparison, Bitcoin currently has 44M UTXOs (occupying more than 1.3 Gb compressed on disk).

The cost of permanently storing the 2^13 entries (1 Mbyte) a year is 1600/1000/1000=0.0016 USD, multiplied by 1000 nodes, it’s 1.16 USD. This means the cost of storing it in RAM is two times the cost of 2^13 SSTORE executions in Ethereum (ETH opcodes are a little underpriced)

Therefore, while all state can be stored in RAM, the system is highly performant, allowing more than 10K tps.

[RSKIP04]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP04.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
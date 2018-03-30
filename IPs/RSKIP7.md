
#  Persistent Storage Rent Paid by Code

Code: RSKIP7

Author: SDL

Status: Rejected

# Abstract

This RSKIP describes an implementation of storage rent based on enabling the code to periodically perform an operation to deposit the rent before the due time.
Motivation
One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever.
In principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.
A well designed crowd-contract should add a revenue method for paying for the memory rent of persistent storage. For example, each operation should be accompanied by a payment in bitcoins, and payments should be collected by the contract. However, such revenue collecting method must be defined at day 0, and at that stage it will be unclear if the revenue model can sustain the memory rent. The simplest case is that every user pays rent independently for the memory it consumes. For example, a DNS-like contract will attach an account balance to every name registered, and users would be free to send money to the registered name account.
When the time comes to pay the rent, the contract would see which names have enough balance, remove the names that cannot pay for the rent, subtract a fixed amount from each name balance, and pay the rent using LIFE_EXTEND-like opcode  (see RSKIP-08).

# Discussion

Several problems arise:

1. A contract cannot schedule an action at a specific time, so triggering the rent-paying code would need to be done from a message coming from the outside world, before the rent deadline comes.

2. The paid amount would be specified in gas, so the gasPrice applies. A contract cannot easily determine the adequate gasPrice to be paid unless the price is semi-fixed. Even if the minimum gas price is published by miners, miners are not forced to accept the rent-paying contract execution, nor any other transaction. Therefore the the rent-paying contract execution should be scheduled using a crontab-like method. But one of the design choices of RSK/Ethereum is to avoid crontab-like scheduling because the CPU cycles consumed by contract execution has a direct effect on block propagation, so the crontab system can be used to perform a DoS on certain miners (e.g. by using an expensive computation which the attacker knows the result, but not the honest miner)

3. Still one more possibility to go through this path is to limit contract execution scheduling to short executions (gas limited). But another problem arises: how to prevent massive number of contracts requesting scheduling at the same time. One possibility is that rent-deadline events are chosen randomly at contract creation time, so the event is not predictable, and an attacker cannot create thousands of contracts with the same planned deadline).

4. Suppose that contracts must pay for memory rent once a month. An owner of a name in the DNS contract would prefer to pay an annual fee, rather than worrying about a monthly fee. If names must reserve bitcoins to pay for gas, then they must know the gasPrice one year ahead of time. This adds unnecessary uncertainty.

There are several questions that must be answered:

1. When to rise the Not-enough-Rent  (NER) “exception”.
2. Rent payment is direct or via a pre-deposit.

The first question admits several possibilities:
a. forced NER checking and processing at certain blocks.
b. deadlines for pre-deposited bounties for hibernating the contract to become active.

The option (a) may suffer from too many processing at certain blocks, even if the NER checking deadlines are somehow randomized.

Regarding the question 2, direct rent paying is undesired because it puts the responsibility of the payment on a new user that may not have any stake in the contract.
This is a list of the ideas that were evaluated (and most of them discarded) in the process of selecting a solution.

S1) Periodic payments in BTC

Every time contract is created, a random rent-paying deadline based on blockhash is assigned in the future, preferably not before 12 months and not after 18 months. 1 month before the deadline, a programmed event is scheduled and its execution is forced. The event is executed with a fixed maximum gas. Also the gasPrice is fixed for the event one month before the event. (still another) problem with this approach is that once the deadline is known, there is an incentive to register a name just after the deadline (to prevent an early pay).  

S2) Not-Enough-Rent (NER) checked when messages are processed

Rent is only to be paid when a message is received. Every time a name balance is increased, the deadline checking code is executed, if the deadline is too close (less than 6 months) the rent is paid in advance. This brings uncertainty (what if nobody remembers to pay in the last 6 months?).

S2B) Rent paid by users when they send a message to a contract

Users directly pay rents. If some time passes and the amount of gas specified by the message is not enough to pay the memory rent, then the contract is hibernated.
This has the drawback that if a contract stores a high amount of data, then the rent may quickly become too high for normal users who are willing to pay. For example, no new users will want to interact with the contract because they will be paying rent for memory they never used.

S3) Persistent Memory Cells are a Ledger

Every persistent cell has its own associated monetary balance and pubkey. The rent paying is done not by executing contract code, but forced by the protocol: every cell is scanned and cell with no balance are removed automatically. The contract must use centinel cells (with no balance) to detect a garbage-collection has occurred, and re-scan its memory to rebuild the necessary data indexes to continue working.

S4) Distributed Memory for rent democratization

S3 lines up with the idea that a contract could use distributed account memory instead of centralized memory (RSKIP01). In this case, the account would periodically pay for its own memory. If not paid, it will be garbage collected. This does not give a solution on how to pay a rent for centralized memory: one possibility is that distributed memory pays a share of centralized memory. For example, if a contract has 10 Kbytes of centralized memory, and 100 Kbytes of decentralized memory, belonging to 100 different users, then each user pays the rent for 10.1 Kbytes of memory. This brings a new problem: what if I want to pay rent for some piece of data that I own, but not for some other. For example, I have 10 PlutoShares and 100 TetherUSD. Since PlutoShares are now worth zero, I don't want to pay rent for that space. A solution is to use different accounts to store different assets (this in turn requires maintaining different private keys for each). I could command my account to remove a certain section of my account memory before paying the rent. Another problem is that if any contract can use my account memory without my authorization, then why should I pay for that? The solution is that only the contracts that I enable should be able to. Therefore the platform needs a special command ENABLE_STORAGE &lt;contract&gt; / DISABLE_STORAGE &lt;contract&gt; to enable account memory use for a specified contract (or alternatively, the VM needs two more opcodes to do the same). Another problem is that if the rent is paid on a specific date, then the assets are worth less just before the pay-day, and more just afterwards: that's ugly. One solution is that doing anything with the contract that holds the distributed chunk of memory must pay the rent for the time it was unused, so transferring a TetherUSD would yield more gas fees if the TetherUSD were not used for a long time.
Let's take for example a DAO. If the shares are bearer-instruments, and we allow them to be transferred in peer to peer mode (without using the centralized memory), then that means that the centralized memory does not know who has the shares. Therefore it cannot pay dividends: users should call the issuer contract to collect dividends. Shares should be represented by tokens having a dividend-paid label. For example, a share would be a tuple (d,a), where d is the amount of dividends cycles it has received, and a is the amount. To split a share in two, the computation is (d,a) = (d,a1) + (d, a2) where a1+a2 = a. Two shares with different dividend cycles cannot be added: the only will less "d" value should have its dividends collected until it matches the "d" value of the other share term to add. The p2p memory system also means that transfer of tokens cannot be specially taxed by the main contract. Of course, if a share vanishes because the owner has not paid the memory rent, then the main contract has no way of knowing this unless shares are periodically registered with the main contract.

S5) External contracts pay for cells

Another bad option: we tag every value in the persistent memory as EXTERNAL, which means that when the external contract pays rent, this cell is also paid. the value corresponds to the address of another contract. When this external contract dies, the pair is automatically erased. Internally this requires a contract to have a list of persistent memory cells pointing.

S6) Child Contracts

S1 + a twist seems to be the best solution. S1 has a drawback: a fully distributed contract REGISTRY that stores information that belongs to other users (such as a DNS) must pay rent for all its users in a centralized way. To solve this problem, we will encourage the use of master-child contracts: every user must have a child contract that stores the data required by the registry and the user that owns the record is responsible for maintaining its rent. However, even a single (a,b) pair is required to be stored in REGISTRY to locate the record.
One option: every child contract rent contributes to parent contract rent (accumulates gas). When a child contract C is created, the parent sets the externalMem property. To keep alive the contract C the user must pay for size(C)+externalMem bytes. Also parents should be able to write to the persistent memory of children using two new opcodes: CSTORE and CLOAD.

All ideas regarding memory rent goes against one of the main goals of smart contracts that is immutability, which is to be a pillar of no third party trust. Child contracts allow different unrelated parties to collaborate to maintain the rent of a single master contract.

If a smart contract solution is organized in several related contracts, the child contract idea does not allow the family of contracts to benefit from child rent. One possible solution is that rent is not automatically paid, but the child contract has a method PayRent that calls the PayRent method of the parent, who then re-distributes the rent to the remaining members of the family. Since the child contract code is chosen by the parent, he can create whatever rent distribution algorithm he desires. If this scheme is implemented, then there can't be a DEPOSIT_RENT opcode that receives a contract address argument: all payments must be done by the contract code itself. Or better, there can be a contract flag that prevents external payments of contract rent. To avoid duplication of child contract code, we should implement opcodes of easy proxy calls.

To allow child contracts to be easily removed when parent contract dies, the child contract address could be built with the 20-byte parent contract address, plus 1 zero pad byte plus 4 bytes (DWORD) of child-addresses. Child addresses would be just the nonce of the parent contract on creation.


I don't see why child contracts should have code if the parent contract can access the child contract's persistent memory. A variant is that child contracts do have code, and this code provides getters and setters to child persistent storage. This option is more “clean” in the sense it does not require two new opcodes. But also makes child contracts more expensive, when actually child contracts are used as lightweight isolated storage. Anyway, it's possible to let the programmer decide which method he will use, and allow both CLOAD/CSTORE and child contract code.

### CSTORE

Arguments: &lt;child-contract-address&gt; &lt;memory-address&gt; &lt;value&gt;

### CLOAD

Arguments: &lt;child-contract-address&gt; &lt;memory-address&gt;

Returns: &lt;value&gt;

### CREATE_CHILD (deprecated version)

Arguments:	&lt;new-address&gt; &lt;externalRent&gt; &lt;in_size&gt; &lt;in_offs&gt; &lt;gas_val&gt;

Creates a child contract. The child contract address is Hash(Parent-address || new-child)

### CREATE_CHILD (new version)

Arguments:	&lt;in_size&gt; &lt;in_offs&gt; &lt;gas_val&gt; (same as create)

Creates a child contract. The child contract address is Parent-address || 0 || parent-nonce

SET_FLAGS : new-bit ForbidExternalRentPayments (default 0 = false)

# Chosen solution

Each persistent memory cell has an additional bit of information “survive”. When a cell is marked to survive (survive=true) then when the next NER  deadline arrives, the system tries to keep that cell for the next rent period. If the cell is not marked (survive=false) then when NER checking occurs, the cell is first cleared (no gas consumed for this)  and then the NER checking occurs.

Contract persistent memory can be of any of three types:
- immutable, paying 10 years in pre-deposit for each cell.
- hibernable on unpaid-rent
- killed on unpaid-rent.

If it is hibernable, if rent is not enough, all the memory (contract+persistent mem) collapses into a single hash. To bring the contract alive again, a message paying the wakeup fee, containing all missing data must be sent.
A new opcode HIBERNATE is added for self-inflicted or hibernation of 3rd parity contracts. This opcode could accept an argument (flags) of whether to hibernate code, data or both, but this was discarded for simplicity Data hibernation can always be achieved programmatically, by destroying data but keeping a hash in persistent memory. Internal hibernation freezes the contract until external wake up is performed. For simplicity, hibernation will always remove all contract code and memory. Self-hibernation can be used by contracts to sleep an amount of time, since no rent is paid during hibernation time.

# Specification

Every contract has two new fields:  rentPreDeposit and shrinkKillOrHibernationBounty (SKHBounty for short). rentPreDeposit field hold a value in gas, while SKHBounty values in bitcoins.

A Contract can be of one of three types: immortal, mortal and ephemeral. Mortal and ephemeral contracts must pay a rent.

The rent payment algorithm is as follow:
-Let t be the current date.
-Let d be the date of the last message sent to a contract.
-Let s be the amount of memory consumed by persistent memory
-Let z be the amount of memory consumed by cells marked with the  “survive” flag.

Then when a message at time t is to be processed,  the contract must pay rent for r = s*rentCost/(t-d) gas. The amount of gas is subtracted from rentPreDeposit and it is paid to all the miners using a smooth function.

The maximum amount of gas that can be stored in rentPreDeposit is s*rentCost/(6 month). This means that a contract cannot invest in gas for a longer period than 6 months.

At any time the contract can make a pre deposit in gas using the opcode DEPOSIT_RENT <gasAmount>.

The rent is automatically deposited when new persistent memory cell is requested with the SSTORE opcode. The cost d to execute the SSTORE opcode is split in the following way:
- x = (a+b+c)
- “a” is automatically pre-deposited for rent,
- “b” is payed to the miner (persistCost)
-“c” is also pre-deposited for contract removal bounty, but in bitcoin at the minimum gas price designated by miners.
- c = b

Because requesting new cells of persistent memory add to the bounty accumulator, it may be the case that the bounty becomes too high that miners may have an incentive to prevent sending messages to the contract so the contract becomes a debtor and the bounty can be collected. This must be solved. To prevent this, one can force the bounty to be spent on rent, but that requires using a reference gasPrice, that we don't have..

Another problem is that if the bounty becomes too high (because the gasPrice was high and was then lowered) a contract owner can hibernate a contract to collect the bounty and then wake it up, providing a lower bitcoin bounty.

It seems there is no escape to gas/btc arbitration.

Another problem is that if blocks are not saturated of gas use, then miners may use the gas left to pay rents for contracts (and offer this a service). The fact is that RSK has a minimal gasPrice and we are planning to forward 50% of the fees collected in a block to the next blocks partially prevents this since the miner will be having a 50% discount on the minimum price, but not zero cost.


When r > rentPreDeposit the contract has become a debtor. When a contract is a debtor, anyone can attempt to kill, shrink or hibernate the contract and collect a contract removal bounty or the shrink bounty by executing a HIBERNATE, SHRINK or KILL opcode. When a HIBERNATE/SHRINK/KILL opcode is executed, the sender can specify an address to receive back a bounty. If hibernated, the preDepositRent and bounty will become zero, and all the bounty will be paid. If shrunk, the bounty will be paid in the proportion of removed cells over the total number of cells (removed cells are the cells not marked to survive)

Generally the contract will include a method such that:

```
public PayMyRent(int amount)
{
 DEPOSIT_RENT(this,amount)
}
```

Mortal and Ephemeral contracts have different rent costs. Ephemeral contracts pay a little less, but if rent is consumed, the contract is destroyed (and the contract bitcoins vanish).
The cost of hibernation does not depend on the size of the memory, since this RSKIP will be implemented on top of the new Trie structure (persistent memory below account address on the Trie). Therefore the root hash of the memory subtree need not be computed.

When the contract is hibernated, both the code, memory and balance are hashed and only a single hash digest is stored in the contract address. While the contract is hibernated it cannot receive nor send payments or messages except a special WAKEUP message. The user can awake the contract by sending the WAKEUP message containing the full code and persistent data. If the code and data does not fit into a message then the user needs to create a proxy contract that composes the code/data in chunks, append the chunks, and sends the contract a wakeup WAKEUP message using the WAKEUP opcode. The WAKEUP opcode has several arguments arguments: the code, the code size, the data, the data size, the contract address and the initial pre-deposit for the rent and pre-deposit for the bounty. WAKEUP returns an error code if the contract could not be woken up: the code 2 means that the code was invalid, 3 means that the data was invalid, and 4 means that the rent is too low for paying the re-hibernation cost.

# Immortal contracts

Contracts can also become immortal by calling IMMORTALIZE and paying 10 years*rentCost*s where s is the current memory consumed. The bitcoin bounty is paid back and moved to the contract normal balance. Immortal contracts may offer long term storage service to other contracts, and how this affects the market should be analyzed. Memory requested by immortal contracts pay a special immortalizeCost gas cost per SSTORE, but do not require a bitcoin bounty.

# Security

This design only is secure if:

1. There is always a reasonable minGasPrice. No contract can pay lower than this price. This is to protect a miner from buying eternal memory at no cost.

2. Some fees percentage should be burned to prevent miners forming a coalition to buy eternal memory at low cost (however they could collude to put the minGasPrice low enough, so burning does not apply).

To prevent services that offer memory to avoid rent, the cost of memory should not be higher than the cost of transferring it from one contract to another.
New opcodes
Let m be the amount of memory persisted by the contract in 32 byte words.

### HIBERNATE

Arguments: &lt;address&gt; &lt;bounty-pay-address&gt;

GasCost: provided by the caller (not taken from the hibernation deposit)

### KILL
Arguments: &lt;address&gt; &lt;bounty-pay-address&gt;

GasCost: provided by the caller (not taken from the hibernation deposit)

### SHRINK
Arguments: &lt;address&gt; &lt;bounty-pay-address&gt;

GasCost: provided by the caller (not taken from the hibernation deposit)

### WAKEUP

Arguments: &lt;contract_address&gt; &lt;code_address&gt; &lt;code_size&gt; &lt;trie_address&gt; &lt;trie_size&gt;

Value: equal or higher to m*f+c
Value accepted: 6 month of rent.
Returns: error_code

### SET_MORTALITY

Arguments: &lt;mortaity_kind&gt;

GasCost:
if switching to immortal from mortal/ephemeral: m*immortalizeCost
	if switching to ephemeral/mortal from immortal: -m*immortalizeCost/2 (refund in gas)

Possible mortality_kind values:
0. Ephemeral
1. Mortal
2. Immortal

Changes the contract mortality type

### DEPOSIT_RENT

Arguments: &lt;address&gt;

Value: amount to deposit. It must be equal or higher than m*f
Value accepted: m*f

If the rent overflows the maximum accepted rent (6 months), the remaining is not deposited.

### SET_FLAGS

new-bit ForbidExternalRentPayments (default 0 = false)

# Performance

If the survive flag is implemented as a bit attached to a cell, the shrink opcode would need to iterate over all memory cells and detect which are marked to survive. To make this process constant time, the memory will be split in two subspaces as two account subtrees in the trie: the survive subspace and the perish subspace. The bit 255 of the keys will be used to decide which subtree the key belongs. If the contract code wishes to mark a cell to survive flag, the cell must be moved from the perish subspace to the survive subspace by setting the bit 255 of the key, and vive-versa. This means that the actual hash security of the trie is 1 bit less. When looking up a cell, the contract must first search the surviving subtree and if not found it must search it in the other subtree. This must be done by masking the bit 255 and setting the bit 255.The contract must make sure a key will never be present in both subtrees. Removing a key requires trying removing the key in the first subtree, and if not found re-trying in the opposite subtree. The contract designer must decide to use or not the two available subspaces, and can ignore them. In that case the author can decide to place all keys in the survive or perish subspaces.

# Sample Use Cases

### User-asset contract

A contract ASSET maintains a local ledger in (a,v) pairs, such that “a“ is the address of the asset owner and “v“ is the asset value owned. When a receives an asset for the first time, a transfer() method is called, and a new memory cell is created in the survive subspace. The user will periodically (once every 6 months) call a method payRent() of the ASSET contract. The method will check if the address is in the survive subspace, and if not, it will move the address there, and pay a deposit corresponding to the cost of 6 months of storage.

```
 public payRent()
{
 address a = msg.sender;
 address bit255 = (1<<255);
 a &=~bit255; // perish subspace
 if (accountTrie[a]!=0) {
     balance = accountTrie[a];
     accountTrie[a] = 0;
     a |= bit255;
    accountTrie[a] = balance
    Amount amount = memoryCost;
    DEPOSIT_RENT(this,amount)
 }
}
```

## User-asset contract using child contracts

This is a case where child contracts are used to increase the parallelization factor. The ASSET contract will not store anything in its own persistent memory, but still needs to pay the cost of maintaining the ASSET contract alive.

Each child contract will be created with the address HASH ( parent-address |  user-address ). Each child contract could have the following method:

```
public payRent()
{
   if (gas<rentGas) thow;
   DEPOSIT_RENT(this,rentGas*5) // pays for code + data
   DEPOSIT_RENT(parent,rentGas) // overpays for some constant code/data
}
```

Another option is that the parent contract implements the following method :

```
public payRent() {
  address a = msg.sender;
  address childContract = SHA3( this.address , a);
  DEPOSIT_RENT(childContract,rentGas);
  DEPOSIT_RENT(this,rentGas);
}
```

## Long term effects

We assume a contract is not hibernated, and the normal workings are the following:

-  Each minute a new record is created
- 75% of the record owners pay the rent every 6 months.
- gasPrice is stable

Just after 6 months, the contracts has 260K records. The total amount of gas paid is 260K &#42;x. Let's assume x=a+b+b, and  a=b=c
The deposit is 260K &#42;x/3. The bounty is 260K/3.

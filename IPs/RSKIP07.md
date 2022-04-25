---
rskip: 7
title: Persistent Storage Rent Paid by Code
description: This RSKIP describes an implementation of storage rent based on enabling the code to periodically perform an operation to deposit the rent before the due time.
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2016-06-11
---

# Persistent Storage Rent Paid by Code

|RSKIP          |07           |
| :------------ |:-------------|
|**Title**      |Persistent Storage Rent Paid by Code |
|**Created**    |11-JUN-16 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Rejected |

# **Abstract**

This RSKIP describes an implementation of storage rent based on enabling the code to periodically perform an operation to deposit the rent before the due time.
Motivation
One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever.
In principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.
A well designed crowd-contract should add a revenue method for paying for the memory rent of persistent storage. For example, each operation should be accompanied by a payment in bitcoins, and payments should be collected by the contract. However, such revenue collecting method must be defined at day 0, and at that stage it will be unclear if the revenue model can sustain the memory rent. The simplest case is that every user pays rent independently for the memory it consumes. For example, a DNS-like contract will attach an account balance to every name registered, and users would be free to send money to the registered name account.
When the time comes to pay the rent, the contract would see which names have enough balance, remove the names that cannot pay for the rent, subtract a fixed amount from each name balance, and pay the rent using LIFE_EXTEND-like opcode  (see [RSKIP08]).

# **Specification**

See discussion [here](https://github.com/rsksmart/RSKIPs/issues/78)

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

## Immortal contracts

Contracts can also become immortal by calling IMMORTALIZE and paying 10 years*rentCost*s where s is the current memory consumed. The bitcoin bounty is paid back and moved to the contract normal balance. Immortal contracts may offer long term storage service to other contracts, and how this affects the market should be analyzed. Memory requested by immortal contracts pay a special immortalizeCost gas cost per SSTORE, but do not require a bitcoin bounty.

## Security

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

## Performance

If the survive flag is implemented as a bit attached to a cell, the shrink opcode would need to iterate over all memory cells and detect which are marked to survive. To make this process constant time, the memory will be split in two subspaces as two account subtrees in the trie: the survive subspace and the perish subspace. The bit 255 of the keys will be used to decide which subtree the key belongs. If the contract code wishes to mark a cell to survive flag, the cell must be moved from the perish subspace to the survive subspace by setting the bit 255 of the key, and vive-versa. This means that the actual hash security of the trie is 1 bit less. When looking up a cell, the contract must first search the surviving subtree and if not found it must search it in the other subtree. This must be done by masking the bit 255 and setting the bit 255.The contract must make sure a key will never be present in both subtrees. Removing a key requires trying removing the key in the first subtree, and if not found re-trying in the opposite subtree. The contract designer must decide to use or not the two available subspaces, and can ignore them. In that case the author can decide to place all keys in the survive or perish subspaces.

## Sample Use Cases

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

[RSKIP08]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP08.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
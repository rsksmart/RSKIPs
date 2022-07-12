---
rskip: 27
title: Highly Efficient Storage Rent
description: 
status: Draft
purpose: Sca, Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-12-29
---

# Highly Efficient Storage Rent

|RSKIP          |27           |
| :------------ |:-------------|
|**Title**      |Highly Efficient Storage Rent|
|**Created**    |29-DIC-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca/Fair |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes that contracts should pay storage rent, to reduce the risk of storage spam and to make storage payments more fair. At the same time this RSKIP discusses the limitations of storage rent due to the additional complexity and overhead that, in some cases, overweight the benefits.

# **Motivation**

One of the problems of the RSK platform is that memory can be acquired at a low cost and never released, forcing all remaining nodes to store the information forever. There almost no examples in real-world commerce where users acquire with a single non-recurring payment eternal rights over a property that requires continued maintenance and therefore implies a periodic maintenance cost to a third party. The cost of maintenance is low but non-negligible, as persistent data must be stored in SSD so access cost matches real cost. That is the case of blockchain state storage, The cost is multiplied by the number of state replicas in the network. In some cases space is given for free (e.g. google drive space), but this is because space is subsidized by other services the google user consumes. Also there is no guarantee Google will offer free space forever. It can be argued that full nodes are altruistic, and therefore they are willing to incur in any storage cost network demands. While this may have been partially true for Bitcoin nodes in the past, this altruistic behaviour can decrease. The number of Bitcoin nodes has been declining, while the number of Bitcoin users has increased considerably, meaning that new users are not willing to run full nodes more than old users. It is expected that block pruning and sharding techniques enable users to commit certain partial amount of storage, but not for the full blockchain. However, the verification of new blocks, more than the historic storage, is what defines a full node. To verify a block, a node needs the full state, or receive Inclusion proofs for all state data used. The sharding factor must be inversely proportional to the number of honest host a peer connects to, so if the state size grows, and other factors remain constant, the local storage must also grow. Therefore, in principle, users should pay a storage rent (e.g. bitcoins/month) for consuming persistent storage. However it is not clear who should pay for this rent. Many contracts are examples of crowd-contracts: programs that are fueled and used by the crowd, therefore they can consume a lot of memory, but no single user is in position of carrying the burden of the rent.  both in terms of monetary effort and the fact that no single user may have the incentive to carry out the task, whatever the cost is.

A well designed crowd-contract should have a revenue generation method for paying for the storage rent. For example, each crowd-contract operation should be accompanied by a payment in bitcoins to a special rent sub-account where the crowd-contract collects all rent oriented income. However, as most crowd-contract are immutable, such revenue collecting method must be defined at day zero, and at that stage it will be unclear if the revenue model can sustain the memory rent. [RSKIP21] presents the problems in depth with storage rent. The main problem is efficiency: most rent payments are micro-payments and therefore the cost to pay (the overhead) is excessive. Another problem is scalability: the system to collect rent may introduce new inefficiencies (such as repeated state writes) interfere with the scalability plan.

Three approaches have been devised:

1. Contracts pay rent once every period using the DEPOSIT opcode.

2. The VM automatically collect rents on calls based on the last time the contract was accessed and the memory size. The cost is 20 gas per byte per year. The cost is rounded up. For example, a contract of 8 Kbytes pays 160K gas a year, or 438 gas a day. This is a socialized cost, since users pay for other users memory consumption. If the rent is not successfully paid for one year, the contract is hibernated (TBD). In case the call do not change the storage or account state of the target contract (including the balance) then, the last access time is not updated and no rent is paid. Contracts can be marked as libraries and are forbidden to hold balance or to store data.  To compensate for the fact that libraries will be immortal, the price of the CALL to library contracts is increased by 1 unit of gas per 8 bytes of code in the library contract. In other words, a call to a library contract of 8 Kbytes in size requires an additional payment of 1024 gas per call. Accounts behave as normal contracts. 

3. The rent is paid in full if a message/call is sent and the deadline is automatically postponed. This is useful for accounts (which consume very little space). The last access time is written at the same time the new balance is changes. 

4. All calls to contracts/accounts extend the lifetime to the contract/account in proportion to the gas consumed by the call (e.g. 700 for CALL or 21K for transaction plus variable consumed) and the memory. The deadline is postponed to up to one year. If the deadLine is in the past, the contract can be hibernated. A call and a transaction are extended to allow spending gas without executing code (spendGas field). To prevent hibernation, user can just call the contract with a high value of SpendGas. In case the call do not change the storage of the contract (including the balance) then the price of the CALL is increased by 1 unit of gas per 8 bytes. Each call to a library contract of 8 Kbytes in size requires an additional payment of 1024 gas.

This RSKIP describes solution 2.

# **Specification**

Each normal contract (not libraries) or account has a new field lastChangeTime. Let d be the timestamp of the block in which the call is executed. Both fields are given in seconds. When a contract call finishes, the VM checks if the contract state has changed, if so following value is computed:

if (d>lastChangeTime)

rentGas =  (storageSize+codeSize+256)*(d-lastChangeTime)*rentPrice/2^32

else

rentGas = 0

There are 31536000 seconds in a year.  2^32 =4294967296

SecondsAYear/2^32= 0.0073425471782684326171875

The initial rentPrice is set so that a SSTORE cell consumes 3000K a year, therefore:

Currently 

128*rentPrice * 0.0073425471782684326171875 = 2000

  rentPrice  = 2000/0.0073425471782684326171875/ 128 =2128

This value is added to the gas consumed. If no more gas is available, the out-of-gas exception is raised. On success, the value of lastChangeTime is updated.

The cost of a byte is 15.625 gas per byte a year.

The storage size  (storageSize) is computed as 128*N where N is the number of entries in the storage trie.

If there are several calls in the same transaction or the same block, only the first will pay, because the remainder will have (d==lastChangeTime)

When a contract is created, the lastChangeTime is set 6 months in the future. This means that some rent is prepaid. 

If a rentGas is lower than 600 gas and no value transfer has been made, then the rent is not paid, nor the lastChangeTime is updated. This protects from micro-transactions. For example, the rent of a minimal contract cannot be paid in advance so easily by executing a call to it: an almost empty contract consumes only 256*15.625=4000 gas/year, and therefore one can only pay the rent once every 50 days.

## Special cases for CODECOPY, CODESIZE and BALANCE

The CODECOPY opcode can be used to use a smart contract as a library of data (e.g. a sine table). This means that a library contract may never be accessed through CALLs. Therefore the new cost of a CALL (1 unit of gas per 8 bytes of target contract size).

For simplicity, CODESIZE and BALANCE are not modified.

## New opcodes

Let m be the amount of memory persisted by the contract in 32 byte words.

### HIBERNATE

Arguments: <address or prefix of address> <size of prefix in bits> 

GasCost: provided by the caller 

returns: nothing

Implementation on with delayed hibernation:

This opcode doesn’t do anything immediately: it just adds the address prefix to a list of hibernations that will be carried on at the end of the block processing (hibernationList).

When all transactions in the block have been processed, the hibernationList is iterated. For each prefix in the list the following action is taken:

If a prefix of an address is given (size<256) then the platform will try to find that node on the trie. If the node is not found, an error is returned. If the node is found, then the following algorithm HibernateNode(node) is executed:

HibernateNode(node):

- r = Scan(node)

- if r = null return

- ProcessParents(node)

Scan(node):

- if deadline is surpassed, hibernate node, return address of node

- if node is terminal node (account or contract) then return null

- r=Scan(node.left)

- if r is null return null

- r=Scan(node.right)

- if r is null return null

- //Both childs have been hibernated, so hibernate self

- Build hibernation record at node, clear node.left and node.right

- return node addess

ProcessParents(node)

- if node.parent.left=node and node.parent.right.isHibernated() or 

    node.parent.right=node and node.parent.left.isHibernated() then

  - Build hibernation record for parent and call ProcessParent(parent)

### WAKEUP

Arguments: contract_address code_ofs code_size trie_ofs trie_size

Returns: error_code

## HIBERATION cost 

The cost of hibernation does not depend on the size of the memory, since this RSKIP will be implemented on top of the new Trie structure (persistent memory below account address on the Trie). Therefore the root hash of the memory subtree need not be computed.

Implementation with immediate hibernation:

If hibernate is called and it returns 1 (meaning an hibernation has taken place) HIBERNATE opcode consumes no gas at all. If HIBERNATE is called but it returns 0 (hibernation not reached) the opcode cost is 300 gas units. Internally, the 300 gas cost is first deducted, and then given back in case the hibernation succeeds. 

Implementation with delayed hibernation: 

The cost of hibernate is always 200 gas. Therefore if gastLimit is 4M, there can be 20K hibernations requests per block.

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
    <td>10000</td>
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
    <td>-15000 </td>
    <td>-5000</td>
    <td>from non-zero to zero
</td>
  </tr>
  <tr>
    <td>CODEBYTE</td>
    <td>200</td>
    <td>100</td>
    <td></td>
  </tr>
</table>


First, the rationale behind CLEAR_SSTORE is that by zero-ing a cell it is actually deleted, so space is saved. However, to determine that a cell has been deleted, it must be first read, and reading pertain a high cost of disk access. All current SSTORE actions require the cell value to be previously read, which is in most cases unnecessary. Reads are blocking: the code cannot proceed execution until the address is fetched. However writes are non-blocking: writes can be cached and executed at the end of the contract.

The underlying problem is that Ethereum does not clearly states if contract storage is pre-loaded when the contract is called, or storage cells are loaded on-demand. Pre-loading does not seems the right approach, as to do this in an optimized fashion requires compacting all contract storage space into a single consecutive disk data chunk. Compacting storage is expensive, and that cost is not paid by SSTORE. 

In RSK platform the premise is that each cell read may require a SSD access. To prevent unnecessary pre-reads, all SSTORE operations consume a constant gas cost, and when the contract finishes some refunds are made depending on the existence of pre-existing cells. This allows in the future that all reads are performed at the end of CALL. However, reads cannot be moved to the end of transaction processing, since the cost of CALL must be known before exiting the CALL to be able to refund the caller with the exact amount. However, reads can be performed in parallel.

Initially [RSKIP21] proposed reducing the cost of the initial acquisition of storage from 20K to 3K. However, as discussed on [RSKIP21], this reduction allows Spam DoS attacks.

<table>
  <tr>
    <td>Identifier</td>
    <td>value</td>
    <td>net cost</td>
    <td>when is paid</td>
  </tr>
  <tr>
    <td>SSTORE </td>
    <td>10000</td>
    <td></td>
    <td>at execution</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_Z_NZ</td>
    <td>0</td>
    <td>10000</td>
    <td>from null to non-zero (refund)</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_NC</td>
    <td>9700</td>
    <td>300</td>
    <td>value not changed 
(based on refund)</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_NZ_NZ</td>
    <td>9000</td>
    <td>1000</td>
    <td>changed from non-zero to non-zero (based on refund)</td>
  </tr>
  <tr>
    <td>REFUND_SSTORE_NZ_Z</td>
    <td>5000</td>
    <td>5000</td>
    <td>from non-zero to zero
(based on refund)</td>
  </tr>
  <tr>
    <td>SSTORE_YEARLY </td>
    <td>2000</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>SSTORE_YEARLY per byye (aprox)</td>
    <td>15.625</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>CODEBYTE</td>
    <td>100</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>CODEBYTE_YEARLY </td>
    <td>15.625</td>
    <td></td>
    <td></td>
  </tr>
</table>


It is important that the implementation checks if a cell address exists in the trie without retrieving its value. The RESET_SSTORE cost is split into two new costs: NC y NZ_NZ.

The costs of SSTORE is modified. The first time a call context executes SSTORE, 10K gas is always deducted. If later SSTORE is executed against the same cell, then only the net value is deducted. 

Example:

<table>
  <tr>
    <td>Previous Value of cell at 0x01</td>
    <td>Instruction executed</td>
    <td>Cost</td>
  </tr>
  <tr>
    <td>0x00</td>
    <td>SSTORE 0x01, 0x05</td>
    <td>10K gas</td>
  </tr>
  <tr>
    <td>0x05</td>
    <td>SSTORE 0x01, 0x06</td>
    <td>1000 gas</td>
  </tr>
  <tr>
    <td>0x06 </td>
    <td>SSTORE 0x01, 0x06</td>
    <td>300 gas</td>
  </tr>
  <tr>
    <td>0x06</td>
    <td>Refund when CALL returns</td>
    <td>0 gas</td>
  </tr>
</table>


Another example:

<table>
  <tr>
    <td>Previous Value of cell at 0x01</td>
    <td>Instruction executed</td>
    <td>Cost</td>
  </tr>
  <tr>
    <td>0x06</td>
    <td>SSTORE 0x01, 0x00</td>
    <td>10K gas</td>
  </tr>
  <tr>
    <td>0x00</td>
    <td>SSTORE 0x01, 0x05</td>
    <td>10K gas</td>
  </tr>
  <tr>
    <td>0x05 </td>
    <td>SSTORE 0x01, 0x00</td>
    <td>-5000 gas (refund)</td>
  </tr>
  <tr>
    <td>0x00</td>
    <td>Refund when CALL returns
(from 0x06 to 0x00)</td>
    <td>-5000 gas (refund)</td>
  </tr>
</table>


## CREATE_DATA (CODEBYTE) cost

Currently every byte of code added pays 200 gas units. This is too high as code stays together and can be stored in a single chunk of disk space. A  cell of 32 bytes of SSTORE data is being priced at 10000 units of gas, but it actually occupies in memory approximately 128 bytes (address+data+child_pointers_overhead+hash=~128). So SSTORE data is priced at 78 units of gas per byte. Therefore CREATE_DATA will be reduced 2 times, to 100 gas units per byte. This is higher than the recurrent cost.

## Loading code on CALL

This RSKIP is intended to work with RSKIP30 (Code page pagination)

## WAKEUP cost

To wake up a contract, the user must provide the spv path and the required hash pre-image. Transferring that data in a transaction has a cost of x per byte. The WAKE UP opcode itself also has a costs that comprises the following sub-costs. The costs represent the cost of one tenth of a year rent. 

* Code byte cost  (1.5625 gas units per byte)

* Storage cell cost (200 units per cell)

* fixed cost to recover balance and other contract internal fields (400 units)

## Can Self-Hibernation save money? Short answer: Generally not.

To see how hibernation can save money in certain cases, imagine a contract with the following properties:

- Code size: 1024 bytes

- Storage size: 4 cells

- Total bytes to transfer (without cell addresses): ~128 bytes

The contract is programmed so it self-hibernates after each operation. The cost of maintaining this contract active for 1 year is : (8*128+1024+256)*15.625=36K units of gas.

The cost of awakening is one tenths: 3.6K 

The cost of transfer every non-zero byte is 68 gas units. Therefore transferring 128 bytes in every transaction costs 8704. The cost of transfer and awake is approximately 12K. Clearly there is only a benefit if the contract will be used less than 36/12=3  times a year.  

Self-hibernation could save money if data could be transferred at lower cost. One of such methods would be ephemeral segwit data, described in [RSKIP28].

Another way is to program the contact to keep all data in volatile memory and only store in the state a hash of the data. Also the contract can store all the code in a library. Therefore the previous contract could consume as little as :

- Code size: 22 bytes (DELEGATECALL)

- Storage size: 1 cell

The contract is programmed to self-hibernate after each operation. The cost of maintaining this contract active for 1 year is : (1*128+22+256)*15.625=6343 units of gas.

The cost of awakening is 634 gas 

The cost of transfer is still 8.7K

The cost of awake+transfer is 9K.

Therefore this only has a benefit if the contract will be used once every 16 months.

Because sending 32 bytes costs 2176 while a year of storage of a cell costs 128*15.625=2000, the self-hibernation method will provide a benefit mostly if the contract uses many storage cells and the cell addresses can be guessed by the code (e.g. are fixed). This is because a full node has a lot of overhead storing a cell.

One additional improvement would be to implement an opcode PROXYCALL that is similar to DELEGATECALL but does not leave anything on the stack, so basically it works as if the code had been replaced temporarily.

[RSKIP21]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP21.md
[RSKIP28]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP28.md


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
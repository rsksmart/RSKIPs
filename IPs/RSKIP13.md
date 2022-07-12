---
rskip: 13
title: Support for stable assets & token issuance
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2016-08-08
---

# Support for stable assets & token issuance

|RSKIP          |13           |
| :------------ |:-------------|
|**Title**      |Support for stable assets & token issuance|
|**Created**    |08-AUG-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

It is of utter importance that the RSK platform to efficiently support stable assets. This RSKIP discusses different approaches to improve the RSK platform to provide better support for stable assets by reducing bottlenecks, increasing tps, security and flexibility.

# **Motivation**

Most of the use cases for financial inclusion involve trading stable assets (fiat-denominated or baskets). The creation of stable assets should not be a privilege of a company, but a feature of the platform. This implies that the platform should allow the creation of user-assets, including all variants such as company shares, property and debt. User assets are generally created in Ethereum by a single master ERC-20 contract, that we will call an ASSET contract. Such contract creates the asset (generally a one time operation) and receives requests to transfer ownership of assets via method calls. The problem with having a single ASSET contract is scalability.  One of the planned features of the RSK platform is scaling by multi-threading verification, as described in [RSKIP04] and [RSKIP02]. This is in-line with current technology forecasts that state that microprocessors will continue adding more cores in the future, as clock rates will not be increasing much due to physical, technological and cost limitations. Multi-threading verification requires transactions executed by different threads to be independent (they do read and change access the same part of the state tree). The [RSKIP04] proposes that as long as contracts do not change their persistent state, they can be called from different threads without side-effects. This leads us to the idea that the ASSET contract may cease to be a bottleneck if all memory writes are done on separate memory areas, one for each receiver. It’s important to note that the asset transfer operation is not like any other contract operations: asset transfers have the property of commutativity for the receiver side (payee). An account may receive several consecutive transfers and as long as no outgoing payment is made, a change in the order of the transfers cannot make them fail or change the final effect.  As a comparison, asset transfer from the sender side is not commutative, and the effects of two consecutive transfers depend on the order of execution: the second transfer may fail if there are not enough funds after the first transfer. Also two transfers from different payers to different payees are commutative: there is no cross-effect. But as we already said, when user assets transfers are managed by a single contract for accounting both increases and decreases of user balances as mandated by ERC20 standard, that contract becomes a bottleneck for scalability since there is no commutativity. scalability should be considered when deciding which model will be the preferred model for stable assets.

There are three possible approaches to remove the bottleneck:

* ETH ERC20

* SPLIT

* DISTRIBUTED MEMORY

* BALANCE CONTRACTS

* BALANCE CHILD CONTRACTS

We explain each approach in more detail:

## 1. ETH ERC20

All operations are controlled by a smart-contract ASSET. In this case ASSET must hold the balance of each asset owner in persistent memory. The drawback of this solution is that wallets must implement two different wallet systems, one for the native coin (ether) and another for other assets.

This is the core method of a ERC20 contract (transfer):

<table>
  <tr>
    <td>contract StandardToken is TokenInterface {

    // token ownership
    mapping (address => uint256) balances;

    // spending permision management
    mapping (address => mapping (address => uint256)) allowed;
    
    
    function StandardToken(){
    }
      
    function transfer(address to, uint256 value) returns (bool success) {
               
        if (balances[msg.sender] >= value && value > 0) {

            // do actual tokens transfer       
            balances[msg.sender] -= value;
            balances[to]         += value;
            
            // rise the Transfer event
            Transfer(msg.sender, to, value);
            return true;
        } else {
            
            return false; 
        }
    }
   ….</td>
  </tr>
</table>


Benefits:

* Compatible with Ethereum

* The ASSET contract could request fees for payments and/or for deposit time (demurrage).

* Since holders can be enumerated, dividends can be paid. However, enumerating holders could require more gas than the block gas limit, and such functionality can be error-prone to implement right. 

Drawbacks:

* Since the balance of user holdings must be maintained by a central contract, the solution does not scale, as all transactions in the user asset belong to the same dependency partition.

* The cost of a user transaction may be high, compared to bitcoin transactions.

* If we implement contract rent, then renting space for balance records in the ASSET registry would turn out to require micropayments. On-chain micropayments are too expensive and can generate unintended spam. For example, if a contract holds the balances of 1M users, then every year 1M users must send micropayments to pay for the rent. One way to solve this is by forcing probabilistic payments: once every storage period ends, a random subset of balance records are selected according to block hashes , and only those owners must pay the storage rent. Implementing this is challenging, also due to block gas limit. Also the fairness of this solution can be questioned.

## 2. SPLIT

This method splits asset issuing from asset transfer. Every contract can issue assets using the ISSUE opcode. That opcodes creates a balance in the contract account containing the pair (asset-id,value) where the asset-id corresponds to the contract address and the value is chosen by the user. Afterwards payments in asset-id can be executed by standard transactions. New CALL/SEND opcodes (CALL_ASSET/SEND_ASSET) must be created to support the transfer of user assets. Wallets can reject transfers if being sent assets using the CALL_ASSET opcode, but cannot reject assets if being sent using the SEND_ASSET opcode.

In the SPLIT model, there is no transfer() method in the issuer contract.

 

Benefits:

* Wallets can use an uniform system to query an account balance in Bitcoin and user assets.

* It’s easy to create SPV nodes for user assets.

* The cost of user asset transfer is minimized

* User assets transfers can be parallelized (using dependencies)

* In case the ASSET contract is hibernated, users can still transact with the user asset.

* The model is fully scalable, since there is no central serialization bottleneck.

Drawbacks:

* No way to request user asset denominated fee to be paid to the issuer per transaction.

* No way to specify demurrage

* No way to enumerate holders and pay dividends.

* As wallets can receive any transaction, then wallets can be "spamed" by thousands of user assets of no real value. The ACCEPTVALUE opcode could prevent spam by filtering which assets are accepted in CALL_ASSETs. However, the SEND opcode cannot be protected. The platform could allow the user to specify which user assets an account is willing to accept, by adding the opcodes ADD_ASSET <asset-id> <minimum-amount> and REMOVE_ASSET <asset-id>.

* Transaction fees will still be paid in bitcoin, which means that external transactions must hold and spend bitcoins even if they are willing to transact in a user asset. It’s seems annoying that users of fiat currencies over RSK must worry about having a bitcoin balance.  A solution to this problem is that the issuer pays for the transaction fees: this can be done using multi-input or aggregated transactions.

## 3. DISTRIBUTED MEMORY

As specified by [RSKIP01] and [RSKIP06], distributed memory can be used in the ETH-ERC20 case so that there is no scalability bottleneck. The basic idea of distributed memory is that a contract can store information not only in its local memory, but in memory of foreign contracts. To prevent spamming, contracts can allow or deny offering local memory to other contracts using new opcodes. Foreign records are protected in contrast memory: only the creators of such records can change them. The ASSET contract would make a payment by reading and writing balance records present in the source and destination contracts of the payment. Even if all transfers go through the ASSET contract, since the asset contract does not write any local field, the VM can parallelize several ASSET contract calls without side-effects.

This would be the transfer method of the ASSET contract. Note that a key keyword foreign is added to the balances mapping to indicate the memory is stored in foreign contracts. Foreign mappings and fields are accessed by: at(address).field.

<table>
  <tr>
    <td>contract StandardToken is TokenInterface {

    // token ownership
    foreign uint256 balance;

    function StandardToken(){
    }
      
    function transfer(address to, uint256 value) returns (bool success) {
               
        if (at(msg.sender).balance >= value && value > 0) {

            // do actual tokens transfer       
            at(msg.sender).balance  -= value;
            at(to).balance          += value;
            
            // rise the Transfer event
            Transfer(msg.sender, to, value);
            return true;
        } else {     
            return false; 
        }
    }
   ….</td>
  </tr>
</table>


However, if storage rent is implemented (see [RSKIP61]) user should be able to select which memory cells (user assets) should be stored in their wallet contracts and which should not. If not, then there wallets could be spammed by foreign assets. New opcodes ADDREMOTEOWNER / REMOVEREMOTEOWNER must be used to allow/deny that external contracts use local memory. 

**Paying fees per transfer**

To allow user asset transfers to pay demurrage/transfer fees and still be scalable, the ASSET contract should avoid writing to a single ASSET persistent storage/value. Instead it should divert the payment of fees by calling one of many a sub-contracts which actually receives the payment. This requires the ASSET contract to randomly split user asset calls depending on some random property of the call. One way of doing it is by adding a NONCE opcode which returns the nonce of the transaction originating the payment. Therefore HASH (block-id | msg.origin | msg.nonce) is an unique value that can be used for splitting. New opcodes to allow this (and more discussion) is provided by [RSKIP11].

The following is a sample contract that implements fees:

<table>
  <tr>
    <td>contract StandardToken is TokenInterface {

    // token ownership
    foreign uint256 balance;
    
    address feeAddrs[];

    function feeAddress() returns (address dst) {
       uint index = hash(blockhash,msg.origin,msg.nonce) % maxFeeAddrs;
       dst =feeAddrs[index]; 
    }
      
    function transfer(address to, uint256 value) returns (bool success) {
               
        if (at(msg.sender).balance >= value && value > 0) {
            uint fee = value / 100;
            at(feeAddress()).balance +=fee; 
            value -=fee;  
            at(msg.sender).balance  -= value;
            at(to).balance          += value;
            
            Transfer(msg.sender, to, value);
            return true;
        } else {     
            return false; 
        }
    }
   ….</td>
  </tr>
</table>


Another problem is what happens if a user sends a non-zero value to the ASSET contract: since that corresponds to a change in the state of the ASSET contract, it collapses all dependency partitions into a single one. Also a user can send another user asset to the ASSET contract.Therefore sending coins to an ASSET contract can be used as a vector of attack to prevent widespread use of that particular user asset. This can be prevented if:

- SEND call (without code exec.) is not implemented, but ACCEPTVALUE is.

- A contract can specify in its flags that is not willing to accept external values.

- A contract can specify in its flags that is a static contract (no state). This property is stronger. The system must make sure that changes in these flags are only reflected in the next block.

Another vector of attack would be if a user sends a user asset to the same contract that issued the asset, or the attacker sends very tiny amounts to all of the fee receiving contracts. That also could force the collapse of dependency partitions. This can be prevented if the ASSET contract checks for the destination address and forbids transfers to itself or one of its fee receiving contracts, but it seems that a more granular control approach is needed, such as filtering the source.

Therefore we see that this approach increases the attack surface and requires several additional protections.

Benefits:

* Scalable

* Shift the responsibility to pay contract rent for owned assets to the users owning such assets. Users can pay rent for multiple user assets at the same time, reducing the impact of the micropayment problem.

* The hibernation of wallet contract due to unpaid distributed storage rent has much lower consequences to the ASSET contract than a hibernation of the ASSET contract itself, which halts all operations with the user asset.

## 4. BALANCE contracts

The ETH-ERC20 method can be improved to store user balances in user assets in ad-hoc contracts.

Therefore the ASSET contract contains a registry of (owner-address,manager-contract-address). When a payment in user asset is received by the ASSET contract, it calls the manager contract of the sender to reduce the balance, and then calls the manager contract of the receiver to increase the balance.  This way non-overlapping transactions in the user asset can be parallelized without requiring distributed memory. 

<table>
  <tr>
    <td>contract StandardToken is TokenInterface {

    // token ownership
    mapping (address => address) manager;
      
    function transfer(address to, uint256 value) returns (bool success) {
            
        uint prevBal =ManagerContract(manager[msg.sender]).getBalance() ;    
        if (prevBal>= value && value > 0) {

            // do actual tokens transfer       
            ManagerContract(manager[msg.sender]).sub(value);
            if (manager[to]==0)
               createManagerContract(to);
            ManagerContract(manager[to]).add(value);
            
            // rise the Transfer event
            Transfer(msg.sender, to, value);
            return true;
        } else {
            
            return false; 
        }
    }
   ….</td>
  </tr>
</table>


CALLs between contracts are expensive, and each transfer would require two calls. This approach still has the problem that a new contract must be created to handle every user asset account, which has a cost. Creating a contract costs 32000 gas units. This is very close to the cost of a single transfer message, so it’s not relevant.

Benefits: 

* allows most of the benefits of distributed memory without more complexity and opcodes.

Drawbacks: 

* There is still the problem of micropayments for storage rent. But is important to note that if we allow a small memory use without storage rent (e.g. 32 bytes), then managing contracts can live forever. Also if managing contracts get hibernated, they could be de-hibernated only when a payment from those addresses needs to be performed, joining the de-hibernation payment with another payment thus preventing micropayments spam.

* Using the manager approach, to create an account, a user must call a method on the ASSET contract to create a managing contract. But this forces the ASSET contract to serialize transactions.  

## 5. BALANCE CHILD contracts

To prevent excessive use of CALLs between the main ASSET contract and the manager contracts, we modify the BALANCE approach to allow the creation of "child" contracts, which are contracts that can be fully controlled by another parent contract. The parent contract would be able to read and write the child contract persistent storage. In that case, the the code would look just as the case shown before that used the “foreign” keyword, but replacing “foreign” by “child” keyword. Each child contract is indexed by a user address, which can be any 256-bit number. The best implementation of balance child contracts requires modifying the state tree so that there can be longer keys and child contract are childs of its parent account. Another approach to implementation is that child contract addresses are generated as a hash of the parent contract address and the child user address.

Benefits: 

* Highly scalable

* Allows most of the benefits of distributed memory without more complexity

* No high storage rent for the ISSUE contract (central cost)

Drawbacks: 

* Requires modifying or adapting the state tree.

## Another approach to undesired transaction serialization

To prevent the serialization of transactions it would be best if contracts could specify a set of additions to memory that are queued until the end of the block execution, so the order of additions become irrelevant. The problem is what happens if two elements are added having equal key but different values (which one will prevail?). There are several ways to solve the problem: 

* partitioning the managing contracts registry in several sub-registries stored in sub-contracts. 

* allowing the ASSET contract to handle per-memory entry dependencies. If memory cell A is read, then no other transaction can write it. if memory cell B is written, then no other contract can read it. Then the creation of accounts would require adding cells, that won’t be used by other transactions. But managing per cell dependencies seems to be excessive housekeeping, adding a lot of cost to memory access.

* Grouping a single write access transaction followed by read-only transactions.  

* As presented before, creating child contracts (or sub-contracts). Sub-contracts are small contracts that are created by a contract that are accessed by a user-chosen key. Internally, a sub-contract can mapped to an element whose key is the hash of the owning contract address concatenated with the user selected key. Then an ASSET contract can create sub-contracts to user managing accounts using the user address as key. 

## Summary

The following table represents the main benefits and drawbacks of each solution:

<table>
  <tr>
    <td>Approach</td>
    <td>Main benefits</td>
    <td>Main drawbacks</td>
    <td>Impl. complexity</td>
  </tr>
  <tr>
    <td>ETH ERC20</td>
    <td>Compatibility with ETH</td>
    <td>Low scalability
High central storage rent</td>
    <td>0</td>
  </tr>
  <tr>
    <td>SPLIT</td>
    <td>Elegant
Highly Scalable
SPV ready
Reduced central storage rent</td>
    <td>No asset transfer fee/demurrage
No business model</td>
    <td>3</td>
  </tr>
  <tr>
    <td>DISTRIBUTED MEMORY</td>
    <td>Scalable
Reduced central storage rent</td>
    <td>Increases attack surface
Requires modifying account model.</td>
    <td>4</td>
  </tr>
  <tr>
    <td>BALANCE CONTRACTS</td>
    <td>Scalable</td>
    <td>Account creation bottleneck</td>
    <td>1</td>
  </tr>
  <tr>
    <td>BALANCE CHILD CONTRACTS</td>
    <td>Scalable
Reduced central storage rent</td>
    <td>Requires modifying or adapting the state tree.</td>
    <td>3</td>
  </tr>
</table>


Currently I’m strongly in favor of BALANCE CHILD CONTRACTS, because it has most of the benefits and a few drawbacks. Secondly, I prefer BALANCE CONTRACTS, because it does not require any modification of the platform. Third I prefer SPLIT mode, because it leverages on the existent value transfer system and may allow miners to receive transaction fees denominated in other currencies in the future. However the SPLIT mode has no associated business model. To implement a business model for SPLIT, an implicit CALL to a manager contract must be implemented, complicating things. Note that all methods with the exception of SPLIT method allow ERC20 compatibility.

[RSKIP02]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP02.md
[RSKIP04]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP04.md
[RSKIP01]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP01.md
[RSKIP06]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP06.md
[RSKIP61]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP61.md
[RSKIP11]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP11.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
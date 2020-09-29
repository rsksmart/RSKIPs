# Implementing Storage Rent in RSK

|RSKIP          |Unassigned          |
| :------------ |:-------------|
|**Title**      |Implementing Storage Rent in RSK|
|**Created**    |September 29, 2020 |
|**Authors**     |Sergio Demian Lerner, Diego Masini, Shreemoy Mishra  |
|**Purpose**    |Sca, Fair, Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This proposal intends to improve resource utilization in RSK by charging users "rent" for storing state data (e.g. account state, contract code, and contract storage) in Unitrie nodes. Storage rent can reduce the risk of storage spam and make storage payments fairer. It can also improve caching and help protect the network from some IO attacks. We propose a system wherein each transaction's gas limit is divided equally between two *gas budgets*: one for regular execution costs and another for rent. Transaction senders can  set the appropriate cumulative gas limit based on new and updated RPC methods e.g. `estimateGas` or `exactimateGas`. Just like execution gas, all collected rent is passed to REMASC for distribution to miners as transaction fees. Rent does *not count* towards block-level gas limits. So there is no adverse impact on the number of transactions that can be included in a block.

# **Motivation and Impact**
At present, incentives to reduce the use of state storage are provided through gas costs for op-codes like `SSTORE`, `CREATE-DATA`, and "refunds" for `SELF-DESTRUCT` and `CLEAR-SSTORE`. However, these are one-time incentives. Storage rent is a recurring cost and this provides continuous incentives for more judicious use of storage. 

Storing blockchain state information is not cheap, especially when one accounts for replication over many full nodes. However, there are also significant *opportunity costs* of storage and these are  related to disk IO performance. At a deeper level, storage rent is really about providing incentives to reduce disk IO costs by helping prioritize information to be held in various caches.  Our primary attention is on trie caches held in RAM rather than key-value datastores on disk. The necessity for fast IO operations is the reason why blockchain nodes use expensive SSDs instead of cheaper HDDs. If RAM becomes so cheap that entire blockchain state can routinely fit in cache - along with multiple snapshots needed for transaction processing -- then there would be no strong benefit from rent like mechanisms. 

In addition to improved node performance through caching, storage rent can also serve as a mechanism to enforce *penalties* for IO-attacks. Suppose attackers deliberately query nodes that they know not to exist in blockchain state. In these cases, we can collect "some" rent for these non-existent nodes - on the fly - as a penalty. The penalty (e.g. the equivalent of 6 months rent) will be small for honest mistakes, but can be significant for large scale attacks.

Other benefits of implementing storage rent include the possibility of using last rent paid timestamps to implement *node hibernation* down the line. 
 
Block gas limit does not apply to storage rent. Rent is an additional uncapped revenue stream for miners. Thus, the transaction fees paid in gas (execution + storage rent) may exceed a block's gas limit. 


# **Specification**
The current RSKIP takes a minimalistic approach to storage rent. It represents a simplification of several prior RSKIPs including [RSKIP21](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP21.md), [RSKIP52](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP52.md), [RSKIP61](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP61.md) and [RSKIP113](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP113.md). See [[1](https://bitslog.com/2018/01/22/storage-rent-revised/)] for additional information on motivation, concept evolution and impact of storage rent.

## Rent computation and collection
### Granularity
Rent is computed and tracked at the granularity of individual *value-containing* (terminal) nodes in the state Unitrie. Rent depends on a node's size and also the duration for which data is stored. A node's size (`nodeSize`) is computed as the node's value length plus 128 bytes (for storage overhead). Therefore, `nodeSize` only approximates the actual space consumed, e.g. it doesn't take into account embedded nodes. Apart from a contract's *storage root*, other non-terminal Unitrie nodes are excluded from storage rent computations.

### Tracking
Each Unitrie node will have a new field `lastRentPaidTime` to store the rent paid time stamp (e.g. in Unix seconds). Rent will follow a pay after use or post-paid model (except for new nodes, see below).

When a transaction is executed, all Unitrie key/value pairs that are queried (e.g. via `SLOAD`, `SSTORE`, or `EXTCODE*`) are stored in a cache. All *new* trie key/value pairs created by the transaction (new accounts, contracts, storage cells) are stored in a different cache. 

After a transaction has been fully processed, both caches are iterated, and storage rent is collected for every node. For all new nodes, approximately 6 months rent is collected in advance, and their  their `lastrentpaidtime` timestamp is set to `6 x 30 x 24 x 3600 = 15552000` seconds from now.


### Pricing and Frequency
Rent is computed in units of gas at a given rate R. A value R = 1/(2^21) *gas per byte per second* has been suggested in prior RSKIPs (e.g. RSKIPs 52, 61 and 113).

**Example computation:** Let `SecondsAYear` be 31536000. Each byte pays 1/2^21 gas per second. Therefore a storage byte pays `SecondsAYear/2^21=15.03` gas units a year. 

- Suppose a simple externally owned account (EOA) has a value length of 10 bytes. Including the overhead, this 'account-node' has a `nodeSize` of 138. Such an account will consume 2075 units of gas a year in rent. 
- A contract with 10K bytes in code and 100 storage cells would pay approximately 380K units of gas/year. 
- A popular contract with 100,000 storage cells will pay approximately 44K gas/day.

To reduce the number of trie or cache IO operations, rent is collected only if the amount due exceeds some thresholds. For *pre-existing* key-value pairs that are modified during a transaction -- e.g. storage cell value changed -- the rent-collection trigger is set at 1000 gas. A higher threshold (10000 gas) is used for nodes that were accessed but not modified e.g. contract code. If these thresholds are not met, then the nodes are not added to rent-tracking caches. Thus, larger thresholds reduce the  frequency of rent related trie IO operations.


For example, if a simple EOA account (with valuelength 10 bytes) is used regularly, the rent will be charged about twice a year. If is is inactive and only a contract checks its balance with the BALANCE opcode, then rent would be collected once every 5 years. 

### Payments
The computation and collection logic is fully automatic and part of transaction execution. All trie nodes *accessed*, *modified* or *created* by a transaction are automatically checked and added to rent-collection caches, provided the collection thresholds are met. Users cannot select or exclude individual rent payments.


Any rent payments due are paid for by the transaction's sender. As mentioned earlier, we propose a simple system wherein each transaction's gas limit is divided equally between two budgets: one for regular execution gas costs and another for rent costs. Transaction senders can set the gaslimit appropriately based on estimates of gas used for execution and rent.


At the end of a transaction all leftover execution gas or rent gas are refunded to the sender's account.

### Rent computation pseudo-code
```
// compute rent for pre-existing nodes (for new nodes 6 months rent is prepaid)
setRentDue(Trie node, boolean modifiedByTX){
    lrpt = node.lastRentPaidTime;
    currentTime = time.Now();
    timeDelta = currentTime - lrpt; //time since rent last paid
    
    // compute rent due, but only for nodes with past due rent
    if (timeDelta > 0) {
        // formula is 1/2^21 * (node valuelength + 128 bytes overhead) * timeDelta (seconds)
        rd = calculateStorageRent(node.valueLength, timeDelta);
    } 
            
    // if rent exceeds high threshold (modified status does not matter)           
    if (rd > notModifiedThresh){ 
        node.rentDue = rd;
        node.lastrentpaidtime = currentTime; //update timestamp
    } else { 
        // Check if amount exceeds lower threshold and node was modified 
        if (modifiedByTX && rd > modifiedThresh){
            node.rentDue = rd;
            node.lastrentpaidtime = currentTime; //update timestamp
        } else {
            // not worth collecting rent / updating trie at this time
            node.rentDue = 0L;
        }                    
}
```



### Example
Suppose user `Alice` sends a transaction to contract `gameOfLife` that modifies two of `gameOfLife`'s storage cells. Rent computations will be triggered for the trie nodes containing Alice's account information, `gameOfLife`'s account information, storage root, the node containing its contract code, and finally, both of its storage cells that were modified. At the end of the transaction, any rent due for any of these nodes is collected from the transaction's gaslimit (the part set aside for storage rent) and the `lastrentpaidtime` timestamps are updated. 

We expect that users will typically end up paying rent for their own nodes, and for some storage cells of contracts that users interact with. Occasionally -- by chance -- individual users may end up paying rent for the node containing a contract's code. We expect such costs to be small and to even out across users over time. In most cases, users can obtain reliable estimates of rent gas before broadcasting a live transaction.


### Exceptions or Reversions
If a transaction ends because of a OOG (not enough execution or rent gas) exception or REVERT, then 25% of the storage rent gas budget is consumed as compensation for IO costs. No rent timestamp updates are made in these cases. 


### Handling CALL transactions
The CALL op code allows transaction senders to specify a gas limit for CALLs ('child' or 'callee'). This provides an opportunity for the 'parent'/'caller' to continue execution in case the child CALL OOGs or experiences some other exception. We **do not** extend the same mechanism to rent gas. All available rent gas is passed to CALLs, and there is no way to construct user-specified rent gas limits. We assume that users will be careful when interacting with contracts they do not trust, otherwise some CALLs may consume their entire rent budget and the TX cannot proceed even if it has some execution gas left over.


# **Research Implementation**
We have developed a [research implementation](https://github.com/rsksmart/rskj/tree/storageRent2020_RSKIP113) of a RSK(J) node with storage rent.

Example of a CREATE transaction with rent computation using the experimental node (on regtest)
```
root@5429dcd447b9:~/code/rskj# ./gradlew test --tests BlockExecRentTest.executeBlockWithOneCreateTransaction
...
...
    
Sender: 1ddd173f36582f9de451b54553aa29eab2656a0b
Receiver: null                                     // contract creation
Contract: 27fcd2c5584134d8426e5028b35df6d4e26b2fd7 //tx.getContractAddr()
    
TX Data: 0x600160026000600160026000 //just some PUSH opcodes for testing gas use
TX value: 10                        //contract endowment

Exec GasLimit 100000    //50:50 split of TX gas limit (set at 200K)
Exec gas used 53706     //21K (base) + 32K (create) + data cost
Exec gas refund 46294

Rent GasLimit 100000    //50:50 split of TX gas limit (set at 200K)
Rent gas used 8130      //1 node rent updated (sender's) + 3 new nodes created
Rent gas refund 91870
    
Tx fees (exec + rent): 61836   // rent gas also passed to miners

No. trie nodes with `updated` rent timestamp: 1 // new Trie::putWithRent(key, value, timestamp)
No. new trie nodes created: 3   //repository changes for trie "access" from Program, TransactionExecutor, VM

Sender Bal 1938164       // initial balance was 2_000_000
Sender LRPT 1595968062   // Sender's account node last rent paid timestamp (LRPT)
Contract Endowment 10   
Contract LRPT 1611520062 // new contract node, LRPT is 6 months in future 
                             // (rent collection logic in  vm.program.RentData)

Block tx fees: 61836     // rent passed to REMASC with normal execution gas

```


# References
1. S.D.Lerner [Blockchain State Storage Rent](https://bitslog.com/2018/01/22/storage-rent-revised/) 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



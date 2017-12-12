# New Event Tree and Extended LOG 

Code: RSKIP45

Author: SDL

Status: Adopted

# Abstract

Currently lightweight wallets must scan all transactions receipts of token contracts to detect if there has been a transfer of funds or the wallet contract has changed its state. Block header bloom filters help to narrow the amount of blocks to scan, but bloom filters can easily be spammed by other contracts to collude on the keys that are being scanned by wallets. In the worst case, if a lightweight node does not receive collaboration from peers, it needs to download every transaction receipt to identify those that affect the state of a specific contract (generally a wallet) to be monitored. This is impractical. Transactions that affect the state of several other contracts will force light clients from downloading receipts containing information they are not interested in. This RSKIP defines a new Event trie whose root is committed in the block header, and where logged items are clustered by contract address, not by transaction index. To write information to this tree, LOG opcode is extended. 

# Discussion

The receipt trie is not well suited for lightweight wallets to use, therefore a new data structure  that reduce the stress on light clients is needed. This RSKIP introduces the Event trie structure, whose root is committed in the block header. The Event Trie is organized such that the keys are contract addresses, and each data element is itself a trie of logged events. The key of each second level trie entry is the log sequential number, and the payload is a RLP-encoded list of the list of topics, the LOG data, and the transaction index originating the log event. 

To log events in the new events trie, we propose the use of the same LOG opcode. Creating still another opcode seems not to be necessary and reduces compatibility with existent compilers. Also the same arguments would be required for the new LOG and the old LOG opcode, which consumes 4 byte addresses, one for each number of arguments. Therefore adding a new LOG opcode would consume for additional addresses, which seem excessive. It was decided to use the same LOG opcodes in an "extended mode". Two methods to switch to the extended mode were evaluated. 

1. Use a specific topic (e.g. 0xff..ff), to log the same message in the two different trees.

2. Switch to a new extended mode by other means using a previous code sequence, and then using the LOG event in the traditional way. 

The last option was chosen because the first present some compatibility problems, duplicates the amount of information logged at the same cost (LOG cost cannot be increased without breaking the 2300 minimum gas sent barrier).

To be as compatible as possible with Ethereum, we allow the VM to switch between old LOG mode and new LOG extended mode using the internal configuration register (COREG). This register is 32 bytes long, and is memory mapped. The description and reasons why we used a memory mapped register are described in RSKIP51. A single bit in the COREG changes the mode of LOG.

When transactions are processed in parallel (RSKIP03), the order in which the logged items are stored should still be deterministic. Generally each transaction that modifies its state belongs to a single verification thread, therefore all transactions that use the new mode of the LOG opcode will belong to the same thread and so the order of logged events is deterministic. 

However a contract that does not modify its state but logs information could be called from different threads. In this case a sequential log index does not work as key to the events trie. We could use as key the transaction index plus a sequential value which starts from zero on each new transaction. However, for simplicity, we decided using only a sequential number and force sequential processing of transactions in the cases of extended-mode LOGs without state modification. This is also because the new LOG mode modifies the account state, as follows.

To allow light clients to follow the list of events created by a contract, a new field is added to each account/contract: blockNumberOfLastEvent. This field is updated each time the contract emits a LOG in extended mode, with the current block number. The initial value of this field is zero. Light clients can use this field to skip to the previous emitted LOG each time they detect a new LOG, and therefore process the list of all logged items in reverse.

An alternate design is that each logged event key be a sequential number that does not reset on each block, but continues. The account state would store the sequential number of the las event logged. This design was not chosen, as it consumes more space, bringing little benefit.

To allow logging the previous value of blockNumberOfLastEvent before the LOG command is executed, a new opcode is added: LASTEVENTBLOCKNUMBER. This opcode pushes into the stack the value of the blockNumberOfLastEvent field. This way light wallets can log this value, and light clients can learn the block number of the previous logged event when they access any logged event.

# Specification

### Event Trie

The block header is modified. The field eventsRoot is added after the receiptTrieRoot

 field. The eventsRoot field is the SHA3 256-bit hash digest of the root node of the trie structure populated with each account events trie root, where keys are the contract addresses.

[contract address] --> [accountTrieRoot] 

Each accountTrieRoot is populated with 

 [index] ->rlp( rlpList(topics..), data, srcTxIndex )

The srcTxIndex is the index of the transaction in the block that originated the log. The index value starts from 0. The maximum index is the number of logs created by the source account in a block  minus one. It must be noted that each logged items does not contain a bloom filter, as opposed as an Ethereum log item.

All topics plus the account address are added to the block bloom filter, after hashed with Keccak, as Ethereum does.

Full nodes must verify the correct construction of eventsRoot when verifying a block.

### LOG in extended mode

The cost of the using the LOG opcode in the new mode is the same as the standard LOG opcode.

To enable the extended mode, a new bit EXTENDEDLOG in the COREG is defined. This is bit 0 (LSB), or bit-mask 0x01,  of the byte at offset 0x80….00 (most significant byte of word at that address). When this bit is set, LOG works in extended mode.

### New Account State

An integer field blockNumberOfLastEvent will be added to the account state. The account will be serialized in RLP as the following pseudo-code shows:

```
nonce = RLP.encodeBigInteger(this.nonce);

balance = RLP.encodeBigInteger(this.balance);

stateRoot = RLP.encodeElement(this.stateRoot);

codeHash = RLP.encodeElement(this.codeHash);

if ((stateFlags != 0) || (blockNumberOfLastEvent!=0)) {

  stateFlags = RLP.encodeInt(this.stateFlags);

  blockNumberOfLastEvent =RLP.encodeLong(this.blockNumberOfLastEvent);

  rlpEncoded = RLP.encodeList(nonce, balance, stateRoot, 

codeHash,   stateFlags,blockNumberOfLastEvent);

} else

   // do not serialize if zero to keep compatibility

  rlpEncoded = RLP.encodeList(nonce, balance, stateRoot, codeHash);

```

The decoding of the account state must take into consideration that the stateFlags and blockNumberOfLastEvent may be missing. The initial values of stateFlags and blockNumberOfLastEvent  is zero.

### LASTEVENTBLOCKNUMBER

This opcode has code 0xac. Has a gas cost of 2. It pushes into the stack the value of blockNumberOfLastEvent of the current account.

## Sample Code

The following RVM assembly code shows a fallback handler for incoming payments using the LOG opcode in extended mode:
```
CALLVALUE   // Store value in memory position 0

PUSH1 0x2, // Offset 32 in memory to store

MSTORE     // store in memory

PUSH1 0x01 // // change configurationRegister, set LSB

PUSH1 0xff //

PUSH1 0x02

EXP	    // Compute COREG address

MSTORE8   // config register changed

LASTEVENTBLOCKNUMBER

PUSH1 0x00 // Offset 0x00 in memory, last 8 bytes will end up in offs 24..31

MSTORE     // store in memory

PUSH1 0x12 // 1st topic: Sample topic

CALLER // 2nd topic: source address

PUSH1 0x28 // memSize, 40 bytes, 32b value + 8 bytes blocknumber first 64-bits.

PUSH1 0x18 // memStart, offset 24 (last 8 LSBs of LASTEVENT...)

LOG2
```


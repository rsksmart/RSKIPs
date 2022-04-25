---
rskip: 45
title: New Event Tree and Extended LOG
description: 
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-06-26
---

# New Event Tree and Extended LOG

|RSKIP          |45           |
| :------------ |:-------------|
|**Title**      |New Event Tree and Extended LOG |
|**Created**    |26-JUN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

# **Abstract**

Currently lightweight wallets must scan all transactions receipts of token contracts to detect if there has been a transfer of funds or the wallet contract has changed its state. Block header bloom filters help to narrow the amount of blocks to scan, but bloom filters can easily be spammed by other contracts to collude on the keys that are being scanned by wallets. In the worst case, if a lightweight node does not receive collaboration from peers, it needs to download every transaction receipt to identify those that affect the state of a specific contract (generally a wallet) to be monitored. This is impractical. Transactions that affect the state of several other contracts will force light clients from downloading receipts containing information they are not interested in. This RSKIP defines a new Event trie whose root is committed in the block header, and where logged items are clustered by contract address, not by transaction index. To write information to this tree, LOG opcode is extended. 

# **Specification**

See discussion [here](https://github.com/rsksmart/RSKIPs/issues/82).

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

[RSKIP51]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP51.md
[RSKIP03]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP03.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
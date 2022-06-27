---
rskip: 30
title: Code Pagination
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-01-01
---

# Code Pagination

|RSKIP          |30           |
| :------------ |:-------------|
|**Title**      |Code Pagination|
|**Created**    |01-JAN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP discusses how to deal with large programs and how calls to such programs should be priced. Currently Ethereum limits a program to 26K bytes to prevent loading an excessive amount of data on a call which has a low cost. However this reduces the capability and simplicity of programming of the platform considerably.

# **Motivation**

## Size of an RLP Encoded Account 

The following fields are encoded in an account (not contract):

nonce = (RLP.encodeBigInteger): avg 4 bytes

balance = (RLP.encodeBigInteger): avg 9 bytes

stateRoot = (RLP.encodeElement): avg 33 bytes

codeHash = (RLP.encodeElement): avg 1 byte

stateFlags = (RLP.encodeInt): avg 1 byte

rlpEncoded = (RLP.encodeList): avg 50 bytes

       

The node in the trie has the additional overhead of :

Object overhead:  16 bytes?

pointer to partial/total byte[] key of this node: 8 bytes

partial/total byte[] key: 32+16=48 bytes

pointer to right node: 8 bytes

pointer to left node: 8 bytes

Total: 88 bytes.

Total RLP + overhead = ~138 bytes.

## Discussion

The amount of code that must be loaded when a contract CALL is executed is variable. Ethereum hard-forked to prevent excessive code size to be used as a vector of attack. However, other approaches are possible. It must be noted that loading from SDD/HDD a single byte or a chunk of 36 Kbytes takes the same for a modern OS, since data is stored in blocks. The problem of program size can be solved in several ways: 

- establishing an upper bound (as Ethereum has done)

- cost of call be in proportion to the size of the code to load

- paginating program memory.

The first approach forces the compiler to split a long program into libraries, and use DELEGATECALL as a means to call between them. The problem with this approach is that DELEGATECALL costs 700 gas, while a normal call within the same contract costs 3+8+3+8=22 gas (push retAddress, push callAddress, JUMP, JUMP). This means that the split into libraries must obey a structural rule, such that there are much more inter-library calls than cross-library calls. 

The second approach has the drawback that callers need to know the code size and callbacks cannot  be easily programmed.

The third approach seems to be the best. 

# **Specification**

Code memory is paginated. each page is 8 Kbytes in size. All the static checks are performed when a contract is created, not when a method is called. At the creation time by an external transaction two checks are performed: first the initialization code (containing the payload) and then the payload itself (when it is returned). If the creation is done by the CREATE opcode, then a single check needs to be performed. Since the cost of each code byte is 100 gas (assuming [RSKIP27]), if there is a 4M gas limit, the maximum code size is 40K bytes, or 6 pages. The check consist of:

- Making sure that no PUSH opcode overlaps the 8K boundary with its payload. This is because JUMP and JUMPI instructions are not allowed to jump into the payload of PUSH opcodes

When the program counter wraps around a page or when a JUMP that points out of a page is executed then the missing page is loaded and a cost of 400 gas units is consumed.. The current RSK platform performs a first pass on code to collect all valid JUMPDEST opcodes that do not lie in PUSH payload, and then only allows those jump destinations. With this change, the table of possible jumpdest will be updated when a new page is loaded by scanning only the loaded page and collecting all JUMPDESTs. This will be possible because PUSH cannot span page boundaries.The page remains in memory until the call ends. A 4M block gas limit allows up to 10K pages to be loaded, totaling 80M bytes of code. The user must however take into account that if the gas limit is reduced, a large contract may be prevented from being called.  

The fact that a certain check is performed when code is created (and not when executed) means that contracts that use code as data sets (e.g. a SINE table), may fail to create such data sets. Currently not such failure is possible (only JUMPDEST are tracked). However, this may change in the future, so it’s better to prefix with a PUSH 0x00, JUMP instruction (an infinite loop), so that the remaining parts of the code are inaccessible. Another possibility is to allow "code" to be split into CODE and initialized const DATA sections, as in most executable formats. DATA section can be memory mapped high into the “code” (e.g.  at base offset 0x100000000). 

[RSKIP27]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP27.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
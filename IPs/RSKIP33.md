---
rskip: 33
title: CODEREPLACE opcode
description: 
status: Adopted
purpose: Sec, Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-01-17
---

# CODEREPLACE opcode

|RSKIP          |33           |
| :------------ |:-------------|
|**Title**      |CODEREPLACE opcode |
|**Created**    |17-JAN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sec/Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

# **Abstract**

The RSK platform has two ways to allow contract code upgrades: using DELEGATECALL and creating a VM inside the EVM. However none of the existing ways is generic nor cheap. A simple yet cheap way to allow upgrades is by implementing an opcode to self-replace the code of a contract. This RSKIP introduces the CODEREPLACE opcode.

# **Motivation**

The DAO bug was one of many bugs that affected smart contracts and directly or indirectly was responsible the loss of funds. Most smart contracts should be prepared to be upgraded, at least during a testing period. Currently the only efficient way to upgrade a contract is by creating a proxy contract. A proxy contract forwards all incoming messages to another contract, whose address is modifiable. Only recently RSK/Ethereum VM enabled the creation of generic (library) proxy contracts with the new RETURNDATACOPY and RETURNDATASIZE opcodes. However a proxy contract cannot easily allow bytecode patching of simple bugs without migrating all the contract persistent memory or delegating all persistent memory operations to a third contract. Just replacing the whole code is much simpler and easier to do. Patching in the EVM requires at least three program memory cells to override. In the first cell a PUSH1 opcode is added, then a 1-byte destination address to jump, and in the third a JUMP. A new appended routine can perform the fixed overwritten function and then jump back to the following instruction.

To protect from accidental of malicious code replacements, the CODEREPLACE opcode should be protected in Solidify by guards:

```
contract CodeReplaceTest {
  function updateCode(byte[] newCode) onlyOwner {    	  
    CODEREPLACE(newCode);	
  }
}
```


# **Specification**

A new opcode is added: CODEREPLACE <offset> <newLength>

Arguments are pushed by caller in the order they appear in the description.

The offset is the offset in volatile memory where the new code is located. The length is the size in bytes of the new code.

The gas cost of this opcode is : 

- Fixed: 15000

- REPLACE_DATA: 50 (paid for every code byte replaced)

- CREATE_DATA: 200 (paid for every new code byte)

If the new length lower lower than the old length, then Fixed+newLength*REPLACE_DATA is paid.

If new length is higher than old length, then Fixed+oldLength*REPLACE_DATA+ (newLength-oldLength)*CREATE_DATA is paid.

The gas is consumed immediately when the opcode is executed, but the code is replaced only after the transaction finishes processing. It could be better to replace the code after the block finishes, to prevent interference with future transaction scheduling systems that perform static code analysis to detect static dependencies.

## Future Improvements

And PATCH opcode could be added to replace only parts of the code.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
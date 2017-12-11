# Memory-Mapped Configuration Register

Code: RSKIP51

Author: SDL

Creation Date: 08-15-2017

Revision: 2

Status: Draft

# Abstract

This RSKIP proposes to define a way to extend the EVM without compromising compatibility. A 32-byte memory-mapped configuration register is defined. The meaning of each bit of this register will be established in future RSKIPs.

# Motivation

The RVM (RSK VM) is based on the EVM. RSK intention is to maintain close to full compatibility with Ethereum’s VM. However, RSK has specific needs and its own roadmap for VM improvements. To extend the VM RSK defines new opcodes. However, to provide 100% compatibility with EVM, RSK defines two VM modes, selectable with a specific code header, as defined with RSKIP50. However the current standard EVM compilers are unable to generate code with this special headers, not allow user-defined opcodes in in-line assembly. It would be useful to be able to switch to new modes of operation and access new opcodes without patching the compilers. 

There are several approaches to enable the extended mode: 

* Redefine an existing opcode (e.g. CALL) so that when executed with specific parameters it works as a call into the "operating system" and enables access to internal RSK functions.

* Redefine a special uncommon opcode sequence as an enabler of a new working mode.

* Create a memory-mapped Configuration Register (COREG) that host a bit which enables the extended mode.

The memory mapped register is better because it allows mode changes with low gas cost, and won’t confuse static analyzers not interfere with code optimizers. The position of the COREG in memory is chosen that the probability that buggy code access this register by mistake is low. For instance, the position 0xff...ff is not good because a wrap-around zero of an array access would overwrite the COREG.

# Specification

The COREG is a 32-byte wide register mapped at address 0x80...00. The register can be accessed by MSTORE, MLOAD and MLOAD8. Accesses with MSTORE/MLOAD should be aligned to the COREG starting offset. A mis-aligned read may rise a OOG exception. The exception is not guaranteed as in the future the COREG size may be extended.

The cost of an MLOAD, MLOAD8 or MSTORE access to the COREG is exactly 3.


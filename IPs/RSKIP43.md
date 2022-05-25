---
rskip: 43
title: Sequential Address format
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-06-23
---

# Sequential Address format

|RSKIP          |43           |
| :------------ |:-------------|
|**Title**      |Sequential Address format|
|**Created**    |23-JUN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft | 

# **Abstract**

This RSKIP discusses and defines a new type of RSK address that can be more securely copy-pasted or, although not normally recommended, typed. This RSKIP extends [RSKIP19].

# **Motivation**

Current RSK addresses consist of 40 hexadecimal digits, such as "7ac5496aee77c1ba1f0854206a26dda82a81d6d8". This addresses will be called “raw” addresses. Raw addresses are difficult to type without mistakes, and carry no checksum, so a of a human mistake can cause the transferred funds to be lost forever. Even if addresses should normally be copy-pasted, the fields where the address is pasted could be unintentionally modified by the user, and again transferred funds may be lost. As the system should support the upgrade of address formats, a format prefix is recommended, to avoid depending address lengths to differentiate them. Also it is important to be able to differentiate addresses in testnets from production nets, and addresses from different blockchains.



# **Specification**

See discussion [here](https://github.com/rsksmart/RSKIPs/issues/81).

This specification uses the "S" address kind specifier defined in [RSKIP19].

A sequential address body consist of two parts: a sequential number and a 20-bit nonce. The sequential number (s) is incremented each time a new address is created. The nonce is assigned automatically by the registry as the first three bytes of SHA3(BLOCKHASH || s), zeroing the first 4 MSB bits of the 24-bit number to obtain a 20-bit number. The sequence number encoded in base35H and it is not padded., afterwards a 4 digit base35H encoding of the nonce is added.

Since the nonce and checksum values most of the time will be base35H encoded as 4 digits, both are (right) padded with zeros up to 4 digits and concatenated. 

Sample numbers and their representations:

<table>
  <tr>
    <td>Number</td>
    <td>Representation of sequence</td>
    <td>Representation in checksum/nonce</td>
  </tr>
  <tr>
    <td>0</td>
    <td>0</td>
    <td>0000</td>
  </tr>
  <tr>
    <td>255</td>
    <td>7A</td>
    <td>007A</td>
  </tr>
  <tr>
    <td>65535</td>
    <td>1IHF</td>
    <td>1IHF</td>
  </tr>
  <tr>
    <td>(1 << 20)-1</td>
    <td>ZFYA</td>
    <td>ZFYA</td>
  </tr>
  <tr>
    <td>(1 << 30)-1</td>
    <td>KFIIWS</td>
    <td>---</td>
  </tr>
</table>


## Sample Sequential RSK addresses (with random checksums)

**PS**9ZLDRMH33-B8JJ

**TS**7A3C7YM-ZLDR

## Security Considerations

The use of the BLOCKHASH as a randomization technique works in PoW-based blockchains such as RSK, but it is not suitable for PoS-based blockchains so in case a migration to PoS is needed, a new address format would be needed, or users would need to wait several confirmations after registering theirs addresses in the registry before using them.

[RSKIP19]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP19.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
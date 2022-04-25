---
rskip: 48
title: Informing average free gas per block
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-11-28
---

# Informing average free gas per block

|RSKIP          |48           |
| :------------ |:-------------|
|**Title**      |Informing average free gas per block|
|**Created**    |28-NOV-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This document describes a new method provided by the REMASC contract to inform users of the gas availability in the last block intervals. To optimize queries, the REMASC will maintain the average free gas per block at powers of 2, until 8192. 

# **Discussion**

Pseudocode for each new block B.

1. prev = B.gasLimit-B.gasUsed

2. w = 0

3. while (w<=13)

    1. b0[w] = b1[w]

    2. b1[w] = prev

    3. prev = (b0[w] + b1[w]) /2

    4. valueAtLevel[w] = prev

    5. if (blockNumber & (1<<w) ==0) break

    6.    w++ 


This is a multi-level averaging filter. The properties of this filter MUST be carefully studied. Each value feeded into a filter of level N is an average of level (N-1) therefore the values come pre-smoothed. The only drawback is that the system updates the filter at level N once every 2^N blocks, which means that the data is always delayed. 

For a level of 8192 (approximately one update every day), the value is delayed on average half a day.

One thing one can do is to combine values at different levels at different times, when the blocknumber is not an exact multiple of 2^N.

Eg 

Average at blocknumber 8192+100 is

( Block[8192+100].valueAtLevel[log2(8192)] + 

Block[8192].valueAtLevel[log2(64)]+

Block[8192+64].valueAtLevel[log2(32)]+

Block[8192+96].valueAtLevel[log2(4)] ) / 4

This computation gives more weight to the last samples. To get a better weight, this is suggested:

( Block[8192+100].valueAtLevel[log2(8192)] *8192+ 

Block[8192].valueAtLevel[log2(64)] * 64+

Block[8192+64].valueAtLevel[log2(32)] *32+

Block[8192+96].valueAtLevel[log2(4)] * 4) / (8192+64+32+4)

This gives an estimation of the last(8192+64+32+4) blocks.

To obtain an estimation of an exact number of blocks, the bit decomposition of the blocknumber is required.

E.g. At point 8292:

    (4 block average at blocknum 104  ) *4 +

(8 block average at blocknum 112 ) * 8 +

(16 block average at blocknum 128 )* 16+

(128 block average at blocknum 256) * 128+

(256 block average at blocknum 512)*256+

(512 block average at blocknum 1024)*512+

(1024 block average at blocknum 2048)*1024+

(2048 block average at blocknum 4096)*2048+

    (4096 block average at blocknum 8192 )*4096+

     ( 64 block average at blocknum (8192+64) ) *64+

     ( 32 block average at blocknum (8192+96) ) *32+

     ( 4 block average at blocknum (8192+96+4) )* 4

(4+8+16+128+256+512+1024+2048+4096+64+32+4)=8192

Addition way is:

* Let H be the high limit

* Let L be the low limit

* Let a = trunc(log2(H-L))

* Let p = 2^a

* Let Ldif = (a-L)

* Let Ldec = accum_averages(L..a)
bit decomposition and accumulation of averages of Ldif in the range L..a

* Let Hdif = (H-2*a)

* Let Hdec = accum_averages(a..H)
bit decomposition and accumulation of averages of Hdif in the range a..H

* Total = a*accum_averages(a..2*a)+ Hdec*Hdif + Ldec*Ldif 

Subtraction way is:

* Let H be the high limit

* Let L be the low limit

* Let a = ceil(log2(H-L))

* Let p = 2^a

* Let Ldif = (L-a)

* Let Ldec = accum_averages(a..L)

* Let Hdif = (H-2*a)

* Let Hdec = accum_averages(L..H)

* Total = a*accum_averages(2*a)+ Hdec*Hdif - Ldec*Ldif 

The idea is that one can compute the averages from the lower limit to a higher power of two or to a lower power of two. If it’s higher, we add. If it’s lower we subtract. 

The advantage of adding is that it uses a largest power (a) smaller than when subtracting. But the actual addition cost depends on the bit decompositions: if (L-a) has low hamming distance, then addition is better. If (a-L) has low hamming distance, then subtraction is better.

# **Specification**

## References:

[https://github.com/raiden-network/raiden/issues/383](https://github.com/raiden-network/raiden/issues/383)

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
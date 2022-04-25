---
rskip: 109
title:  Lower Storage Gas Costs for Shorter Keys 
description: 
status: Draft
purpose: Sca, Usa
author: SDL (@sergiodemianlerner)
layer: 
complexity: 2
created: 2019
---
#  Lower Storage Gas Costs for Shorter Keys

| RSKIP          | 109                                      |
| :------------- | :--------------------------------------- |
| **Title**      | Lower Storage Gas Costs for Shorter Keys |
| **Created**    | 2019                                     |
| **Author**     | SDL                                      |
| **Layer**      | Sca, Usa                                 |
| **Complexity** | 2                                        |
| **Status**     | Draft                                    |

# Abstract

The RSKIP108 specifies a new method to compress storage cell keys for the Unitrie. A new gas cost scheme is required to incentivize users to use shorter keys, and therefore reduce full node requirements. 



## Specification

 

The new gas cost for creating a new storage cell is changed from 20k to:

​	newCellCost = 5000 + size_cost
​	size_cost = 230*(keyBytes+dataBytes)

This is the SET cost (zero to non-zero). As 0x00 is trimmed to 0x00, the minimum value for keyBytes is 1, and the minimum value for dataBytes is 1.

The maximum SET cost is: 19720

The minimum SET cost is: 5460

When a cell value is changed, the cost of the keyBytes stay constant, but the cost of dataBytes can change. Therefore both the REFUND value and the RESET (non-zero to non-zero) cost must be modified.

Any change attempt that does not modify the value has a cost of 200 (similar to SLOAD).  For instance zero to zero, or A to A. Will call this new cost the UNCHANGE cost.

The REFUND value is computed as SET cost minus 5000. (Maximum 14720, minimum 460).

The CLEAR cost (from non-zero to zero) is still computed as 5000.

The RESET cost now only accounts for value changes from a non-zero value to a different non-zero value. This has a base cost of 5000 plus the difference between new cell costs and old cell cost (maximum 12130, minimum 5000).



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



```

```

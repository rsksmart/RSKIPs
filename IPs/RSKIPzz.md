#  Unitrie Node Hibernation with Miner-originated Awakening

| RSKIP          | zzz                                                   |
| :------------- | :---------------------------------------------------- |
| **Title**      | Unitrie Node Hibernation with Miner-originated Awaken |
| **Created**    | 2019                                                  |
| **Author**     | SDL                                                   |
| **Layer**      | Sec, Sca                                              |
| **Complexity** | 1                                                     |
| **Status**     | Draft                                                 |

# Abstract

RSKIP61 presents the cache-oriented storage rent without hibernation. All previous proposal focused on hibernation by immediate or delayed deletion of data and the decentralized awakening based on providing the missing data in transactions. But RSKP61 suggests a new model where data is not hibernated by modifying state information but hibernated at a higher (softer) consensus level: nodes won't accept the use of trie nodes whose unpaid rent is higher than a predefined threshold, unless the data in the nodes is transferred along the block. This proposal focuses on the hibernation and awakening methods that minimizes the overhead.

# Discussion

# Specification

A new field is added to Unitrie nodes lastRentPaidTime. 







# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



```

```

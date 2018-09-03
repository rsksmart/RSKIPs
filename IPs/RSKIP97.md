#  **Removal of difficutly drop**  

| RSKIP          | 97                             |
| :------------- | :----------------------------- |
| **Title**      | Difficulty drop                |
| **Created**    | 2018                           |
| **Author**     | LS                             |
| **Purpose**    | Sec                            |
| **Layer**      | Core                           |
| **Complexity** | 1                              |
| **Status**     | Draft                          |


# Abstract

This RSKIP proposes to remove the difficulty drop that blockchain has when a new block isn't mined after ten minutes. 


# Motivation

The difficulty drop was added on the beggining of the blockchain as a safeguard when not enough hashing power was present on the system.
Since the hashing power is significant enough the drop can be removed on the next hard fork.


## Specification

In the current system the blockchain accepts a difficulty drop on a block after ten minutes with no newer blocks. The change proposed in the hard fork removes the drop and uses only the expected ways to adjust blocks difficulty.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



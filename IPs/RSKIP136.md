# State sync

|RSKIP          |136           |
| :------------ |:-------------|
|**Title**      |State sync|
|**Created**    |2019 |
|**Author**     |LS & IM |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This document describes a new mechanism to synchronize RSK nodes. This mechanism would allow to synchronize the RSK blockchain by downloading an state and other necessary components to have a syncrhonized node to a current state.


# Motivation

This new sync mechanism sustains on the idea that a blockchain can start processing blocks whenever it gets to download the state that represents to the current best block of the blockchain. In order to validate the new downloaded state, we need to verfiy the proof of work from the current block, so all  the previous headers are required too. Finally in the case of the RSK blockchain we need to retrieve the information about 4000 blocks behind to process the next block and be able to pay the fees in the REMASC contract.

We'll call this new sync mechanism STATE-SYNC from now on. The goal for this new algorithm is to reach a synchronized blockchain at a SYNC-BLOCK but without paying the cost of downloading and executing all the missing blocks. To achieve this __data__ is required:

* The current state at SYNC-BLOCK
* The previous 4000 blocks from behind the SYNC-BLOCK, to allow REMASC contract to be executed on the next block and subsequents
* All the previous headers in order to validate the proof of work for the SYNC-BLOCK

# Specification



# Rationale


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
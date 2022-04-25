---
rskip: 98
title: Deactivation of the federated fallback system for block production 
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2018-07-
---
# Deactivation of the federated fallback system for block production 

|RSKIP          |98           |
| :------------ |:-------------|
|**Title**      |Deactivation of the federated fallback system for block production |
|**Created**    |JULY-2018 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes the forever deactivation of the fallback system that allows a federation to produce blocks instead of the miners.


# **Motivation**

The RSK was launched with a fallback block production system that activates if the hasrate decreases below a pre-established threshold.
This system was added to protect RSK from easy reorg attacks during the ramp up of the hashing power. 
Having a federated block production backup system may transmit a wrong message to the RSK community that sometime in the future the RSK community may switch back to this system, lossing the important value of decentralization shared by the RSK community.
This month (July, 2018) RSK achieved enough hashrate that it is highly improbable that the back up system will ever be needed.
Therefore we propose that the system is forever deactivated in the next consensus upgrade.

# **Specification**

If block number >= N (TBD), only blocks presenting a valid merge-mined proof-of-work will be accepted.

# Backwards Compatibility

This change is a hard-fork and therefore all nodes (light and full) must be updated. 

# Test Cases

TBD

## Security Considerations

If RSK hashrate drops below 20% of Bitcoin's hashrate, users would need to wait many more block confirmations to reach the same level of security and the Bitcoin network would need to be actively monitored for malicious forks.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
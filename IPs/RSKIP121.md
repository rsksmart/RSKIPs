# Orchid Network Upgrade

|RSKIP          |121           |
| :------------ |:-------------|
|**Title**      |Event log for Bridge lock operations |
|**Created**    |26-APR-19 |
|**Author**     |PP |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes to emit and event log on successful lock operations (BTC=>RSK) to gain visibility and transparence.

# **Motivation**
RSK community has proposed to have an event for lock operations so they can trigger actions when they detect one of their intrest.

# **Specification**

Topic:"lock_btc_topic"
Data: [Btc Tx Hash, Btc Tx Serialized]

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

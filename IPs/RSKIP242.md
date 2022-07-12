---
rskip: 242
title: Proxy code Incentive
description: 
status: Draft
purpose: Sca, Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-05-15
---
# Proxy Code Incentive

|RSKIP          |242           |
| :------------ |:-------------|
|**Title**      |Proxy code Incentive|
|**Created**    |15-MAY-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca, Fair |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes the modification of the cost to create a contract. Currently the contract creation cost depends linearly on the number of bytes installed. However, with the Unitrie, when installing 44 or less bytes of code, those bytes are stored in the trie node (without a hash digest indirection). Installing more than 44 bytes costs approximately 128 additional bytes in overhead, plus calling a contract of that size requires accessing to an additional key-value pair in the database. We propose that installing 44 or less bytes has a cost of only 30 gas/byte, while installing more costs as if 128 additional bytes had been installed. Also creating a contract costs 53K, but this cost is unjustified. We propose to reduce it to 25K. This benefits proxy contracts, commonly used in multi-signature wallets.

# **Motivation**

Financial inclusion will require the deployment of millions of multi-signature wallets or smart wallets. Those wallets will probably be proxies to a few library contracts, which will be audited and tested. Creating wallets should be as cheap as possible. RSKIP allows deployment cost of smart-wallets to be reduced from 62K (which is the minimum current cost), to 26.3K. 


# **Specification**

Starting from an activation block (TBD), two cost changes occur.

First, fixed gas cost for contract creation (both in transaction, CREATE and CREATE2) is reduced from 53K to 25K.

Second. the cost of installing the code (bytes returned by the contract initialization code) now obey the following formula:

```
if (codeLength <= 44)
   cost = codeLength*30;
   else
   cost = (codeLength+128)*200
```




# Rationale

With the proposed changes, installing a long contract will cost slightly less than before.

If the trie node data embedding threshold is changed in the future, this change should be reflected in the cost of contract creation.

The cost of a single byte in the state (for example, in contract storage) is about  ~80 gas/byte, that cost includes ~40% of node size and indexing overhead. We choose the cost 30 because there is no overhead involved and we slightly subsidize the use of proxies because they reduce access time in calls.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

## Security Considerations

No new denial of service or resource abuse risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

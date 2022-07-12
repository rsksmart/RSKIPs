---
rskip: 132
title: Bridge ReceiveHeaders Gas Cost increase
description: 
status: Adopted
purpose: Fair
author: JD (@josedahlquist), SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2019-07-11
---
# Bridge ReceiveHeaders Gas Cost increase

|RSKIP          |132           |
| :------------ |:-------------|
|**Title**      |Bridge ReceiveHeaders Gas Cost increase|
|**Created**    |11-JULY-2019 |
|**Author**     |JD & SDL |
|**Purpose**    |Fair |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

# **Abstract**

This RSKIP proposes an the re-pricing of the method ReceiveHeaders in the Bridge contract.
Currently the cost is fixed at 25K gas and a variable cost of 1650 per additional header. We've performed performance tests which show that this method is underpriced, both in ther fixed and variable components.
	

# **Motivation**

Make the RSK platform fair in terms of gas costs and prevent potential DoS attacks.

# **Specification**

The fixed gas component of the ReceiveHeaders call is increased from 25K to 66K. The variable gas component (affecting additional headers except the first) is increased from 1650 to 3500.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

# Test Cases

TBD

## Security Considerations

This change improves the security of the network by preventing potential DoS attacks based on repeated ReceiveHeaders calls.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
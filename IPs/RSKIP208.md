---
rskip: 208
title: checkEnvironment Precompile method
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-01
---
# checkEnvironment Precompile method


|RSKIP          | 208 |
| :------------ |:-------------|
|**Title**      |checkEnvironment Precompile method|
|**Created**    |JAN-2021 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

This RSKIP proposes to add a method `checkEnvironment()` to a new `Environment`  pre-compile contract. The new method receives as argument a list of fields that represents the minimum environment requirements to execute a method and it checks that the requirements are satisfied. This RSKIP provides the same functionality of [RKIP203](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP203.md), but in a way which allows the future addition of additional environment variables, such as rent gas limit, for storage rent accounting.

## Motivation

The motivation is the one described in [RSKIP203](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP203.md) plus we add the requirement that contracts that forward contracts calls do not become suddenly vulnerable if a network upgrade adds another hidden counter to the environment.

This RSKIP proposes to create a new `Environment` pre-compile contract with a method to check if the minimal environment requirements given as the `env` argument are satisfied. The argument contains a list of fields in RLP format. This RSKIP defines that the first argument of the list is the stack depth. The gas limit is left outside `env` because the caller must provide the value externally to `CALL`. The remaining fields are undefined and are ignored. It is expected that future network upgrades assume default values for missing fields. Users and relayers will negotiate a content for `env`. Smart contract do not need to interpret the content of `env`, and can simply pass it to the `checkEnvironment()` method.

## Specification

The method with signature ` checkEnvironment(bytes env) returns (uint32)` is added to a new pre-compiled contract named `Environment`, created at address `0x0000000000000000000000000000000001000011`. The call returns 0 on success. On failure it returns the position of the item in `env` that is not satisfied by the environment (1==first item). 

The argument `env` corresponds to a list of items, each coded in RLP. It is not an RLP list. 

The first item is the minimum stack depth. If the argument length is zero, the default value assumed for the stack depth item is 200.

Additional arguments are ignored, even if they decode to invalid RLP data.

The gas cost specific to this method call is 40.


## Backwards Compatibility

This is a hard-forking change, but it shouldn't affect existing smart contracts, because no contract should call a precompile contract with an nonexistent method selector.


## Test Cases

TBD

## Implementation

TBA

## Security Considerations

None identified.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

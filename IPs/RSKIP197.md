---
rskip: 197
title: Fix Precompile Calls Not Conforming With CALL Semantics
description: 
status: Accepted
purpose: Usa
author: FJ (@fedejinich)
layer: Core
complexity: 2
created: 2020-12-15
---
# Fix Precompile Calls Not Conforming With CALL Semantics

|RSKIP          |197           |
| :------------ |:-------------|
|**Title**      | Fix Precompile Calls Not Conforming With CALL Semantics |
|**Created**    |15-12-2020 |
|**Author**     |FJ |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Accepted |

## Motivation

Before the Iris hard-fork, a precompiled contract could not revert, but only could raise an OOG, consuming all gas passed to the CALL.

## Specification

When `blockNumber >= IRIS_HARD_FORK`: 
- Make every precompiled contract call respect the CALL semantics.

## References

[1] RSKIP197 Implementation https://github.com/rsksmart/rskj/pull/1392

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

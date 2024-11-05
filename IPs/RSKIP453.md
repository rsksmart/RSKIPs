---
rskip: 453
title: Prevent address creation on failed CREATE/CREATE2 operations
description: 
status: Draft
purpose:    
author: AS
layer: Core
complexity: 2
created: 2024/10/09
---
# Prevent address creation on failed CREATE/CREATE2 operations


|RSKIP          | 453 |
| :------------ |:-------------|
|**Title**      |Prevent Address Creation on Failed CREATE/CREATE2 Operations|
|**Created**    |OCT-2024 |
|**Author**     |AS |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


## Abstract

This RSKIP proposes aligning Rootstock (RSK) contract deployment behavior with Ethereum by failing contract creation. This change ensures no empty contract accounts are left in the state, mirroring Ethereum's behavior.

## Motivation

RSK currently leaves empty contract accounts in the state when contract creation fails, differing from Ethereum. Aligning with Ethereum will improve compatibility, security, and user experience in the ecosystem.

## Specification

1. **Contract Size Limit Exceeded**:  
   When deploying contracts via CREATE/CREATE2 opcodes, if the code size exceeds the max limit, the contract creation should fail, and no empty contract account should remain in the state.

2. **Insufficient Gas**:  
   If gas runs out during contract creation (CREATE/CREATE2), the contract creation should fail, and no empty contract account should be created.

3. **No Gas Refund**:  
   In both cases (code size or gas limit), all remaining gas should be consumed, with no refunds provided.   

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
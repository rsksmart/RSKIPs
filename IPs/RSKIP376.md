---
rskip: 376
title: Set version 2 to PowPeg migration transactions
created: 12-DEC-23
author: MI
purpose: Usa
layer: Core 
complexity: 1
status: Draft
description: 
---

|RSKIP          |376           |
| :------------ |:-------------|
|**Title**      |Set version 2 to PowPeg migration transactions |
|**Created**    |12-DEC-23 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

Since the implementation of RSKIP201 [1], peg-out transactions are created with version number 2 following the specification in BIP68 [2]. This RSKIP proposes that also PowPeg migration transactions are created with version 2 to be consistent with peg-out transactions.

## Motivation

Peg-out and migration transactions should be treatead equally by the Bridge and should both have the same format.

## Specification

When a PowPeg composition change is executed and the migration process starts in the Bridge, migration transactions sending funds from the retiring to the newly active PowPeg address should be created using version 2.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP201](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md)

[2] [BIP68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

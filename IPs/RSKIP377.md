---
rskip: 377
title: Store the last retired federation **standard** P2SH script
created: 17-MAY-23
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Adopted
description: 
---

|RSKIP          |377           |
| :------------ |:-------------|
|**Title**      |Store the last retired federation **standard** P2SH script |
|**Created**    |17-MAY-23 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes a change on the value stored in the Bridge as the last retired federation P2SH script. Instead of building the P2SH script from the complete redeem script of the federation, the proposal is to extract the **standard** redeem script (i.e. remove the emergency multisig specific parts [1]) and build the P2SH script to be stored from it.

## Motivation

The Bridge contract in Rootstock is constantly evolving and adding new features. Some of these new features add the possibility of creating new PowPeg addresses by adding different op codes to the current PowPeg redeem script without changing the signers' composition. See RSKIP 201 [1] and RSKIP 176 [2] as an example of this use case.

These different use cases show that there could be multiple different redeem scripts created for the same PowPeg composition. Different transactions could then have different PowPeg redeem scripts all belonging to the same PowPeg composition. The last retired federation P2SH script is stored so that it is possible to identify transactions that were created by the PowPeg conformation that existed until the latest PowPeg change.

To simplify identifying these transactions with distinct redeem scripts all belonging to the same PowPeg structure, this RSKIP proposes to store as the last retired federation P2SH script a P2SH script built only from the **standard** redeem script.

Standard redeem script format:
```
OP_M
PUBKEYS...N
OP_N
OP_CHECKMULTISIG
```

## Specification

Update the current implementation of RSKIP186 [3]. When a PowPeg change occurs extract the standard redeem script from the current PowPeg redeem script, create a P2SH script from it and store it under `lastRetiredFedP2SHScript` storage key.


## References

[1] [RSKIP 201](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md)

[2] [RSKIP 176](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP176.md)

[3] [RSKIP 186](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP186.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

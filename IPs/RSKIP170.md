---
rskip: 170
title: Peg-in to any address
description: 
status: Draft
purpose: Usa
author: MI <marcos@iovlabs.org>
layer: Core
complexity: 2
created: 2020-09-01
---
# Peg-in to any address

|RSKIP          |170           |
| :------------ |:-------------|
|**Title**      |Peg-in to any address |
|**Created**    |01-SEP-20 |
|**Author**     |MI |
|**Purpose**    |USa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a way for users to transfer BTC to RSK, indicating the RSK address in which they want their RBTCs transferred to and a BTC refund address in case the peg-in can not be completed for some reason.

## Motivation

Allowing users to peg-in BTC to any address in RSK, EOA or contract.

## Specification

This RSKIP proposes an extension to the current peg-in workflow. By including an output with an OP_RETURN op code and certain payload in the transaction sent to the federation to perform the peg-in, the user can determine the RSK address where the RBTCs will be transferred to and a BTC refund address. 

The BTC transaction sent to the federation must contain 
 one output with value 0 and OP_RETURN op code followed by the following data:
- `52534b54` as prefix indicating the output is for RSK. This values stands for `RSKT` (for RSK  transfer) in ascii and hex encoded. (4 bytes)
- Protocol version in hexa (1 byte). Currently **01**
- A valid RSK address (20 bytes)
- [Optional] A valid BTC refund address in the following format:
    - Address type (1 byte)
        - 01 for P2PKH
        - 02 for P2SH
    - Hash needed to build refund address (20 bytes)
        - For P2PKH type address: public key hash 
        - For P2SH type address: script hash

### Example output
```
{
      "value": 0.00000000,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_RETURN 52534b5400010e537aad84447a2c2a7590d5f2665ef5cf9b667a014f4c767a2d308eebb3f0f1247f9163c896e0b7d2",
        "hex": "6a2b52534b54010e537aad84447a2c2a7590d5f2665ef5cf9b667a014f4c767a2d308eebb3f0f1247f9163c896e0b7d2",
        "type": "nulldata"
      }
    }
```
Payload included: `52534b54010e537aad84447a2c2a7590d5f2665ef5cf9b667a014f4c767a2d308eebb3f0f1247f9163c896e0b7d2`

- First 4 bytes correspond to the prefix that indicates the output has peg-in instructions for RSK (`RSKT` in hexa): `52534b54` 
- Next 1 byte that correspond to the version number in hexa: `01`
- Next 20 bytes indicate rsk destination address: `0e537aad84447a2c2a7590d5f2665ef5cf9b667a`
- Next 1 byte indicating btc refund address type, in this case P2PKH: `01`
- Finally 20 bytes indicating the public key hash that will be used to get the btc refund address: `4f4c767a2d308eebb3f0f1247f9163c896e0b7d2`

### New peg-in flow

When a BTC transaction is sent to the Bridge, the following steps take place:

1. Blockchain validations as performed in the existing flow (e.g. amount of confirmations, PMT valid)
2. Parse transaction to get RSK destination address and BTC refund address
   1. If no OP_RETURN data is present, follow the current flow. Get tx sender to use as BTC refund address and derive RSK destination address from BTC public key.
   2. If OP_RETURN data is present, get RSK destination address from the payload and the BTC refund address if present. In case there is no BTC refund address included in the payload, get the tx sender.

### Notes
- No restriction is added to the maximum length in bytes of the transaction.
- No restriction is added to the total amount of outputs the transaction can have.
- There is no limit to the amount of outputs with OP_RETURN the peg-in transaction can have. But there can be only one with `RSKT` prefix indicating peg-in instructions. Transactions with more than one OP_RETURN outputs containing `RSKT` prefix will be rejected, following the existing protocol for refunds.
- The output with OP_RETURN containing the peg-in instructions does not have to be in any particular order among the other outputs.

## Rationale

[RSKIP41](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md) includes optional fields with the purpose of executing a smart contract in RSK, in case the RSK destination address is a contract instead of an account. In this version of the protocol, the possibility of executing a smart contract from the peg-in transaction is not contemplated. 

If the RSK destination address set in the payload is a contract then the locked funds will be transferred to fund this contract. It is **not** required that the contract has a payable fallback function to receive the funds. In case the recipient contract has a payable fallback function with some implementation, the code will not be executed. Only the transfer will be made.

## References

[1] https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md

[2] https://bitcoinfoundation.org/files/bitcoin/core-development-update-5/

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

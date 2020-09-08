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

This RSKIP proposes a way for users to be able to lock funds from BTC, indicating the RSK address in which they want their RBTCs transferred to and a BTC refund address in case the peg-in can not be completed for some reason.

## Motivation

Allowing users to lock funds to any address in RSK.

## Specification

The BTC transaction sent to the federation must contain at most one output with value 0 and OP_RETURN op code followed by the following data:
- Protocol version (2 bytes). Currently **1**
- A valid RSK address (20 bytes)
- [Optional] A valid BTC refund address in the following format:
    - Address type (1 byte)
        - 1 for P2PKH
        - 2 for P2SH
    - Hash needed to build refund address (20 bytes)
        - For P2PKH type address: public key hash 
        - For P2SH type address: script hash

### Example output
```
{
      "value": 0.00000000,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_RETURN 00010e537aad84447a2c2a7590d5f2665ef5cf9b667a014f4c767a2d308eebb3f0f1247f9163c896e0b7d2",
        "hex": "6a2b00010e537aad84447a2c2a7590d5f2665ef5cf9b667a014f4c767a2d308eebb3f0f1247f9163c896e0b7d2",
        "type": "nulldata"
      }
    }
```
Payload included: `00010e537aad84447a2c2a7590d5f2665ef5cf9b667a014f4c767a2d308eebb3f0f1247f9163c896e0b7d2`

- First 2 bytes correspond to the version number: `0001`
- Next 20 bytes indicate rsk destination address: `0e537aad84447a2c2a7590d5f2665ef5cf9b667a`
- Next 1 byte indicating btc refund address type, in this case P2PKH: `01`
- Finally 20 bytes indicating the public key hash that will be used to get the btc refund address: `4f4c767a2d308eebb3f0f1247f9163c896e0b7d2`

### Notes
- Only one output with OP_RETURN op code is allowed in the pegin transaction. Transactions with more than one OP_RETURN outputs will be rejected.
- No restriction is added to the maximum length in bytes of the transaction.
- There is no limit to the amount of outputs the transaction can have, as long as there is no more than one with OP_RETURN op code.
- The output with OP_RETURN does not have to be in any particular order among the other outputs.

## Rationale

[RSKIP41](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md) includes optional fields with the purpose of executing a smart contract in RSK, in case the RSK destination address is a contract instead of an account. In this version of the protocol, the possibility of executing a smart contract from the pegin transaction is not contemplated. If the RSK destination address set in the payload is a contract wallet then the funds locked will be transferred to fund this contract, if the contract is not wallet then those funds will be lost.

## References

[1] https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md

[2] https://bitcoinfoundation.org/files/bitcoin/core-development-update-5/

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

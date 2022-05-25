---
rskip: 60
title: Checksum Address Encoding 
description: 
status: Adopted
purpose: ST
author: JL (@julianlen), IO (@ilan)
layer: Net
complexity: 1
created: 2018-06-25
---
# Checksum Address Encoding

|RSKIP          |60           |
| :------------ |:-------------|
|**Title**      |Checksum Address Encoding |
|**Created**    |25-JUN-2018 |
|**Author**     |JL,IO |
|**Purpose**    |ST |
|**Layer**      |Net |
|**Complexity** |1 |
|**Status**     |Adopted |

# **Motivation**

- Avoid typing confusion in addresses.
- Differentiate addresses of different networks.

# **Abstract**

Addresses can be validated using an injective function that makes capital letters redundant.
RSKIP-0060 **describes an address checksum mechanism** that can be implemented in any network based on [EIP-55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md).

# **Specification**
In Javascript:
```javascript
function toChecksumAddress(address, chainId = null) {
    const strip_address = stripHexPrefix(address).toLowerCase()
    const prefix = chainId != null ? (chainId.toString() + '0x') : ''
    const keccak_hash = keccak(prefix + strip_address).toString('hex')
    let output = '0x'

    for (let i = 0; i < strip_address.length; i++)
        output += parseInt(keccak_hash[i], 16) >= 8 ?
            strip_address[i].toUpperCase() :
            strip_address[i]

    return output
}
```

Adds the chain id as a prefix. Converts the address to hexadecimal. Calculates [keccak](https://csrc.nist.gov/csrc/media/publications/fips/202/final/documents/fips_202_draft.pdf) with the prefixed address. Prints `i` digit if it's a number, otherwise checks if `i` byte of the hash of the keccak. If it's grater than 8 prints uppercase, otherwise lowercase.


- `chainId` unique values defined in [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md).

- This algorithm is compatible with [EIP-55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md). This can be achieved using `null prexix`.

# **Implementation**

| Project | Languaje | Reference |
| - | - | - |
| [RSK Labs - rskjs-util](https://github.com/RSKSmart/rskjs-util) | JavaScript | [Code](https://github.com/rsksmart/rskjs-util/blob/5f284ee833ce8d958216804107d0bb90b3feb52e/index.js#L30) |
| [Trezor - trezor-core](https://github.com/trezor/trezor-core) | Python | [Code](https://github.com/trezor/trezor-core/blob/270bf732121d004a4cd1ab129adaccf7346ff1db/src/apps/ethereum/get_address.py#L32) |
| [Trezor - trezor-crypto](https://github.com/trezor/trezor-crypto) | C | [Code](https://github.com/trezor/trezor-crypto/blob/4153e662b60a0d83c1be15150f18483a37e9092c/address.c#L62) |

# **Rationale**
 
Benefit:
- Allows implementation in any network. Can distinguish between testnets and mainnets.
- Backwards compatiblity with many hex parsers that accept mixed case, allowing it to be easily introduced over time.
- Compatibility with [EIP-55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md) checksum implementation.

# **Test cases**

```
Valid with network id: 30
    0x5aaEB6053f3e94c9b9a09f33669435E7ef1bEAeD
    0xFb6916095cA1Df60bb79ce92cE3EA74c37c5d359
    0xDBF03B407c01E7CD3cBea99509D93F8Dddc8C6FB
    0xD1220A0Cf47c7B9BE7a2e6ba89F429762E7B9adB

Valid with network id: 31
    0x5aAeb6053F3e94c9b9A09F33669435E7EF1BEaEd
    0xFb6916095CA1dF60bb79CE92ce3Ea74C37c5D359
    0xdbF03B407C01E7cd3cbEa99509D93f8dDDc8C6fB
    0xd1220a0CF47c7B9Be7A2E6Ba89f429762E7b9adB

Valid with network id: None
    0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed
    0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359
    0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB
    0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb

Invalid with network id: 1234
    0x5aaeb6053F3E94C9B9A09f33669435E7Ef1BeAed
    0xfb6916095ca1df60bb79Ce92ce3ea74c37c5d359
    0xDBF03B407C01E7CD3CBEA99509D93f8DDDC8C6FB
    0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb
```
# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


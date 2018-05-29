# Checksum Address Encoding

|RSKIP          |60           |
| :------------ |:-------------|
|**Title**      |Checksum Address Encoding |
|**Created**    |25-JUN-16 |
|**Author**     |JL,IO |
|**Purpose**    |ST |
|**Layer**      |Net |
|**Complexity** |1 |
|**Status**     |Draft* |

## Motivation

- Avoid typing confusion in adresses.
- Differentiate addresses of different networks.

## Abstract

Addresses can be validated using an injective function that makes capital letters redundant.
RSKIP-0060 **describes an address checksum mechanism** that can be implemented in any network based on [EIP-55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md).

## Specification
In Javascript:
```javascript
function toChecksumAddress(address, chainId = null) {
    const strip_address = stripHexPrefix(address).toLowerCase()
    const prefix = chainId != null ? (chainId.toString() + '|') : ''
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

## Implementation

`chainId` unique values defined in [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md).

This algorithm is compatible with [EIP-55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md). This can be achieved using `chainId = null`.

## Rationale
 
Benefit:
- Allows implementation in any network. Can distinguish between testnets and mainnets.
- Backwards compatiblity with many hex parsers that accept mixed case, allowing it to be easily introduced over time.
- Compatibility with [EIP-55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md) checksum implementation.

## Test cases

```
Valid with network id: 30
    0x5AaEb6053f3e94C9B9A09f33669435e7Ef1BeaeD
    0xFb6916095ca1dF60BB79Ce92cE3EA74C37C5D359
    0xdbf03b407C01e7cd3CbEA99509D93F8Dddc8c6fB
    0xD1220A0cF47C7B9bE7a2E6ba89F429762e7b9aDB

Valid with network id: 31
    0x5AAEb6053f3E94c9B9A09f33669435e7EF1BeaeD
    0xfB6916095CA1Df60bb79ce92CE3Ea74c37C5D359
    0xDBF03B407C01E7Cd3cBEa99509d93f8DddC8C6Fb
    0xd1220a0cf47C7b9be7A2e6BA89f429762e7b9AdB

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

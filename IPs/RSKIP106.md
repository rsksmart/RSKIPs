---
rskip: 106
title: Precompiled contract for HDWallet utility functions
description: 
status: Adopted
purpose: Usa
author: AM <amendelzon@iovlabs.org>
layer: Core
complexity: 1
created: 2019-02-01
---
# Precompiled contract for HDWallet utility functions

|RSKIP          |106 |
| :------------ |:------------- |
|**Title**      |Precompiled contract for HDWallet utility functions |
|**Created**    |01-FEB-19 |
|**Author**     |AM |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

A new precompiled contract (i.e., native, hardwired onto the RSK consensus layer) is introduced. This contract contains a few complex HDWallet-related operations that help leverage some of the work that smart contracts need to repeatedly perform.

## Motivation

With the arising need of making ICOs on top of the RSK network, a need for different token and/or token-emitting contracts to perform certain complex operations on a regular basis is in place. If written in solidity (or assembly WOLOG) and executed on the RVM, each of these functions would be expensive - and even more expensive to write and test every time. Furthermore, different implementations can lead to errors potentially protruding to loss of funds. A well-written, tested and precompiled function set that covers these operations will leverage the complexity on the smart contracts' side while at the same time enforcing a certain level of security by implementing these as part of the RSK consensus.

## Specification

A new precompiled contract is to be accessible at the `0x0000000000000000000000000000000001000009` address. It will make usage of `HDWallet utilities` to derive BTC addresses, and will implement the following functions (signatures and return values are as follows):

- `toBase58Check(bytes, uint8) returns (string)`
- `deriveExtendedPublicKey(string xpub, string path) returns (string)`
- `extractPublicKeyFromExtendedPublicKey(string xpub) returns (bytes)`
- `getMultisigScriptHash(uint8 minimumSignatures, bytes[]) returns (bytes)`

## Error handling

In case of error, each of these methods behaves as if a solidity `assert` statement was being used. That is, contract state is reverted and all the gas is consumed.

### toBase58Check

The method `toBase58Check(bytes hash, uint8 version) returns (string)` takes as input a 20-byte hash and a version (see ref #2 for possible version values) and returns a base58Check encoded string of the concatenation of `version` and `hash`.

#### Validations

- `hash` must be exactly 20 bytes long.

#### Gas cost

This method has a fixed cost of 8,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

Given a public key, we want to generate a Bitcoin P2PKH address for testnet. We have the compressed key (for uncompressed keys, compression would be needed first) `02e6930d0659a24df1cc0061203d7845ca2e0c6b96bc1ad16e44c1cef24be92de8`. Its `hash160` is the result of applying `ripemd160(sha256(bytes))` to it. That yields `0d3bf5f30dda7584645546079318e97f0e1d044f`, which is the `hash` parameter we need for our function. For testnet, we use version `111`. Therefore, we would have that:

```
toBase58Check('0x0d3bf5f30dda7584645546079318e97f0e1d044f', 111) => 'mgivuh9jErcGdRr81cJ3A7YfgbJV7WNyZV'
```

### deriveExtendedPublicKey

The method `deriveExtendedPublicKey(string xpub, string path) returns (string)` takes as input an extended public key encoded in base58Check (as described in ref #4), an HD derivation path - a string of the form

```
S ::= n | n/S
```

with `n` an unsigned integer - also as described in #4 - and produces a base58Check encoded string corresponding to the extended public key obtained by deriving the given extended public key with the given path.

#### Validations

- `xpub` must be a valid base58check-encoded extended public key (either for mainnet or testnet).
- `path` must be a valid path according to the BNF definition given above.

#### Gas cost

This method has a fixed cost of 55,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

Given the base58Check-encoded extended public key `tpubD6NzVbkrYhZ4YHQqwWz3Tm1ESZ9AidobeyLG4mEezB6hN8gFFWrcjczyF77Lw3HEs6Rjd2R11BEJ8Y9ptfxx9DFknkdujp58mFMx9H5dc1r`, and the derivation path `2/3/4`, we would have that:

```
deriveExtendedPublicKey('tpubD6NzVbkrYhZ4YHQqwWz3Tm1ESZ9AidobeyLG4mEezB6hN8gFFWrcjczyF77Lw3HEs6Rjd2R11BEJ8Y9ptfxx9DFknkdujp58mFMx9H5dc1r', '2/3/4') => 'tpubDCGMkPKredy7oh6zw8f4ExWFdTgQCrAHToF1ytny3gbVy9GkUNK2Nqh7NbKbh8dkd5VtjUiLJPkbEkeg29NVHwxYwzHJFt9SazGLZrrU4Y4'
```

### extractPublicKeyFromExtendedPublicKey

The method `extractPublicKeyFromExtendedPublicKey(string xpub) returns (bytes)` takes as input an extended public key encoded in base58Check (as described in ref #4) and returns a byte array corresponding to the compressed public key of the given extended public key.

#### Validations

- `xpub` must be a valid base58check-encoded extended public key (either for mainnet or testnet).

#### Gas cost

This method has a fixed cost of 6,800 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

Given the base58Check-encoded extended public key `tpubDCwK6XsmwUx641qZ6Uyb2pcZXMCoFcyNBFZLbxwJT4iwgmUhY9DsW9FgaFUNE5s1KtM1upxpDuCvx1TbAqQMLMGFF5b1F6KpRpuHcLwcp4k` we would have that:

```
extractPublicKeyFromExtendedPublicKey('tpubDCwK6XsmwUx641qZ6Uyb2pcZXMCoFcyNBFZLbxwJT4iwgmUhY9DsW9FgaFUNE5s1KtM1upxpDuCvx1TbAqQMLMGFF5b1F6KpRpuHcLwcp4k') => '0x0360a7239100410a4f6019778fce803c6b168a9b1922a0eafbdff11e866070d9f8'
```

### getMultisigScriptHash

The method `getMultisigScriptHash(uint8 minimumSignatures, bytes[] publicKeys) returns (bytes)` takes as input a minimum required number of signatures and an array of (either compressed or uncompressed) secp256k1 public keys and returns the 20-byte hash (i.e., output of `hash160`) of the Bitcoin N-of-M multisig scriptPub corresponding to the given arguments. It is important to notice that the compressed version of the given public keys will be used to generate the script. Therefore, public key compression will not make a difference in the function output.

#### Validations

- `minimumSignatures` must be greater than zero and lower or equal to the number of public keys.
- `publicKeys` must be at least 1 and at most 15 (see ref #3 for details on the restriction).
- Each of the elements of `publicKeys` must be either of size 33 (compressed) or size 65 (uncompressed).

#### Gas cost

This method has a base cost of 13,500 gas units for the minimum number of keys (2). Then, per additional key an extra cost of 500 gas units is charged. So, for example, for 5 keys we would have a base cost of 13,500 plus the additional cost on the 3 extra keys: 1,500. That would result in a total of 15,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

Given the following public keys:

- 02276a07b202503d39a43896300224fb649818d1486b0eefde6c9cd7cf1e32eed8
- 0363e3de0d55459387f221dcda80f9283e303a08bcfeb9cf32875a43fefc7ecb1f
- 03173019874589385b0295b839bb70e5d3c33178c110bacd872e245b656b3e8f43

and _two_ minimum required signatures, we would have that:

```
getMultisigScriptHash(2, ['0x02276a07b202503d39a43896300224fb649818d1486b0eefde6c9cd7cf1e32eed8', '0x0363e3de0d55459387f221dcda80f9283e303a08bcfeb9cf32875a43fefc7ecb1f', '0x03173019874589385b0295b839bb70e5d3c33178c110bacd872e245b656b3e8f43']) => '0xafa22d52922aa6657417c55b6cebdf708b594576'
```

_Note_: The public keys used here are compressed, but might as well be uncompressed. The output should be exactly the same since the function handles the conversion.


## References

[1] Technical background of version 1 Bitcoin addresses
https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses#How_to_create_Bitcoin_Address

[2] Base58Check encoding
https://en.bitcoin.it/wiki/Base58Check_encoding

[3] Bitcoin Improvement Proposal #16 (BIP16)
https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki

[4] Bitcoin Improvement Proposal #32 (BIP32)
https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

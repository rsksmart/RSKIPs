# Precompiled contract with assorted BTO utility functions

|RSKIP          |00 |
| :------------ |:------------- |
|**Title**      |Precompiled contract with assorted BTO utility functions |
|**Created**    |01-FEB-19 |
|**Author**     |AM |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

A new precompiled contract (i.e., native, hardwired onto the RSK consensus layer) is introduced. This contract contains a few complex BTO-related operations that help leverage some of the work BTO contracts need to repeatedly perform.

## Motivation

With the arising need of BTOs (ICOs over Bitcoin) on top of the RSK network, a need for different BTO token and/or token-emitting contracts to perform certain complex BTO-related operations on a regular basis is in place. If written in solidity (or assembly WOLOG) and executed on the RVM, each of these functions would be expensive - and even more expensive to write and test every time. Furthermore, different implementations can lead to errors potentially protruding to loss of funds. A well-written, tested and precompiled function set that covers these operations will leverage the complexity on the BTO contracts' side while at the same time enforcing a certain level of security by implementing these as part of the RSK consensus.

## Specification

A new precompiled contract is to be accessible at the `0x0000000000000000000000000000000000000009` address. It will implement the following functions (signatures and return values are as follows):

- `toBase58Check(bytes20, uint8) returns (string)`
- `deriveExtendedPublicKey(string xpub, string path) returns (string)`
- `extractPublicKeyFromExtendedPublicKey(string xpub) returns (bytes)`
- `getMultisigScriptHash(uint8 minimumSignatures, bytes[]) returns (bytes20)`

### toBase58Check

The method `toBase58Check(bytes20 hash, uint8 version) returns (string)` takes as input a 20-byte hash and a version (see ref #2 for possible version values) and returns a base58Check encoded string of the concatenation of `version` and `hash`.

#### Sample usage

Given a public key, we want to generate a Bitcoin P2PKH address for testnet. We have the compressed key (for uncompressed keys, compression would be needed first) `02e6930d0659a24df1cc0061203d7845ca2e0c6b96bc1ad16e44c1cef24be92de8`. Its `hash160` is the result of applying `ripemd160(sha256(bytes))` to it. That yields `0d3bf5f30dda7584645546079318e97f0e1d044f`, which is the `hash` parameter we need for our function. For testnet, we use version `111`. Therefore, we would have that:

```
toBase58Check('0x0d3bf5f30dda7584645546079318e97f0e1d044f', 111) => 'mgivuh9jErcGdRr81cJ3A7YfgbJV7WNyZV'
```

### deriveExtendedPublicKey

The method `deriveExtendedPublicKey(string xpub, string path) returns (string)` takes as input an extended public key encoded in base58Check (as described in ref #4), an HD derivation path - a string of the form

```
S ::= M/P
P ::= n | n/P
```

with `n` an unsigned integer - also as described in #4 - and produces a base58Check encoded string corresponding to the extended public key obtained by deriving the given extended public key with the given path.

#### Sample usage

Given the base58Check-encoded extended public key `tpubD6NzVbkrYhZ4YHQqwWz3Tm1ESZ9AidobeyLG4mEezB6hN8gFFWrcjczyF77Lw3HEs6Rjd2R11BEJ8Y9ptfxx9DFknkdujp58mFMx9H5dc1r`, and the derivation path `M/2/3/4`, we would have that:

```
deriveExtendedPublicKey('tpubD6NzVbkrYhZ4YHQqwWz3Tm1ESZ9AidobeyLG4mEezB6hN8gFFWrcjczyF77Lw3HEs6Rjd2R11BEJ8Y9ptfxx9DFknkdujp58mFMx9H5dc1r', 'M/2/3/4') => 'tpubDCGMkPKredy7oh6zw8f4ExWFdTgQCrAHToF1ytny3gbVy9GkUNK2Nqh7NbKbh8dkd5VtjUiLJPkbEkeg29NVHwxYwzHJFt9SazGLZrrU4Y4'
```

### extractPublicKeyFromExtendedPublicKey

The method `extractPublicKeyFromExtendedPublicKey(string xpub) returns (bytes)` takes as input an extended public key encoded in base58Check (as described in ref #4) and returns a byte array corresponding to the compressed public key of the given extended public key.

#### Sample usage

Given the base58Check-encoded extended public key `tpubDCwK6XsmwUx641qZ6Uyb2pcZXMCoFcyNBFZLbxwJT4iwgmUhY9DsW9FgaFUNE5s1KtM1upxpDuCvx1TbAqQMLMGFF5b1F6KpRpuHcLwcp4k` we would have that:

```
extractPublicKeyFromExtendedPublicKey('tpubDCwK6XsmwUx641qZ6Uyb2pcZXMCoFcyNBFZLbxwJT4iwgmUhY9DsW9FgaFUNE5s1KtM1upxpDuCvx1TbAqQMLMGFF5b1F6KpRpuHcLwcp4k') => '0x0360a7239100410a4f6019778fce803c6b168a9b1922a0eafbdff11e866070d9f8'
```

### getMultisigScriptHash

The method `getMultisigScriptHash(uint8 minimumSignatures, bytes[]) returns (bytes20)` takes as input a minimum required number of signatures and an array of compressed secp256k1 public keys and returns the 20-byte hash (i.e., output of `hash160`) of the Bitcoin N-of-M multisig scriptPub corresponding to the given arguments.

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

## Rationale

TODO (if any)

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

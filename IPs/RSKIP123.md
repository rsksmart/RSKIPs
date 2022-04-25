---
rskip: 123
title: Multikey federation members 
description: 
status: Draft
purpose: Sca, Sec
author: AM <amendelzon@iovlabs.org>
layer: Core
complexity: 2
created: 2019-05-28
---
# Multikey federation members

|RSKIP          |123           |
| :------------ |:-------------|
|**Title**      |Multikey federation members |
|**Created**    |28-May-19 |
|**Author**     |AM |
|**Purpose**    |Sca,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

A network upgrade is introduced to enhance federation members with two additional keys. That is, each federation member will have three keys in total: one for Bitcoin transactions, one for RSK transactions and a last one, which we call MST, reserved for future use.

## Motivation

The current state of the federation security model relies heavily on a Bitcoin multisignature wallet to protect pegged Bitcoin funds. Each federation member holds a distinct key to that multisignature wallet. The main problem lies in that this exact same key is also currently used by the federation members to sign every single RSK transaction they send to the Bridge contract -- mandatory given that some Bridge methods are restricted only to federation members as an early protection against e.g. DoS attacks. This poses a big security risk in that a breach of a single key can compromise different aspects of the federation system, including loss of funds.

## Specification

At the time of writing, the `Bridge` contract works with a `Federation`, which is an abstraction for a set of distinct secp256k1 public keys, which can ultimately be used to represent a Bitcoin multisig (N of M) P2SH. To achieve this, the `Bridge` contract stores each federate member’s public key. The upgrade to this native contract involves replacing the set of public keys with a set of `FederationMember` objects, each of which will consist of three secp256k1 public keys (see subsection below). These keys will be assigned an individual role each, replacing the previous single multi purpose key model. In order to avoid a fork to the network as a consequence of the upgrade, each `FederationMember` will initially be constituted by three copies of the same previous multi-purpose key. The federate nodes that represent these federation members (by having control of the corresponding private keys) will also be upgraded in order to become multi-key aware, but in the same initial fashion. Therefore, even though both the `Bridge` contract and the federate nodes will be signing distinct transaction types with distinct keys, in practice these keys will initially remain the same and the system will operate as if unchanged. Only upon a federation change will distinct keys be specified, allowing for the upgrade to take full effect.

### Key set

As already mentioned, each federation member will have control over three distinct keys. These are:

- `BTC`: this key will be used solely for the signing of Bitcoin transactions related to the two-way peg. That is, this is the key that constitutes the multisig P2SH for the two-way peg. It is also the key that will be copied to the other two keys when the initial network upgrade takes place.
- `RSK`: this key will be used solely for the signing of federation member RSK transactions. That is, every single transaction that needs to be authenticated by any smart contract (e.g., the `Bridge` native contract) as coming from a federation member will be signed using this key.
- `MST`: this key will not have an immediate use. The idea is to have a distinct key to `BTC` and `RSK` that can potentially serve as a master key that can in turn be used to derive multiple distinct keys for purposes that require the federation's authorization.

### Storage upgrade

In order to transition between single-key federation members to multi-key federation members without forking the network, a storage version approach was selected (see Rationale for a discussion). The read and write logic for a federation into the `Bridge` contract's storage is as follows:

- Writing: before the network upgrade, write the federation slot with the federation serialized using the legacy format (single-key per federation member); do not write the storage version slot. After the network upgrade, write the storage version slot with a version number chosen for the multi-key federation format, and then write the federation slot with the federation serialized using the multi-key format.
- Reading: read both the storage version slot and the federation slot. If storage version is empty, proceed the federation desserialization with the legacy federation format. Otherwise, desserialize the federation according to the version number read (which should correspond to the multi-key format).

### Bridge methods upgrade

Due to the change in the number of keys per federation member, some methods of the `Bridge` native contract need an upgrade, i.e., some methods are disabled in favor of new ones. To avoid mistaking old versions of methods with new versions of these methods, a design decision was made: new methods have different names than old ones. The list of disabled methods is the following:

- `getFederatorPublicKey(int256) -> bytes`.
- `getPendingFederatorPublicKey(int256) -> bytes`
- `getRetiringFederatorPublicKey(int256) -> bytes`
- `addFederatorPublicKey(bytes) -> int256`

For each disabled method there is a new corresponding multi-key version method that replaces it. These are, in order:

- `getFederatorPublicKeyOfType(int256, string) -> bytes`: given an index between `0` and `n-1`, where `n` is the total number of federation members in the currently active federation, and a string indicating the type of key to retrieve (one of the case-sensitive strings `btc`, `rsk` or `mst`), this method returns the compressed (Bitcoin, RSK or MST, respectively) public key for the currently active federation.
- `getPendingFederatorPublicKeyOfType(int256, string) -> bytes`: given an index between `0` and `n-1`, where `n` is the total number of federation members in the currently pending federation, and a string indicating the type of key to retrieve (one of the case-sensitive strings `btc`, `rsk` or `mst`), this method returns the compressed (Bitcoin, RSK or MST, respectively) public key for the currently pending federation, if any.
- `getRetiringFederatorPublicKeyOfType(int256, string) -> bytes`: given an index between `0` and `n-1`, where `n` is the total number of federation members in the currently retiring federation, and a string indicating the type of key to retrieve (one of the case-sensitive strings `btc`, `rsk` or `mst`), this method returns the compressed (Bitcoin, RSK or MST, respectively) public key for the currently retiring federation, if any.
- `addFederatorPublicKeyMultikey(bytes, bytes, bytes) -> int256`: given three secp256k1 public keys corresponding to Bitcoin, RSK and MST keys, this method adds a new federation member with those keys to the currently pending federation and returns `1` to indicate success. If no pending federation currently exists, `-1` is returned. If any member of the currently pending federation has a match in any of the corresponding new keys, `-2` is returned.

## Rationale

When deciding upon the implementation of this network upgrade, one of the main issues discussed was how to handle the storage upgrade. There were essentially two possible approaches:

- **Version-based storage reads, block number based writes**. This is the approach that was ultimately agreed on. It is both simple and flexible in that is allows for the actual storage upgrade (i.e., the new format writes) to happen at any point after the designated network upgrade block number is reached.
- **Transaction induced storage upgrade**. This approach consisted on an automatic transaction (in the style of the REMASC transactions) being triggered once exactly on the designated network upgrade block number. This transaction would simply update the corresponding storage with the new format. Then, the format for reads and writes from and to storage from the `Bridge` contract itself would only rely on the current block number. Given the complexity associated with triggering the "upgrade transaction", this approach was ultimately discarded.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---
rskip: 57
title: Derivation Path for Hierarchical Deterministic Wallets
description: 
status: Draft
purpose: Usa
author: IO (@ilan)
layer: Net
complexity: 1
created: 2018-04-05
---

# Derivation Path for Hierarchical Deterministic Wallets

|RSKIP          |57           |
| :------------ |:-------------|
|**Title**      |Derivation Path for Hierarchical Deterministic Wallets |
|**Created**    |05-ABR-2018 |
|**Author**     |IO |
|**Purpose**    |Usa |
|**Layer**      |Net |
|**Complexity** |1 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes a unique derivation path for RSK deterministic wallet implementations, based on [BIP-0044](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki).

# **Motivation**

From a base private key, users can derive different child keys for privacy reasons or to facilitate identification of incoming payments.

To avoid incompatibility between wallets due to the usage of different techniques for derivation, a common derivation algorithm must be used. As an example, Trezor and Ledger use different derivation paths for Ethereum. This is misleading.

The best practice is to use the same derivation BIP-0044 path for all wallets.

# **Implementation**

Wallets implementing RSK support must use the following derivation path:

* **Purpose: 44**, based on BIP-0043. Hardened derivation is used at this level.
* **Coin type: 137** for RSK MainNet and 37310 for RSK TestNet, based on SLIP-0044. Hardened derivation is used at this level.
* **Account: 0**, as initial index. Hardened derivation is used at this level.
* **Change: 0**, used for external chain. Public derivation is used at this level.
* **Address index:** Addresses are numbered from index 0 in sequentially increasing manner. This number is used as child index in BIP32 derivation. Public derivation is used at this level.

*Public and hardened derivations based on BIP-0044*

|Network|Derivation path|
|-|-|
|RSK MainNet|m / 44' / 137' / 0' / 0 / N|
|RSK TestNet|m / 44' / 37310' / 0' / 0 / N|

## History
[BIP-0032](https://www.github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) describes hierarchical deterministic wallets (or "HD Wallets"): wallets which can be shared partially or entirely with different systems, each with or without the ability to spend coins. The specification consists of two parts. In a first part, a system for deriving a tree of keypairs from a single seed is presented. The second part demonstrates how to build a wallet structure on top of such a tree.

[BIP-0043](https://www.github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) introduces a "Purpose Field" for use in deterministic wallets based on algorithm described in BIP-0032.

[BIP-0044](https://www.github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) defines a logical hierarchy for deterministic wallets based on an algorithm described in BIP-0032 and purpose scheme described in BIP-0043. This BIP is a particular application of BIP43.

[SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) specifies the Coin Type body described on BIP-0044.

RSK MainNet becomes network 137. RSK TestNet becomes network 37310.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
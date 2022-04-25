---
rskip: 19
title: RSK Address formats
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2016-11-24
---

# RSK Address formats

|RSKIP          |19           |
| :------------ |:-------------|
|**Title**      |RSK Address formats|
|**Created**    |24-NOV-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Pre-git revisions

Modification Date: 06-23-2017

Date: 24-11-2016

Revision: 2

Status: Draft

# **Abstract**

This RSKIP discusses different types of addresses that could be added in the future and how to prepare clients to support such address. [RSKIP32]  (Sequential Address format) discusses one of these types. The address formats presented here can be more securely copy-pasted or, although not normally recommended, typed. Double-hashed addresses [RSKIP32]  do not need a checksum, but the checksum allows fast off-chain validation in the client side (e.g. Web app).

# **Motivation**

Current RSK addresses consist of 40 hexadecimal digits, such as "7ac5496aee77c1ba1f0854206a26dda82a81d6d8". This addresses will be called “raw” addresses. Raw addresses are difficult to type without mistakes, and carry no checksum, so an human mistake can cause the transferred funds to be lost forever. Even if addresses should normally be copy-pasted, the fields where the address is pasted could be unintentionally modified by the user, and again transferred funds may be lost. As the system should support the upgrade of address formats, a format prefix is recommended, to avoid depending address lengths to differentiate them. Also it is important to be able to differentiate addresses in testnets from production nets, and addresses from different blockchains.

## Discussion

Bitcoin has a Base58 address format that includes a four-byte checksum. An example of a Bitcoin address is “1FXqE2ixnnSB1kvwbMtWma5xQ2bVbkSq3f”. The drawback of Bitcoin addresses is that, even if they include a network identifier, it is not obvious. Ethereum proposes the use of IBAN compatible addresses (ICAP addresses). However, there are several limitations with IBAN addresses: scarce space for address, short checksum, additional complexity. 
Even if the address space in RSK is 160 bits, the number of effective addresses that will ever be created is much lower, probably not more than 2^30 or 1 billion addresses. One billion active addresses would consume about 100 Gigabytes of ledger state data. 

We define a simple way to differentiate different namespaces such as different systems can use different addresses types and encodings.

Base36 encoding has the drawback that 0 (zero), and O (capital o) look similar. Therefore we’ll use a base 35 encoding where the O is replaced by Z (the alphabet is therefore "0123456789ABCDEFGHIJKLMNZPQRSTUVWXY"). We’ll call this encoding Base35H.

This protosal specifically adds an optional blockchain identifier (sometimes called chain ID). If the chain Id is not present, is is assumed it refers to the RSK mainchain. 

# **Specification**

Every address starts with a network specifier. Current networks specifiers are:


|Prefix | Network type        |
|-------|---------------------|
|M      | Main network        | 
|T      | Test network        |

Next to the the network specifier, is the optional chain Id, in decimal format.

To avoid confusion with hexadecimal addresses, network specifiers should not be hexadecimal digits. Also it is recommended that the char “R” is not used as a network chars since IBAN-encoded addresses would start with R.

Then follows an address type specifier character. Currently address types are:

|Prefix | Address Type                         |
|-------|--------------------------------------|
|R      | Raw address (hexadecimal encoding)   |
|C      | Compact Raw address base35H encoded  |
|S      | Sequential address                   | 

Then follows the address body. Then a hyphen (-) is added and a 20-bit checksum encoded in BASE35H. The checksum is built as the first 3 bytes of SHA3 ( everything before the checksum not including the hyphen), zeroing the first 4 MSB bits.

|Net type | [Chain Id]  |Addr type        |Addr body     | checksum     |
|---------|-------------|-----------------|--------------|--------------|
| X       | NNN         | XXXXX….XXXXXX   | XXXX         |              |

### Sequential address (S )

A sequential address body consist of two parts: a sequential number and a 20-bit nonce as defined in [RSKIP43]. 

### Raw address (R ) body

A raw address body consist of an address type byte (according to [RSKIP16]) and a number between 160-bit and 248-bits expressed in hexadecimal, insensitive to case. The limit to 248 bits is because the EVM handles registers with a maximum size of 256 bits, and therefore they cannot handle a 256-bit address plus a 1 byte type specifier. When entered or presented on screen, most significant zeros should not be removed.

Examples: 
 20-byte single-hashed address
 MR007ac5496aee77c1ba1f0854206a26dda82a81d6d8-IJ5A
 
 20-byte single-hashed address with chain ID
 M60R007ac5496aee77c1ba1f0854206a26dda82a81d6d8-IJ5A

 20-byte double-hashed address
 MR027ac5496aee77c1ba1f0854206a26dda82a81d6d8-IJ5A
 
 31-byte double-hashed address
 MR0396aee77c1ba1f0854206a26dda82a81d6d85496aee77c1ba1f0854206a26dda8-J5AI

### Compact Raw Address (C ) body

A compact raw address is similar to a raw address but the address consist of the raw address expressed in Base35H (generally 31 digits, but can be less if the address starts with zeros). The address type is also coded in BASE35H. To decode, the decoder must starts with LSB digits, until all BASE25H digits have been consumed, and a  byte array has been created. The highest byte of this byte array contains the address type.

Examples: 
 20-byte single-hashed address
 MCXDMDJRI6NFBGKY97LNI53E1FI5SZH9D-IJ5A
 
 Sample RSK addresses (with random checksums)
 MS9ZLDRMH33-B8JJ
 TS7A3C7YM-ZLDR
 MR007ac5496aee77c1ba1f0854206a26dda82a81d6d8-IJ5A

### Encoding of standard RSK addresses into IBAN addresses

An IBAN code consists of up to 34 case insensitive alphanumeric characters. It contains three pieces of information:
The country code; a top-level identifier for the context of the following (ISO 3166-1 alpha-2);
The error-detection code; uses the mod-97-10 checksumming protocol (ISO/IEC 7064:2003);
The basic bank account number (BBAN); an identifier of the institution, branch and client account, whose composition is dependent on the aforementioned country.


The country code for RSK will be RK.

To encode a standard RSK address into an IBAN address the RSK address checksum is removed, because IBAN contains its own (although weaker) checksum.

The IBAN encoding allows a 140-bit C address to be encoded as IBAN, because a 140-bit number requires no more than 28 base35H chars, and 2 chars are consumed by network and address type.

|Country code | Checksum |Standard RSK Address (limited to 30 chars) |
|-------------|----------|-------------------------------------------|
|RK           | NN       |XXXXXXXX...XXXXXX                          |

Sample IBAN address (with random checksums)

 RK25PC3FJDKEHF738FZAMPORU3FJS0M1M6
 RK61PS9ZLDRMH33

To accept IBAN addresses the receiving software must use a special web3 service to scan the worldstate tree for any account starting with the specified 140-bit prefix. If more than one address is found, the function should fail with error.

[RSKIP32]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP32.md
[RSKIP43]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP43.md
[RSKIP16]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP16.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
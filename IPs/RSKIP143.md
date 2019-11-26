# Segwit Compatibility

|RSKIP          |143          |
| :------------ |:-------------|
|**Title**      |Segwit Compatibility|
|**Created**    |22-OCT-19 |
|**Author**     |PGP |
|**Purpose**    |Sca,USa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft|

## Abstract

This document describes the procedure to change the 2wp proccess to accept segwit transactions sent using a P2SH-PWPKH wallet. 
 
## Motivation

The 2wp process only accepts transactions that were sent using a P2PKH address. This RSKIP describes how the Bridge could accept P2SH-PWPKH[1] addresses since this is one of the most popular wallet nowadays and the segwit transactions use less space in a block. 

## Specification

During the execution of the 2wp process, every new transaction is parsed to obtain the public key from it. If this identification cannot be made, then the Bridge considers the transaction as invalid.
This RKSIP proposes changing the Bridge code of RegisterBtcTransactions function. This function is responsible for parsing Bridge transactions and obtaining the public key value from the data sent. 
RegisterBtcTransactions performs a number of validations to the BTC transaction to verify its legitimacy. After those validations have passed, it also checks if it's a lock, in that case it performs the lock using the sender's public key. These last two steps are the ones that need to be changed.
• Lock check. This check is currently only considering the input format for P2PKH is accepted.
• Sender public key fetching. Before getting the public key, this step validates the input's format, only the P2KH format.
Assuming it is not possible to get the P2PKH public key from the transaction's input, it should be checked if it is a P2SH-PWPKH type of input. The RegisterBtcTransactions function needs to verify if it is a lock transaction and then get the public key format using Segregated Witness format defined in BIP 144 [2].   

This is the only change required to be able to accept P2SH-PWPKH segwit transactions.

It has been checked that the functions of the Bitcoin library used support this type of format.

## References

[1](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki

[2](https://github.com/bitcoin/bips/blob/master/bip-0144.mediawiki) https://github.com/bitcoin/bips/blob/master/bip-0144.mediawiki

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

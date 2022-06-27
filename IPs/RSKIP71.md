---
rskip: 71
title: Transfer 2300 gas units for code execution in external transactions
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2019
---
#  **Transfer 2300 gas units for code execution in external transactions**  

| RSKIP          | --                                                           |
| :------------- | :----------------------------------------------------------- |
| **Title**      | Transfer 2300 gas units for code execution in external transactions |
| **Created**    | 2019                                                         |
| **Author**     | SDL                                                          |
| **Purpose**    | Usa                                                          |
| **Layer**      | Core                                                         |
| **Complexity** | 1                                                            |
| **Status**     | Draft                                                        |

# Abstract

A standard no-data transaction consumes 21K gas. Most wallets and users will automatically set a 21K gas value for their transactions. This RSKIP proposes to lower this base cost to 18700, but at the same time require a minimum amount of 21K gas to accept transactions as valid. The 2300 gas difference will passed to the receiver contract, in case it is a contract. This enables the receiver, in case it's a wallet, to log the reception of the funds.

# Motivation

Many services that could benefit from using smart contracts  are using simple accounts because other wallets do not compute correctly the amount of gas that should be specified in transactions. For example, exchanges could benefit from using contracts for user accounts because having multi-signature control and enabling the exchange to swipe the funds of multiple wallets with in a single transaction. To support smart-contracts as wallets all that is required is that the sender provides a minimum amount of extra gas for handling the reception. In fact the CALL opcode already passes a minimum of 2300 gas in a value transfer. This RSKIP proposes that the same amount is passed for external calls.

## Specification

The base cost of the transaction is still 21K gas, but 2300 gas is transferred to the receiver in case it's a contract execution. Even if the contract consumes less than 2300, no gas is returned.




# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



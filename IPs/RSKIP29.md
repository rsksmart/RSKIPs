---
rskip: 29
title: Change in Account creation cost to prevent Spam
description: 
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2017-01-01
---

# Change in Account creation cost to prevent Spam

|RSKIP          |29           |
| :------------ |:-------------|
|**Title**      |Change in Account creation cost to prevent Spam
|**Created**    |01-JAN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Rejected |

# **Abstract**

The cost of account creation is directly related to the cost of a Spam attack on the blockchain state. The only way to prevent spam is by increasing the cost of creating persistent storage (code, data and accounts). This RSKIP address how to increase the cost of creating spam accounts with "dust" SBTC.

# **Motivation**

If account creation costs as little as a CALL with value transfer (about 25K, or 1 US cents), then an attacker can spam the state with 10M accounts with as little as 100K USD. Ten million accounts may consume in an indexed data structure 2.5G bytes. This in turns increases the download of the state for new full nodes and SPV nodes by more than one hour. Even if using storage rent, the state won’t be cleaned for some time (e.g. 1 month of initial rent payment). 100K USD may be high for an individual attacker, but for a competing cryptocurrency, a bank or a governments it may be negligible..

# Discussion

There are several ways to prevent state spam:

1- Set a very close deadline for rent payment when creating a new account (not contracts, which are always pre-created). Set a special "initial" rent to be paid.

2-  Force accounts to be hibernated on creation

3- Prevent the creation of new accounts with CALLs unless a minimum amount of SBTC is transferred. The minimum amount is chosen to be not less than 210K gas (10 times the cost of an external transaction), or 10 cents.

4- Create a special type of address that represents the creation of the account. For example, the address begins with the letter "N" of “New”.  Therefore payers can ask recipients to pay for the cost of account creation when they receive the special address, and a fixed cost for account creation is set, whether is it created by CALL or by an external transaction.

The solution 3 is preferable. If the initial cost for creating an account must be higher than an external transaction (21K) and internal account creation (25K) (e.g. 10 cents or 210K gas), then external transactions must also be blocked from propagating in the network depending on the existence of the account if the transferred amount is not enough. 

## Comparing the cost to persist Code and Data

For a call to create an account, it consumes NEW_ACCT_CALL gas, which is 25000. A CALL creates an account even if it does not transfer value to it, therefore the initial account creation costs is 700+25000=25.7K gas. If an account consumes 256 bytes, the cost per byte is 100.3 gas. 

We we limit the creation to value transfers only (to prevent the creation of accounts with zero bitcoins) then the initial account creation costs is 34.7K (700 CALL + 9000 VT_CALL + 25000 NEW_ACCT_CALL). If an account consumes 256 bytes, the cost per byte is 135.8 gas. 

The cost in Ethereum of each code byte is 200 (CREATE_DATA), while RSK plans to reduce this to 100 gas by implementing storage rent [RSKIP27]. Therefore the cost of code persistence (100) and the cost of account persistence (100 gas) are similar.

The cost of SSTORE is 20K initially in Ethereum, minus 15K of refunds for storing a cell that consumes approximately . From the point of view of a Spam attacker, the cost is 20K, because he cannot re-use that gas to simultaneously acquire more cells. Therefore the cost per byte is 156 gas per byte. In RSK, if implementing storage rent, the cost of SSTORE is reduced to 10K. Therefore the cost per byte is 78 gas. Again, the costs of SSTORE data (78) are on the same rage as code persistence (100) and account persistence (100).

By implementing this RSKIP, the cost of account creation rises to 200K/256=781 gas per byte. Clearly account creation won’t be the weakest link for an attacker to spam the state. If  a similar threshold would be implemented for SSTORE, each SSTORE would need to cost 100K gas.

**Therefore this RSKIP is useless unless other protective measures are simultaneously taken. This RSKIP is now considered deprecated.**

# **Specification**

The minimum amount of SBTC to create an account is set to 210K (equivalent currently to 10 US cents). Therefore creating 10M accounts costs 1M USD.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[RSKIP27]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP27.md
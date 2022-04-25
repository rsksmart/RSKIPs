---
rskip: 87
title: Whitelisting unlimited mode 
description: 
status: Adopted
purpose: Usa
author: JD (@josedahlquist)
layer: Core
complexity: 2
created: 2018
---
#  **Whitelisting unlimited mode**  

| RSKIP          | 87                             |
| :------------- | :----------------------------- |
| **Title**      | Whitelisting unlimited mode 	  |
| **Created**    | 2018                           |
| **Author**     | JD                             |
| **Purpose**    | Usa 		                      |
| **Layer**      | Core                           |
| **Complexity** | 2                              |
| **Status**     | Adopted                        |

## Abstract

This RSKIP proposes a new whitelisting mode which will ease the whitelisting process.

## Motivation

Currently the whitelisting process implies re-whitelisting an address after a successful lock. In order to provide a more fluid service we propose adding a new whitelisting mode which enables locking BTC from and address indefinitely until set otherwise.

## Specification

In order to implement this change we suggest making the following modifications:
* Rename the existing whitelist mode to One-Off.
* Add a new whitelisting mode, Unlimited.
* Add a query method to gain information on the whitelisting state of a given address.

### addOneOffLockWhitelistAddress (formerly addLockWhitelistAddress)

Rename the existing method to avoid confusion on the type of whitelisting used, without any change in its actual behavior.
```
function addOneOffLockWhitelistAddress(string address, int256 maxValue) returns (int256)
```
Given a base58 BTC address and a maximum accepted value (in satoshis), it will return a value indicating success or failure.

This method will check if the address is valid, then will check if it's already whitelisted, finally it will add a new entry with the maximum value to the one-off storage. 

This mode will:
* reject any lock above the given maximum value.
* remove the whitelisted address after it's been used once.

### addUnlimitedLockWhitelistAddress

This new method will allow the user to add an unlimited whitelisted address.
```
function addUnlimitedLockWhitelistAddress(string address) returns (int256)
```
The main difference is that it wont't require a maximum value as it won't have a lock limit. It will perform the same validations as the previously described method and if they are successful it will add a new entry to the unlimited storage.

As opposed to the one-off mode, this whitelisted address won't be removed after the first usage; the caveat being that the user should call the removeLockWhitelistAddress method to prevent further locks from this address.


Both methods will return the same coded results. All results are returned by BridgeSupport class.

* Success (LOCK_WHITELIST_SUCCESS_CODE): 1.
	** Format address is correct.
	** The sender of the change to the whitelist is authorized to perform the change.
	** The address added to the whitelist hasn't been added before.  
* Unauthorized call (LOCK_WHITELIST_GENERIC_ERROR_CODE): -10
	** The new address is being added to the whitelist, but the sender of the operation is not authorized to perform the change.
* Invalid address (LOCK_WHITELIST_INVALID_ADDRESS_FORMAT_ERROR_CODE): -2
	** The function `getParsedAddress` takes a base58address and returns an RSK Address. If the base58address doesn't belong to any network, getParsedAddress throw this error.
* Already existing address (LOCK_WHITELIST_ALREADY_EXISTS_ERROR_CODE): -1.
	** The address added to the whitelist has been added before.
* Unknown error (LOCK_WHITELIST_UNKNOWN_ERROR_CODE): 0

It's worth mentioning that if you whitelist an address using one mode and then try to whitelist it using the other mode, this will produce a `-1` response. Always make sure to remove the whitelisted address before trying to add it again.

### removeLockWhitelistAddress

This method won't change but seems important to remark its existence and usefulness with the proposed unlimited mode.
```
function removeLockWhitelistAddress(string address) returns (int256)
```
This method expects a base58 address and if an entry matches that address, it will remove it.

The response codes of this method are equivalent to those of the creation, with the exception of the status `-1` meaning that the specified address is NOT whitelisted.

### getLockWhitelistEntryByAddress

Finally, this method will return information associated to the whitelisting status of a given address.
```
function getLockWhitelistEntryByAddress(string address) returns (int256)
```
given a valid base58 address it will return a coded result:
* Not existing (LOCK_WHITELIST_ENTRY_NOT_FOUND_CODE): -1.
	** When address queried is not in the LockWhitelist.
* Unlimited mode (LOCK_WHITELIST_UNLIMITED_MODE_CODE): 0
	** If the mode is not One-Off, then it is unlimited.
* One-off mode: any value bigger than zero, where that value is the amount of satoshis used as the maximum value. 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



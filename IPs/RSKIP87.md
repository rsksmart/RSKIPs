#  **Whitelisting unlimited mode**  

| RSKIP          | 87                             |
| :------------- | :----------------------------- |
| **Title**      | Whitelisting unlimited mode |
| **Created**    | 2018                           |
| **Author**     | JD                            |
| **Purpose**    | Sca, Usa                      |
| **Layer**      | Core                           |
| **Complexity** | 2                              |
| **Status**     | Adopted                          |

# Abstract

This RSKIP proposes a new mode of whitelisting which will ease the whitelisting process.


# Motivation

Currently the whitelisting process implies re-whitelisting an address after each usage, successful or not. In order to provide a more fluid service we propose to add a new whitelisting mode which enables an address to lock BTC indefinitely until set otherwise.

## Specification

In order to enable this new mode we have made added 2 new methods to the bridge contract and we also deprecated the existing one.

### addOneOffLockWhitelistAddress (formerly known as addLockWhitelistAddress)

We have renamed the existing method to avoid confusion on the type of whitelist used.
It works exactly as the previous one.
```
function addOneOffLockWhitelistAddress(string address, int256 maxValue) returns (int256)
```
You provide a base58 BTC address and the maximum accepted value (in satoshis), and it will return a value indicating success or not.

This method will check if the address is valid, then will check if it's already whitelisted, and if it isn't it will add a new entry with the max value to the oneOff storage. 

The particularity of this mode is that it will:
* reject any lock above the given max value
* remove the whitelisted address after the usage

### addUnlimitedLockWhitelistAddress

This new method allows the user to add an unlimited whitelisted address.
```
function addUnlimitedLockWhitelistAddress(string address) returns (int256)
```
The main difference is that it doesn't require a maxValue as it doesn't have a lock limit. It will perform the same validations as the prevously described method and if they are sucessful it will add a new entry to the unlimited storage.

As opposed to the OneOff mode, this whitelisted address won't be removed after the first usage, so keep in mind that will stay active until you remove it.


Both methods return the same coded results:
* success: 1
* unauthorized call: -10
* invalid address: -2
* already existing address: -1
* unknown error: 0

It is valid to mention that if you whitelist an address using one mode and then try to whitelist it using the other mode, this will produce a `-1` response. Always make sure to remove the whitelisted address before trying to add it again.

### removeLockWhitelistAddress

This method hasn't changed but it seems important to remark its existance and utility with the newly added Unlimited mode.
```
function removeLockWhitelistAddress(string address) returns (int256)
```
This method expects a base58 address and if it belongs to a whitelisted address it will remove it.

The response codes of this methods are equivalent of those of the creation, with the exception that the status `-1` means that the specified address is NOT whitelisted.

### getLockWhitelistEntryByAddress

Finally, this is a new query method useful to know if a given address is whitelisted, and if it is, which mode was used to create it.
```
function getLockWhitelistEntryByAddress(string address) returns (int256)
```
given a valid base58 address it will return a coded result:
* not existing: -1
* Unlimited mode: 0
* OneOff mode: any value bigger than zero, where that value is the amount of satoshis used as the max value. 

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



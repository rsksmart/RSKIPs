---
rskip: 540
title: Bridge method `getEstimatedFeesForNextPegOutEvent` improvements and new parameterized method
created: 04-DEC-25
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |540           |
| :------------ |:-------------|
|**Title**      |Bridge method `getEstimatedFeesForNextPegOutEvent` improvements and new parameterized method |
|**Created**    |04-DEC-25 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes improvements to the `getEstimatedFeesForNextPegOutEvent()` Bridge method and introduces a new method `getEstimatedFeesForPegOutAmount()` that allows users to estimate fees for a specific peg-out amount. The improvement changes how the extra peg-out output is calculated in the fee estimation, using the minimum peg-out value. Note that these methods return estimates based on the current state of the Bridge and are not guaranteed to match the actual fees charged when the peg-out event occurs.

## Motivation

As described in RSKIP271 [[1]](#references) and RSKIP385 [[2]](#references), the `getEstimatedFeesForNextPegOutEvent()` method was created to allow users to estimate peg-out fees. This method builds a peg-out transaction using the available UTXOs in the Bridge and adds an extra peg-out request to represent the hypothetical user request.

Currently, the implementation uses 1 BTC as the amount for this hypothetical peg-out request. This value may not accurately represent typical user transactions, especially for smaller peg-outs. Using the minimum peg-out value instead provides a more conservative and realistic fee estimation that works for all valid peg-out amounts, ensuring users receive accurate estimates regardless of the amount they intend to peg-out.

Additionally, users currently cannot estimate fees for a specific peg-out amount they intend to perform. A new method `getEstimatedFeesForPegOutAmount()` would allow users to query the estimated fees for their specific peg-out amount, providing better user experience and more accurate fee predictions. It is important to note that these estimates are based on the current state of the Bridge and may differ from actual fees due to changes in UTXO availability, peg-out requests state, or fee configuration.

## Specification

### 1. Modification to `getEstimatedFeesForNextPegOutEvent()`

The method `getEstimatedFeesForNextPegOutEvent()` should be modified so that when building the peg-out transaction and adding the extra hypothetical peg-out request, it uses the minimum peg-out value instead of 1 BTC.

The method signature remains unchanged:
```
function getEstimatedFeesForNextPegOutEvent() public view returns (uint256);
```

**Current behavior:**
- The method builds a peg-out transaction using available UTXOs in the Bridge and current peg-out requests
- It adds an extra peg-out request for 1 BTC to represent the hypothetical user request
- Returns the estimated fees for this transaction

**New behavior:**
- The method builds a peg-out transaction using available UTXOs in the Bridge and current peg-out requests
- It adds an extra peg-out request for the minimum peg-out value (as defined in RSKIP219 [[3]](#references)) instead of 1 BTC
- Returns the estimated fees for this transaction

### 2. New method `getEstimatedFeesForPegOutAmount(uint256 pegOutAmount)`

A new method is added that allows users to estimate fees for a specific peg-out amount. This method builds a peg-out transaction similar to `getEstimatedFeesForNextPegOutEvent()` but uses the provided `pegOutAmount` parameter for the hypothetical peg-out request.

**Method signature:**
```
function getEstimatedFeesForPegOutAmount(uint256 pegOutAmount) public view returns (uint256);
```

**Behavior:**
- This method builds a peg-out transaction using available UTXOs in the Bridge and current peg-out requests
- It adds an extra peg-out request using the provided `pegOutAmount` parameter as the amount
- Returns the estimated fees for this transaction
- The `pegOutAmount` parameter must be greater than or equal to the minimum peg-out value as defined in RSKIP219 [[3]](#references)

**Validation:**
- The `pegOutAmount` parameter must be greater than or equal to the minimum peg-out value
- If the validation fails, the method should revert with an appropriate error message

### Important Note on Fee Estimation Accuracy

**The fees returned by both methods are estimates and are not guaranteed to match the actual fees charged when the peg-out event occurs.**

Several conditions may change between the time the estimation is made and when the actual peg-out transaction is created, which can affect the final fee cost:

- **New peg-out requests**: Other users may add peg-out requests to the queue, changing the number of outputs in the transaction
- **Fee per KB changes**: The fee per KB value configured in RSK may be updated, affecting the transaction fee calculation
- **UTXO availability**: The specific UTXOs available at estimation time may differ from those available when the peg-out event is executed

Users and applications should be aware that the estimated fees are based on the current state of the Bridge at the time of the call and may differ from the actual fees charged. The estimates should be used as a guide rather than a guarantee.

## Rationale

### Using Minimum Peg-out Value

Using the minimum peg-out value instead of 1 BTC for the hypothetical peg-out request in `getEstimatedFeesForNextPegOutEvent()` provides several benefits:
- **More realistic estimates**: The minimum peg-out value better represents typical user transactions, especially for smaller amounts
- **Conservative approach**: Using the minimum ensures the fee estimate works for all valid peg-out amounts
- **Better UX**: Users performing small peg-outs will receive more accurate fee estimates

### New Method `getEstimatedFeesForPegOutAmount()`

The new method `getEstimatedFeesForPegOutAmount()` addresses the limitation where users cannot estimate fees for their specific peg-out amount. This provides:
- **Better accuracy**: Users can get fee estimates tailored to their specific transaction amount
- **Improved UX**: Users can compare fees for different peg-out amounts before submitting their request
- **Flexibility**: Enables applications and wallets to provide more accurate fee predictions to users

### Backwards Compatibility

The modification to `getEstimatedFeesForNextPegOutEvent()` maintains the same method signature, ensuring backwards compatibility with existing code that calls this method. The behavior change is an improvement that provides more accurate fee estimates, especially for smaller peg-out amounts.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

The modification to `getEstimatedFeesForNextPegOutEvent()` changes its behavior but maintains the same method signature, so existing contracts and applications calling this method will continue to work, though they will receive different (more accurate) fee estimates.

The new method `getEstimatedFeesForPegOutAmount()` is an addition and does not affect existing functionality.

## Security Considerations

No new security issues were identified related to this RSKIP. The new method `getEstimatedFeesForPegOutAmount()` includes validation to ensure the peg-out amount meets the minimum requirements, preventing invalid fee calculations.

Applications and users should be aware that the fee estimates returned by these methods are not guaranteed and may differ from actual fees due to changing conditions (UTXO availability, peg-out requests, fee configuration) between estimation and execution. This is expected behavior and not a security issue, but applications should handle potential fee discrepancies appropriately.

## References

[1] [RSKIP271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md): Peg-out batching functionality
[2] [RSKIP385](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP385.md): Bridge method `getEstimatedFeesForNextPegOutEvent` improvement
[3] [RSKIP219](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP219.md): New minimum values for peg-in and peg-outs

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

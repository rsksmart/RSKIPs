---
rskip: 502
title: PowPeg and Union Bridge integration
created: 19-MAR-25
author: MI
purpose: Sca
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |502           |
| :------------ |:-------------|
|**Title**      |PowPeg and Union Bridge integration |
|**Created**    |19-MAR-25 |
|**Author**     |MI |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes the integration of the Union Bridge [1] with the existing PowPeg to enable secure RBTC transfers between the two systems. The PowPeg will facilitate the transfer of RBTC to the Union Bridge, with the addition of a locking cap to mitigate risks by limiting the total amount of RBTC that can be transferred. The integration will extend the current Bridge contract's functionality, incorporating new methods for managing transfers, updating the Union Bridge contract address, and adjusting the locking cap. These changes will ensure that the PowPeg remains secure and robust while supporting the ongoing development of the Union Bridge. Special considerations have been made to allow seamless testing and updates during the Union Bridge's development phase, with restrictions placed on mainnet interactions to avoid unauthorized changes.

## Motivation

The Union Bridge is a new project currently under development that will interact with the Rootstock network. Integrating the Union Bridge with the existing PowPeg is essential to facilitate the transfer of RBTC. A key concern is ensuring that the integration remains secure, and that risks are minimized through a locking cap that controls the amount of RBTC transferred. The PowPeg's current functionality must be extended with minimal disruption, while also accounting for the need to manage the evolving nature of the Union Bridge.

Additionally, the Union Bridge is still in development, and its contract address is likely to change as it evolves. To accommodate this, the proposed solution includes a flexible mechanism for updating the Union Bridge contract address during its testing and development phases, ensuring that these changes can occur securely without risking mainnet operations.

This proposal aims to ensure that the PowPeg remains secure while enabling a smooth transition and integration with the Union Bridge as it matures, all while minimizing risks and maintaining full compatibility with the current system.

## Specification

### 1. New storage entry for Union Bridge contract address
A new storage entry is necessary to store the address of the Union Bridge contract. This will allow the Bridge to identify when transactions are coming from the Union Bridge, and provide a mechanism for managing these interactions securely.

Storage entry name: `unionBridgeContractAddress`

Maximum size: 32 bytes

### 2. Logic for updating the Union Bridge contract address
The Union Bridge contract address will be updated periodically during its development and testing phase. A new method will be added to the Bridge contract to allow updating this address. This method will only be available on testnet and regtest environments and will not be enabled on mainnet.

**Method signature:**
```
function setUnionBridgeContractAddress(address unionBridgeContractAddress) public returns (int256);
```

**Response codes:**
- 0: Success – The Union Bridge contract address was successfully updated.
- -1: Unauthorized caller – The caller does not have the necessary permissions to update the contract address.
- -10: Generic error – A non-specific error occurred while processing the request.

**Environment restriction:**

This method will be only enabled for testnet and regtest environments.
It will be disabled on mainnet to prevent unauthorized updates.

### 3. Authorizer for changing the Union Bridge contract address

An authorizer key will be created to allow the update of the Union Bridge contract address. This key will be similar to the whitelist authorizer [2], with a single key being sufficient for this purpose. No voting or multisig will be needed since this will only be relevant on testnet.

### 4. Union Bridge locking cap

A locking cap will be implemented to control the maximum amount of RBTC that can be transferred to the Union Bridge. This cap helps mitigate risks by preventing excessive transfers beyond the system's control limits.

In order to integrate the locking cap logic with the Union Bridge, new methods need to be added to the existing Bridge contract to enable querying and updating the locking cap. Based on RSKIP134 [3], which provides a standard mechanism for managing caps in smart contracts, new methods are defined.

#### 4.1 Method to get the current locking cap

A method will be added to allow anyone to query the current locking cap. This ensures transparency regarding the maximum allowable amount that can be transferred to the Union Bridge.

**Method signature:**

```
function getUnionBridgeLockingCap() public view returns (uint256);
```

#### 4.2 Method to adjust the locking cap

This method will allow authorized accounts to adjust the locking cap. The cap can only be increased, and the increase should be subject to a predefined maximum (e.g., doubling the current cap). The purpose of the adjustment method is to ensure that the cap can scale with demand, but only in a controlled way.

**Method signature:**

```
function increaseUnionBridgeLockingCap(uint256 newCap) public returns (int256);
```

**Response codes:**

- 0: Success – The locking cap has been adjusted successfully.
- -1: Unauthorized caller – Only authorized entities can adjust the locking cap.
- -2: Invalid value – The specified cap value is invalid (e.g., less than the current cap or excessive).
- -10: Generic error – A non-specific error occurred while processing the request.

### 5. Method for Transferring RBTC to the Union Bridge

A new method will be added to the Bridge contract to allow the transfer of RBTC to the Union Bridge. The method will have the following restrictions:

- Only callable by the Union Bridge contract.
- The total requested amount (current request + previously requested amounts) must not exceed the current locking cap.

**Method Signature:** 

```
function requestRBTC(uint256 amountInWeis) public returns
```

An event will be emitted each time this method is invoked, to provide transparency about the transfer.

```
event rbtc_requested(address indexed requester, uint256 amountInWeis);
```

### 6. Tracking transfers to the Union Bridge

A new storage entry will be used to track the total amount of RBTC that has been transferred to the Union Bridge. This is essential to ensure that the locking cap is not exceeded.

**Storage entry name:** weisTransferredToUnionBridge

Tracks the total amount of RBTC (expressed in weis) transferred to the Union Bridge.

### 7. Receiving funds back from the Union Bridge

The Union Bridge contract will have the capability to send funds back to the PowPeg. When this happens, the tracking entry for the amount transferred will need to be updated to reflect the returned RBTC.

A new method will be added to the Bridge contract to allow the transfer of RBTC from the Union Bridge back to the PowPeg. The method will have the following restrictions:

- It will only be callable by the Union Bridge contract address.
- The amount sent back cannot exceed the total amount of RBTC previously transferred to the Union Bridge, as tracked by the **weisTransferredToUnionBridge** storage entry. In other words, the Union Bridge cannot return more RBTC than it has previously received.

**Method Signature:** 

```
function releaseUnionRBTC() public returns (int256)
```

**Response codes:**

- 0: Success – Funds were successfully transferred from the Union Bridge back to the PowPeg, and the internal tracking was updated accordingly.
- -1: Unauthorized caller – The caller is not the Union Bridge contract address.
- -2: Invalid value – The amount being returned exceeds the previously transferred amount.
- -10: Generic error – A non-specific error occurred while processing the request.

An event will be emitted each time this method is invoked, to provide transparency about the transfer.

```
event union_rbtc_released(address indexed receiver, uint256 amountInWeis);
```

### 8. Pause mechanism

To allow for emergency halts or suspension of interactions between the PowPeg and the Union Bridge, a pause mechanism will be implemented. When the system is paused, all transfers between the PowPeg and the Union Bridge will be suspended until the pause is lifted.

The pause mechanism will allow authorized signers to pause the system temporarily. Once the system is paused, no transfers of RBTC between the PowPeg and Union Bridge can occur. The system can later be unpaused, resuming the normal transfer of RBTC.

**Method signature:** 

```
function toggleUnionBridgeIntegrationPause() public returns (int256);
```

**Response codes:**

- 0: Success – The system has been toggled (paused or unpaused) successfully.
- -1: Unauthorized caller – Only authorized entities can toggle the pause state.
- -10: Generic error – An unexpected issue occurred during the toggle action.

## Backwards Compatibility

This RSKIP does not break any existing functionality in the PowPeg. The changes introduce new methods and storage entries that do not interfere with existing operations. The locking cap and other features are added as extensions to the current bridge functionality.

## Security Considerations

**Locking cap:** The locking cap ensures that the amount of RBTC transferred to the Union Bridge remains within controlled limits, preventing excessive transfers.

**Contract address update:** The method to update the Union Bridge contract address is limited to testnet environments to prevent potential security risks on  mainnet.

**Authorization:** The new authorizer mechanism ensures that only authorized parties can modify key parameters, such as the Union Bridge contract address.

## Conclusion

This RSKIP outlines the necessary steps for integrating the Union Bridge with the PowPeg while maintaining security and minimizing risks. The proposed changes will allow the PowPeg to transfer RBTC to the Union Bridge while providing mechanisms for tracking, limiting transfers, and pausing the integration if needed.

## References

[1] Union Bridge whitepaper https://arxiv.org/abs/2501.07435

[2] RSKIP87 https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP87.md

[3] RSKIP134 https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP134.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

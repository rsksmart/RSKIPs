# Precompile to get transaction refunds

|RSKIP          | 139 |
| :------------ |:-------------|
|**Title**      |Precompile to get transaction refunds|
|**Created**    |2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

The GasStation Network (GSN) requires a first contract to bill a second contract for the gas consumed. The first contract can learn how much has the second contract consumed by calling GAS opcode before and after the second contract has executed. However, the first contract cannot learn how much gas the second contract execution has refunded. If it were known, the refunded gas could be deducted from the bill. Currently the second contract is overpaying gas. This RSKIP proposes the creation of a new native contract and method that returns the accumulated refund.



# **Specification**

The accumulated refund (REFUND) is the gas that will be refunded because of cleared storage cells and destroyed contracts. A new native contract called STATEProvider is created at address 0x000000000000000000000000000000000100000A.

The contract has a single method with ABI signature: getRefund() returns (uint256);

The method returns the current accumulated refund. Calling any other method results in a OOG exception.

The cost of the execution is 10.

It must be noted that the actual refund executed at the end of the transaction is capped to half of the gas consumed. However getRefund() returns the maximum possible refund (without the cap).



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 
# Increase max timestamp difference between btc and rsk blocks for Testnet

|RSKIP          |297           |
| :------------ |:-------------|
|**Title**      |Increase max timestamp difference between btc and rsk blocks for Testnet |
|**Created**    |13-APR-22 |
|**Author**     |VK |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Accepted |

## Abstract

As of now, there's a consensus validation rule, which says that a difference between timestamps of RSK and BTC merged
mining block headers cannot be bigger than 5 minutes. This RSKIP proposes to increase that value for `testnet` to 120 minutes.

## Motivation

Quite often BTC blocks in `testnet` are being generated with timestamps in the future (however the timestamp cannot
be more than 2 hours in the future according to the protocol). This increase to 120 minutest of the max difference between
RSK and BTC timestamps has to mitigate the issue with future BTC blocks.

## Specification

Return `7200` in the `org.ethereum.config.Constants#getMaxTimestampsDiffInSecs` method, if a current network is `testnet`
and `RSKIP297` is activated, otherwise, return the previous value - `300`.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

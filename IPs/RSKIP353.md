---
rskip: 353
title: Align RSK P2SH redeem script with Bitcoin Core standard transactions checks
created: 24-OCT-22
author: MI
purpose: Usa,Sec
layer: Core
complexity: 2
status: Draft
description: 
---

|RSKIP          |353           |
| :------------ |:-------------|
|**Title**      |Align RSK P2SH redeem script with Bitcoin Core standard transactions checks |
|**Created**    |24-OCT-22 |
|**Author**     |MI |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

On RSK Mainnet block #4,675,281, the network created the first Bitcoin UTXOs containing the PowPeg Time-locked Emergency Multisig (ERP). See RSKIP-201 [1] for more details about ERP. 

The ERP proposal required a change to the peg-out Bitcoin transactions’ scripts, making them longer and more complex. We detected that the RSK P2SH redeem script is mischaracterized as “non-standard” by the Bitcoin Core nodes with the default configuration. Although the transactions are valid from a Bitcoin consensus point of view, default-configured Bitcoin nodes will not include them in their transaction mempool or broadcast them to the rest of the network. As a result, transactions are not being confirmed on the Bitcoin network, even when the pegout process on the RSK network side is completed.

Full details are explained in [2].

This RSKIP proposes a change to the RSK P2SH redeem script so it complies with current Bitcoin Core validations for standard transactions.

## Motivation

As explained in [2], after ERP activation, peg-out Bitcoin transactions are being characterized as "non-standard" by the Bitcoin Core nodes with the default configuration. As a result, peg-out transactions are not being confirmed on the Bitcoin network. Changing RSK redeem script as described in next section will let the network resume its peg-out operations. 

## Specification

Current RSK P2SH redeem script looks like this:

```
OP_NOTIF
   OP_M
   PUBKEYS...N
   OP_N
OP_ELSE
   OP_PUSHBYTES
   CSV_VALUE
   OP_CHECKSEQUENCEVERIFY
   OP_DROP
   OP_M
   PUBKEYS...N
   OP_N
OP_ENDIF
OP_CHECKMULTISIG
```

The proposal is to change it to the following:

```
OP_NOTIF
   OP_M
   PUBKEYS...N
   OP_N
   OP_CHECKMULTISIG
OP_ELSE
   OP_PUSHBYTES
   CSV_VALUE
   OP_CHECKSEQUENCEVERIFY
   OP_DROP
   OP_M
   PUBKEYS...N
   OP_N
   OP_CHECKMULTISIG
OP_ENDIF
```

The `OP_CHECKMULTISIG` opcode is moved inside the `OP_ELSE` statement just after the `OP_N` opcode, making the script comply with Bitcoin Core standard checks. 

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated.

## References

[1] [RSKIP-201: ](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md)

[2] [Incident report: RSK peg-out service outage](https://blog.rsk.co/noticia/incident-report-rsk-peg-out-service-outage/) 

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

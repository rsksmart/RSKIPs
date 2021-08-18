# Expanding the emergency multisig

|RSKIP          | X                                |
| :------------ |:---------------------------------|
|**Title**      | Expanding the emergency multisig |
|**Created**    | AUG-2021                         |
|**Author**     | JL2                              |
|**Purpose**    | Sec                              |
|**Layer**      | Core                             |
|**Complexity** | 2                                |
|**Status**     | Draft                            |

## RFC 2119 compliance
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Background

In the Iris hard fork, the RSK community has adopted RSKIP-201, which institutes a 3-of-4 emergency multisig with a one-year timelock in case of inactivity by a quorum of PowHSM nodes (henceforth, "the Iris multisig").[1] RSKIP-201 intends to ensure the availability of funds in case a quorum of PowHSM nodes goes offline for an extended period of time due to hardware failure, software bug, or some other issue. While RSKIP-201 does provide a backup option in case of Powpeg failure, it opens up new attack vectors due to the difference in security models between the Powpeg and the Iris multisig.  

[1] https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md  

## Proposal

We propose to modify and expand the Iris multisig so that it is a 7-of-12 multisig, in line with the 7-of-12 PowHSM nodes required to operate the Powpeg. The new emergency multisig would use Schnorr key aggregation to reduce the size of the emergency multisig transactions, saving block space and transaction fees. Emergency multisig signers SHOULD keep their private key in separate national legal jurisdictions and geographies to minimize the risk of coordinated attacks and natural disasters. The emergency multisig signers MUST also be a totally separate group of legal entities from the PowHSM operators.

## Specification

- Change the Iris emergency multisig to a 7-of-12 multisig using Schnorr key aggregation.  
- Ensure that no PowHSM operators are also emergency multisig signers, and that each emergency multisig signer is in a separate national legal jurisdiction and geography.  

## Signatories

Following the requirements established in [RSKIP225-Emergency Multisig public keys](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP225.md), the signatories for the emergency multi-signature are listed below:

- IOVlabs ([link to message](https://iovlabs.org/pow-peg-emergency-multisig.html))  
- MoneyOnChain  
- Jameson Lopp ([link to message](https://keybase.pub/lopp/RSK-key.txt))  
- Adrian Eidelman ([link to message](https://keybase.pub/aeidelman/RSK-key.txt))  
- TBD  
- TBD  
- TBD  

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

## Test Cases

TBD

## Implementation

TBD

## Dependencies

This RSKIP depends on a change to the Powpeg firmware to support Taproot peg-outs. The new emergency multisig would use a Schnorr key aggregation scheme enabled by the Schnorr/Taproot bitcoin soft fork which is expected to be activated in November 2021.[2][3][4]  

[2] https://blockstream.com/2018/01/23/en-musig-key-aggregation-schnorr-signatures/  

[3] https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki  

[4] https://github.com/bitcoin/bitcoin/pull/21686/files  

## Alternatives considered

**Removing the emergency multisig**  

This option was considered because there are some in the community who feel that having no emergency multisig is preferrable to having the Iris emergency multisig. We decided that, while we recognize the shortcomings of the Iris multisig, we would rather work to improve it rather than un-do the consensus that was formed for implementing the Iris multisig in the Iris hard fork.  

**A backup Powpeg**  

With this option, in case of a failure of the primary PowHSM nodes, the funds held in the Powpeg multisig would be sent to a multisig controlled by backup PowHSM nodes. These backup PowHSM nodes would be powered down until they are needed in case of emergency, thus protecting them from wear and tear and potential disasters such as an electromagnetic pulse. The backup PowHSM nodes would also be run by a different set of entities in different national legal jurisdictions and geographies than the primary PowHSM nodes. 

The main downside of this approach is that if the primary PowHSM nodes fail due to hardware or software issues, the backup PowHSM nodes might be vulnerable to the same issue and fail as well. This could be mitigated by using a heterogeneous hardware and software stack, consisting of different hardware from different manufacturers and a different implementation of the software used to operate the PowHSM nodes. However it will take more time to research and consider the costs and tradeoffs involved in this approach, and we prefer to make some positive change in the short term to strengthen the security of the emergency multisig.

**Conclusion**  

Having considered these alternatives, we decided on a simple expansion of the emergency multisig to increase the number of signatures required as a way of improving redundancy and raising the cost of an attack.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---
rskip: 305
title: Segwit for our Peg
description: This RSKIP introduces Segwit (Bitcoin Soft Fork 2017) for use in RSK Pegs and what are its benefits.
status: Draft
purpose: Sca Usa Sec
author: PG (@patogallardo) - RFV (@ramsesfv) - NV (@NVescovo)
layer: Core
complexity: 2
created: 2022-06-01
---

# Segwit for our Peg

|RSKIP          | 305        |
| :------------ |:-------------|
|**Title**      |Segwit for our Peg |
|**Created**    |01-JUN-2022 |
|**Author**     |Patricio Gallardo, Ramsès Fernàndez-València, Nicolás Vescovo|
|**Purpose**    |Sca Usa Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP introduces Segwit (Bitcoin Soft Fork 2017) for use in RSK Pegs and what are its benefits in terms of performance, cost reductions and security.

# **Motivation**

The RSK peg out uses multisignatures to implement a threshold scheme that protects the bitcoins locked. Multi-signatures are simple and relatively 
efficient for small groups, but they become expensive as the number of signers increases. Each input signed in a peg-out transaction must transmit
m-out-of-n signatures, and each signature consumes 65 bytes, and n public keys, consuming 33 or 65 bytes each, depending on if they are compressed or not.
Bringing Segwit to the RSK peg will tackle all these problems.

# **Specification**

## **Introduction to Segwit**
Segwit stands for segregated witness, and it is an upgrade to the Bitcoin consensus rules and network protocol proposed and implemented as a BIP – 9 soft-fork that was activated on Bitcoin’s mainnet in 2017.

In Bitcoin argot, a digital signature is one kind of witness. Nevertheless, the concept is much broader that refers to any solution that can satisfy the conditions upon a UTXO and unlock it for spending.

Before segwit’s introduction, every input in a transaction was followed by the witness data that unlocked it. The witness data was embedded in the transaction as part of each input. It is called segregated witness because it implies separating the signature or unlocking script of a specific output.

The main reason for its creation was to protect against the possibility that small pieces of transaction information could be changed (invalidating new cryptocurrency blocks). Segwit also reduces transaction times by increasing block capacity.


## **Why Segwit?**
Segwit has several effects in terms of scalability, security or performance that we describe briefly:
* **Transaction malleability:**
By segregating the witness and the transaction, the transaction hash used as identifier no longer includes information about the witness, 
therefore one avoids malleability attacks.

* **Script versioning:**
By introducing segwit scripts every locking script is associated with a script version number, which allows scripting language to be improved in a way compatible with soft-fork upgrades and to introduce new syntax or semantics.

* **Scaling:**
Moving the witness data outside the transaction one avoids the problems associated with one of the main contributors to the total size of transactions. This leads to an improvement in Bitcoin’s scalability. Once a signature is validated, witness data can be ignored and does not need to be transmitted.

* **Signature verification improvement:**
Segwit improves the signature functions, including CHECKSIG or CHECKMULTISIG, to reduce computational complexity. Before segwit, signature production required a number of hash operations proportional to the size of the transaction involving complexities of the order O(n<sup>2</sup>). With Segwit the complexity becomes O(n).

* **Offline signing improvement:**
Segwit signatures include the value referenced by each input in the hash that is signed, therefore an offline device does not need previous transactions: if the amounts do not match, the signature will not be accepted.



# **Rationale**

We first analyze two other protocols Frost and MuSig2. Although some advantage is obtained in relation to performance, this is not enough to counteract 
the greater difficulty of implementation that they need. This is why we consider that implementing Segwit is the best option.

| Topics            | FROST      | Musig2    | Segwit    |P2SH (current)  |
| ----------------- |:----------:| ---------:| ---------:| --------------:|
|PowHSM Complexity  | High       | Low       | ?         | -              |
|Fee Cost type      | Fixed      | Variable  | Variable  | Variable       |
|INPUT Fee Cost (vbytes) 7/13| 16| 120       | 238.75    | 955            |
|INPUT Fee Cost (%)7/13| -98%    | -87%      | -75%      | -              |
|TX Fee Cost (vbytes) (1)| 106   | 210       | 328.75    | 1045           |
|TX Fee Cost (%) (1)| -89%       | -78%      | -66%      | -              |
|Verification Complexity|Medium/Low| High    | Low       | Low            |
|Robustness         | Medium/Low | Low       | High      | High           |
|Implementation time| Long term  |Medium term|Short/Medium-term| -        |
|Threshold N limit (2)|Unlimited (3)| 34     | 67        | 15             |
 
**Observations:**
1. A TX is composed of 1 input and 2 outputs.
2. This is the higher N, independent of M. We are considering M as the half of N, which is the worst-case scenario. If M is more or less than half of N, the total starts to decrease. 
3. FROST unlimited threshold limits it just in theory and what we can represent through aggregated signatures, practically we would have restrictions at the Bridge complexity level.




# **Implementations**

The main components that should be modified due to the adoption of Segwit are the signing functions: CHECKSIG, CHECKSIGVERIFY, CHECKMULTISIG, and 
CHECKMULTISIGVERIFY.

Before Segwit, signatures in transactions are applied on a commitment hash, which is computed from the transaction data, locking specific parts of the 
data indicating the signer’s commitment to those values. 

The way the commitment hash was computed introduced the possibility that a node verifying the signature can be forced to perform a significant number 
of hash computations, as shown in the analysis.

Segwit represents an opportunity to address this problem by changing the way the commitment hash is calculated. For Segwit v0 witness programs, signature 
verification occurs using an improved commitment hash algorithm as specified in BIP-143.

The new algorithm achieves:

1. A more gradual increase of hash operations, as discussed in previous sections. This reduces the chances of suffering DoS attacks with complex 
transactions. 

2. A signer can commit to a specific input value without needing to retrieve and check the previous transaction referenced by the input. For offline 
devices, this simplifies the communication between hosts and the hardware wallets.

The adoption of Segwit involves two important scenarios:

1. The ability of a sender’s wallet that is not segwit-aware to make a payment to a recipient’s wallet that can process segwit transactions: If the 
sender’s wallet is not upgraded to segwit, but the receiver’s wallet is upgraded and can handle segwit transactions, they can use nonsegwit transactions. 
But the receiver would likely want to use segwit to reduce transaction fees. In this case, the receiver’s wallet can construct a P2SH address that 
contains a segwit script inside it. The sender’s wallet sees this as a P2SH address and can make payments to it without any knowledge of segwit. The 
receiver’s wallet can then spend this payment with a segwit transaction, taking full advantage of segwit and reducing transaction fees. Both forms of 
witness scripts, P2WPKH and P2WSH, can be embedded in a P2SH address. The first is noted as P2SH(P2WPKH) and the second is noted as P2SH(P2WSH).

2. The ability of a sender’s wallet that is segwit-aware to recognize and distinguish between recipients that are segwit-aware and ones that are not, 
by their addresses.

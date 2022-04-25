---
rskip: 75
title: Native Off-Chain Probabilistic payments
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2018-05-07
---
# Native Off-Chain Probabilistic payments


|RSKIP          |75           |
| :------------ |:-------------|
|**Title**      |Native Off-Chain Probabilistic payments|
|**Created**    |07-MAY-2018 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

Transaction costs will increase with the demand for the platform services because the block gas limit cannot be arbitrarily increased without affecting other properties of the platform. The second layer payment networks, such as Lumino, enable off-chain payments at low cost. However these networks require pre-locking funds. In this RSKIP we propose an alternative to the second layer networks to perform cheaper payments without the need to lock funds, based of probabilistic payments. Also this RSKIP proposes one variant of off-chain probabilistic payments.

# **Motivation**

Many new micropayment use cases can be realized with payment channels, such as pay-per-minute of video, bandwidth, CPU or consulting services. However second layer networks are complex and bring second layer networks have different security guarantees than blockchain transactions. Probabilistic payments are payments that only occur with certain probability. Suppose a payment stream of $0.01 payment per second. This can be transformed into a stream of one $ 0.10 payment per 10 seconds. If the probability of each payment is 1/10, then each payment can be a $ 1 probabilistic payment, therefore it can be transformed into a stream of $ 1 1/10 probabilistic payment per 10 seconds, which results in one actual $1 payment every 100 seconds, on average.

A probabilistic transaction can involve smart contract execution.

## Discussion

Alice want to Probabilistically pay to Bob. Both agree in a chance value c, such as the probability is 1/c. 

Alice picks a random number RA, and sends H(RA) to Bob. Bob picks another random number RB and sends RB to Alice. Now Alice reveals RA and Bob revelals RB, Both compute V = RA xor RB. If V mod c ==0, then Alice pays Bob with a normal On-chain transaction. 

# **Specification**

No network change is required. All the protocol is a private two-party message exchange.

## Example Use Case

Because probabilistic transactions may be referenced in many blocks, the protocol for fair probabilistic payment must be carefully designed. 

Pay-for-minute between Alice and Bob:

* Let assume the rate is $ 0.1 per 10 seconds. Let the average block interval be 10 seconds.

* Alice creates a probabilistic transaction T for Bob. The transaction would pay $ 0.6 (the amount if every block would reference the transaction). Let’s say she specifies $ 1. then she broadcasts the transaction.

* Bob waits until T is broadcast, and assumes miners are referencing it in probabilisticTx. Then it starts giving Alice a 1 minute service.

* Each time T is referenced, Bob considers that Alice probabilistically paid $ 0.1. Let A(t) be the accumulated amount of probabilistic payments at delta time t (t is specified in seconds).

* When T is executed, Alice immediately creates a transaction T’ with the next nonce, specifying the same amount, and she broadcast it. Alice can increase or decrease the amount her transactions are being referenced too often or too little. 

* If Bob has provided a service for t time and A(t)< t*0.1/10+0.6 then Bob interrupts the service, and waits for more  references.

* Alice must make sure that A(t) is high enough, either by increasing the gasPrice, or increasing the amount paid per transaction.

See [RSKIP74] for a Comparison between on-chain and off-chain probabilistic payments

[RSKIP74]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP74.md	


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
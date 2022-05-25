---
rskip: 198
title: Minpeg, a miners' multisig in the peg
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-01
---
# Minpeg, a miners' multisig in the peg


|RSKIP          | 198 |
| :------------ |:-------------|
|**Title**      |Minpeg, a miners' multisig in the peg|
|**Created**    |JAN-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||


# **Abstract**

In this RSKIP we propose a new layer of security for the RSK Powpeg that engages the active miners in a multi-signature scheme that represents a new requirement for validity of peg-out transactions. The new group and system will be called **minpeg**. There is a key difference between a federation and a minpeg. A federation group is generally static or it has a built-in mechanism to add or remove new members through voting. The minpeg group is a dynamically chosen set of top miners from the set of active miners, and members come and go automatically as they increase or decrease their participation on the RSK hashrate. Internally, the multisignature is renewed at periodic intervals, according to the miners' contributed hashrate on them. Also, the federated model is characterized by full identification of its members by real-world names (either individual or legal).  In contrast, minpeg  participants do not need to reveal their real-world identities.  

This RSKIP achieves some of the capabilities of a "discrete" drivechain, in the sense that miners attest for the correctness of peg-outs. By "discrete" we emphasize that the hashrate measurements is taken at discrete intervals rather and continuously for every block. However this RSKIP does not provide to the peg a forced long peg-out delay, which is a key property of the drivechain proposals. A forced-delay could be achieved in the future with the addition of covenants to Bitcoin scripts. 

## Motivation

The current security of the Powpeg against invalid peg-outs relies on the assumption that:

(the majority of the RSK hashrate is honest **OR** the majority of the Powpeg functionaries are honest)

**AND** the PowHSM manufacturer is honest **AND** the PowHSM is tamper-proof.

Assuming of the honesty of the HSM manufacturer and lack of vulnerabilities in the device is Powpeg's strongest assumption. There are several ways in which the PowHSM manufacturer can cheat or the PowHSM can fail:

1. The PowHSM can leak the private key to manufacturer in a covert channel over Bitcoin transaction signatures.
2. The manufacturer can hide a backdoor to extract the private key with the collusion of Powpeg functionaries.
3. The manufacturer can add a surreptitious method to selectively skip certain code paths, such as the proof-of-work validation. 
4. The manufacturer, or any malicious party, may posses knowledge of a side-channel (i.e. a power glitch timing) that can be used to force the PowHSM to perform invalid execution branches, skipping proof-ok-work validation.

This RSKIP attempts to protect RSK from all these potential vectors of attack.

The defense-in-depth strategy adopted by the RSK community states that there should be no single point of failure. While the probability of the described attacks is very low, the defense-in-depth strategy demands that an additional protection layer exists. This new layer should keep the system secure even in case the HSM manufacturer is compromised. This RSKIP proposes such new layer of protection. 

### Threat Model

The new layer of defense added gives a chosen set of miners certain additional responsibilities and therefore powers, which brings up the question of miners' trust. However, the "dishonest miner" has very limited capabilities with the changes proposed in this RSKIP. First, because the minpeg multisig is an additional requirement of the Bitcoin peg-out script, the best attack miners can perform is full peg-out censorship. Full censorship can only be profited from through extortion, and that risk is mitigated with the addition of time-locked emergency multi-signature in the upcoming Iris network upgrade. Second, malicious miners cannot easily hide their actions. RSK merge-mining is normally attributable. To become surreptitious, a dishonest miner must stop adding identity information to blocks (or start using others' miners identities) much before the attack. Hiding the identity of a the majority of Bitcoin or RSK hashrate without this being noticed by the community seems to be difficult.




## Specification

Every E=40000 blocks, which roughly corresponds to 15 days at the current block rate, the bridge triggers a **funds recycle protocol** or simply a **recycle**.
In a recycle, all UTXO held by the bridge are moved into a new multisignature address. The minpeg system becomes active immediately after the first recycle occurs. The new multisignature address scriptPub has three two: the first which enforces the PowHSM multisignature and the second which enforces a miner's hashrate-weighted multisignature.

The script has the following format:

```
<powpegN> <powHSM_pubkey1>  ... <powHSM_pubkeyN> <powpegM> OP_CHECKMULTISIGVERIFY 
<minerPubkey1> OP_CHECKSIG
OP_IF
  <attestation_weight1>
OP_ELSE
  0
OP_ENDIF

OP_SWAP
<miner_pubkey2> OP_CHECKSIG
OP_IF
  <attestation_weight2>
  OP_ADD
OP_ENDIF

OP_SWAP
<miner_pubkey...> OP_CHECKSIG
OP_IF
  <attestation_weight...>
  OP_ADD
OP_ENDIF

OP_SWAP
<miner_pubkeyN> OP_CHECKSIG
OP_IF
  <attestation_weightN>
  OP_ADD
OP_ENDIF

<half_of_total_attestation_weight_plus_one>
OP_GREATERTHAN
```

The powHSM_pubkey(i) public keys correspond to a multisignature managed by PowHSMs.

The miner_pubkey(j) public keys correspond to a multisignature managed by the 20 top active miners. Each elected miner potentially contributes with attestation_weight(j). The weighted multisignature is accepted if the weight surpasses half of the total weight plus one.

Miners have the option to participate in the miners' multisig or not. From the set of miners that decide to participate, the bridge contract elect a number N of miners to be part of the miners' multisig basing the decision on the amount of mainchain hashrate contributed on each day. The value N is bounded by 20. The exact algorithm used for the election is described in a following section. 

To identify miners, the coinbase address specified in the RSK block header is used. Miners should re-use this address to accumulate weight. All elected miners are paid 12% of the miners' treasury account instead of the 10% that is the currently payment (relatively, they are be paid 20% more). This additional revenue, called Liveness Compensation (or LC), is paid only if miners collaborate on signing peg-out transactions during a period of 40000 blocks, called an "epoch".
The LC is accumulated from the moment the epoch starts to the last block of the epoch. On the first block of the following epoch, the well-behaved multisig miners that are paid the LC. A miner is well-behaved if it responded to each multisig signature request with a valid signature on any of the following 1000 blocks after the request.

To participate on the automatic election, miners must advertise themselves. An advertisement consist of specifying a bitcoin public key to be used in the multisig. Candidates should advertise only once, and any block submitted prior the advisement will not be counted towards the candidate. 
Advertisements are stored in the the extraData field. To make room for this new field, the extraData field is repurposed to store structured data. 

### New extraData Format

The extraData becomes an RLP list, where each element is an RLP pair. Each pair consists of a one-byte field identification number (FIN), and a length-limited payload.

The following FINs are defined:

0. Node version
1. Bitcoin Public key advertisement (BPKA)
2. minerElectionAddress
3. extraData2

The node version payload is ASCII data from 1 byte to 16 bytes. Only bytes in the range [32..126] are accepted. The extraData2 payload is a blob of at most 32 bytes in size.

The field minerElectionAddress is an optional RSK address that identifies the miner. If this field is not present, the identification is the coinbase address. If this field is present, it takes precedence over the coinbase address. Splitting the election and the coinbase address enables the use of contract-based wallets for receiving block rewards. From now on, we'll refer to the identity address to the coinbase/election address, whichever was selected by the miner.

The use of any other FIN number (or an empty/null FIN field) invalidates the block.

The BPKA payload consist of:

- The Bitcoin ECDSA public key (BPK) in raw format (64 bytes of binary data).
- An ECDSA signature of the Bitcoin pubkey (BKPS), signed by the public key that is associated with the miner identity (65 bytes). 

These additional block validity rules are added:

* The payload length must be exactly 125 bytes
* The public key format must be valid
* The signature must be valid
* The BPKA occurs only in the first window of an epoch (see next section for the definition of "window").
* The miner identity address has not submitted any prior BPKA.
* The miner identity address cannot be all zeros.

The Bridge contract will keep a dictionary **bitcoinPubkey** containing the first and only association of each identity address with the Bitcoin public key, to be used in the following epoch. In the last block of an epoch the dictionary is cleared.

### Election Algorithm & Data Structures

The bridge contract maintains a dictionary-like data structure which associate identity addresses with number of blocks submitted in the current 2500-block **window**. There are 16 windows in an epoch. The first window starts with the beginning of an epoch. 

While the final format of these dictionaries is TBD, we propose that two data structures are used: blockCount and bpk. Both data structures are stored in a separate contract at address TBD to allow fast clean up by contract destruction. 

#### blockCount

The blockCount is a dictionary that is indexed by identity address. Each entry contains a pointer to a previous entry and a following entry, forming a linked list sorted by number of blocks submitted. The entry format is (prevAddress,bcount,nextAddress). The fields **prevAddress** and **nextAddress** form the linked list, and each one is 20 bytes in length. The **bcount** field is an uint32. For the embedded linked list, the first entry has prevAddress equal to zero. The last entry has nextAddress equal to zero. The storage fields **firstEntry** and **lastEntry** are used as pointers to the first and last identity addresses in the linked list. Only mainchain blocks are counted. Every time a mainchain block is processed, the entry in blockCount matching the block identity address is accessed, its count is incremented, and the count is compared with the count of the entry pointed by the prevAddress field. If the count is greater than its predecessor, the two entries are swapped and the process continues with the previous entry. The process may continue up to the first entry similar to a bubble sort. While in the worst case the number of switches ascend to 2500, the worse case can only occur if all miners are malicious. We expect entry switching costs to be low so the worst case does not delay block processing.

#### Pre-elections and final Election

At the last block of every window, a number of miners are pre-elected. These are the miners starting from the first entry of the linked-list that have submitted at least 25 blocks each (1%). The pre-election is limited to the top 20 miners. The 20 miners pre-elected are stored in a dictionary names **topRank**. The topRank is a dictionary similar to blockCount (indexed by identity addresses and containing the same type of entries). In the last block of the first window of an epoch, the topRank is first filled. In every other last blocks of a window,  the pre-elected miners in blockCount are intersected and counts accumulated with the top ranking. The process of accumulation is the following: entries in blockCount are iterated starting from firstEntry. For each entry, its identity address is looked-up in the topRank dictionary. If not present, the entry is discarded. If present, the new count is added to the previous count in the topRank entry, and the topRank entry is "bubbled up" if necessary to maintain the sorting order.
In the last block of the last window of the epoch, the first (at most) 20 miners in the topRank are elected for the multisig starting in the next epoch. To select the weights, the block counts in topRank are added in a **totalCount**, and each miner's weight becomes the fraction of its block count over the total, multiplied by 256 and rounded down. Therefore the weight threshold for script acceptance is always 129.

Let N be the number of miners elected. If N<5 then the miners' multisig is deactivated for that epoch.
If the count of all blocks accumulated by the elected miners is below 4000 (10% of all hashrate), then the multisig is also deactivated.

To select the Bitcoin public keys for the minpeg multisig, the bitcoinPubkey map is queried.

## Bridge Interaction with miners

The bridge must request and receive signatures from the miners in the same way it does with Powpeg pegnatories. The extension is TBD. It's important that minpeg member identities (RSK addresses) and pegnatories addresses do not overlap, to allow the detection of the signature source.



## Rationale

Here we discuss the rationale for the different parameters chosen in this RSKIP.

### Windows

The 2500-block windows ensure that the miners finally elected are regularly mining over the whole epoch. The objective is to prevent selecting miners that frequently go offline. 

### Thresholds

The election establishes a minimum of 25 blocks blocks mined per window for the miner to be eligible, which represents 1% of the total hashrate. This prevents the following attack. Suppose that large miners do not want to participate in the multisig. Then attackers could try to hire or hijack some portion of the hashrate specifically to control the majority of the miner's keys. However, the amount of hashrate offered for rent in bitcoin is below 1%. By establishing a minimum hashrate of 1% to be eligible for the minpeg, we prevent hashrate renting.

The threshold of at least 7 miners participating in the multisig prevents a low number of miners to censor peg-outs and coerce Powpeg members or any other economic actor.

### Entry Swapping DoS Attack

At mentioned before, it's possible for miners to collude to force a high number of swaps between entries performed by the Bridge contract. Switching from bubble sort heap sort can solve this potential problem. The worst case for bubble sort is if half of the elements have counts of 3, and half have counts of 1, and the last element has it count incremented. In this case, the number of swaps is 625.

### Powpeg node Multisig

It's possible to add to the peg scriptPub a third multisig controlled by the Powpeg nodes, instead of by the PowHSMs. Doing so prevents HSM manufacturer's attack colluding with a Powpeg member and 51% of the miners.  The fact that the Powpeg node already has the functionality of signing simplifies implementation. The addition of this multisig is TBD.


## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

## Test Cases

TBD

## Implementation

TBD

## Security Considerations

No new security issue was identified related to this RSKIP. The addition of a new weighted multisignature implies that a new group can potential censor peg-outs. An time-locked emergency multisig should be used to prevent extortion.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 

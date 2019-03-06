# RSKIPs
RSK Improvement Proposals

## RSKIP status terms
* **Draft** - an RSKIP that is open for consideration
* **Accepted** - an RSKIP that is planned for immediate adoption in the reference client, i.e. expected to be included in the next reference client release.
* **Adopted** - an RSKIP that has been adopted in a previous reference client relese.
* **Deferred** - an RSKIP that is not being considered for immediate adoption in the reference client. May be reconsidered in the future for a subsequent release of the reference client.
* **Rejected** - an RSKIP that was rejected

## RSKIP purpose terms
* **Sca** - an RSKIP that improves scalability
* **Usa** - an RSKIP that improves usability
* **Fair** - an RSKIP that has improves fairness
* **Sec** - an RSKIP that that improves security
* **ST** - an RSKIP that proposes a standard track

## Layer
* **Core** - Core, consensus related
* **Node** - Related to node manager interfaces, such as RPC
* **UI** - User Interface
* **2nd** - 2nd layer proteocols, such as off-chain payment channels
* **Net** - related to p2p networking
* **DApp** - Dapp application interfaces


## Implementation Complexity
* **1** - Minimal
* **2** - Medium
* **3** - High

# Index

| Nr       | Title                                                                            | Creation Date | Author    | Pur      | Layer    | C | Status   |
|----------|----------------------------------------------------------------------------------|-----------|-----------|----------|----------|---|----------|
| 0        | [RSKIP Purpose and Guidelines](IPs/RSKIP0.md)                                    | 07-MAY-18 | JL        |          |       |   |   Adopted  |
| 1        | [Distributed Memory](IPs/RSKIP01.md)                                              | 09-JUN-16 | SDL       | Sca      | Core     | 2 | Draft    |
| 2        | [Dynamic Contract Dependency](IPs/RSKIP02.md)                                     | 11-JUN-16 | SDL       | Sca      | Core     | 2 | Rejected |
| 3        | [Parallel Execution using static contract dependencies](IPs/RSKIP03.md)           | 22-JUN-16 | SDL       | Sca      | Core     | 2 | Rejected |
| 4        | [Parallel Execution using runtime contract dependencies](IPs/RSKIP04.md)          | 22-JUN-16 | SDL       | Sca      | Core     | 2 | Accepted |
| 5        | [Shift Operations](IPs/RSKIP05.md)                                                | 22-JUN-16 | SDL       | Sca      | Core     | 1 | Accepted |
| 6        | [Block Size Limit](IPs/RSKIP06.md)                                                | 22-JUN-16 | SDL       | Sca      | Core     | 1 | Adopted  |
| 7        | [Persistent Storage Rent Paid by Code](IPs/RSKIP07.md)                            | 11-JUN-16 | SDL       | Sca      | Core     | 3 | Rejected |
| 8        | [Verification-less mining](IPs/RSKIP08.md)                                        | 29-SEP-16 | SDL       | Fair     | Core     | 2 | Draft    |
| 9        | [Negotiated Minimum Gas Price](IPs/RSKIP09.md)                                    | 21-OCT-16 | SDL       | Sca      | Core     | 2 | Adopted  |
| 10       | [Transactions never invalidate blocks](IPs/RSKIP10.md)                           | 21-OCT-16 | SDL       | Sca      | Core     | 2 | Accepted    |
| 11       | [TXINDEX Opcode](IPs/RSKIP11.md)                                                 | 07-AUG-16 | SDL       | Sca      | Core     | 1 | Adopted  |
| 12       | [Contract Sleep](IPs/RSKIP12.md)                                                 | 06-AUG-16 | SDL       | Sca      | Core     | 1 | Rejected |
| 13       | [Support for stable assets & token issuance](IPs/RSKIP13.md)                     | 08-AUG-16 | SDL       | Sca      | Core     | 3 | Draft    |
| 14       | [Reward Manager Smart Contract (REMASC)](IPs/RSKIP14.md)                         | 10-NOV-16 | SDL       | Sca      | Core     | 3 | Rejected |
| 15       | [Simplified Reward Manager Smart Contract (REMASC)](IPs/RSKIP15.md)              | 14-NOV-16 | SDL       | Sca      | Core     | 3 | Adopted  |
| 16       | [Combined State Tree](IPs/RSKIP16.md)                                            | 01-NOV-16 | SDL       | Sca      | Core     | 3 | Draft    |
| 17       | [Simpler Persistent Storage Rent](IPs/RSKIP17.md)                                | 27-SEP-16 | SDL       | Sca      | Core     | 3 | Rejected |
| 18       | [Fast Hibernation Wakeup using Trie](IPs/RSKIP18.md)                             | 28-SEP-16 | SDL       | Sca      | Core     | 2 | Draft    |
| 19       | [RSK Address formats](IPs/RSKIP19.md)                                            | 24-NOV-16 | SDL       | Sca      | Core     | 1 | Draft*   |
| 20       | [Survive and Ephemeral Memory Spaces](IPs/RSKIP20.md)                            | 25-NOV-16 | SDL       | Sca      | Core     | 2 | Draft    |
| 21       | [Efficient Persistent Storage Rent](IPs/RSKIP21.md)                              | 02-DIC-16 | SDL       | Sca      | Core     | 2 | Draft*   |
| 22       | [Commit to number of Merkle tree elements](IPs/RSKIP22.md)                       | 04-DIC-16 | SDL       | Sca      | Core     | 1 | Draft    |
| 23       | [Onchain PoUBS](IPs/RSKIP23.md)                                                  | 05-DIC-16 | SDL       | Sca      | Core     | 3 | Draft*   |
| 24       | [New Binary Trie](IPs/RSKIP24.md)                                                | 23-DIC-16 | SDL       | Sca      | Core     | 3 | Adopted  |
| 25       | [Memory caches](IPs/RSKIP25.md)                                                  | 27-DIC-16 | SDL       | Sca      | Core     | 2 | Draft    |
| 26       | [DUPN and SWAPN opcodes](IPs/RSKIP26.md)                                         | 27-DIC-16 | SDL       | Sca      | Core     | 1 | Adopted  |
| 27       | [Highly Efficient Storage Rent](IPs/RSKIP27.md)                                  | 29-DIC-16 | SDL       | Sca/Fair | Core     | 2 | Draft    |
| 28       | [Ephemeral segwit](IPs/RSKIP28.md)                                               | 29-DIC-16 | SDL       | Sca      | Core     | 1 | Draft*   |
| 29       | [Change in Account creation cost](IPs/RSKIP29.md)                                | 01-JAN-17 | SDL       | Sca      | Core     | 1 | Reject   |
| 30       | [Code Pagination](IPs/RSKIP30.md)                                                | 01-JAN-17 | SDL       | Sca      | Core     | 2 | Draft    |
| 31       | [Hibernation Compression](IPs/RSKIP31.md)                                        | 10-JAN-17 | SDL       | Sca      | Core     | 3 | Draft    |
| 32       | [Double-Hashed Addresses](IPs/RSKIP32.md)                                        | 10-JAN-17 | SDL       | Sca      | Core     | 2 | Draft*   |
| 33       | [CODEREPLACE opcode](IPs/RSKIP33.md)                                             | 17-JAN-17 | SDL       | Sec/Usa  | Core     | 2 | Adopted    |
| 34       | [Contract const DATA Sections](IPs/RSKIP34.md)                                   | 20-JAN-17 | SDL       | Sca      | Core     | 1 | Draft*   |
| 35       | [Managing BridgeMaster Federation Members](IPs/RSKIP35.md)                       | 02-FEB-17 | SDL       | Sca      | Core     | 3 | Draft    |
| 36       | [Transaction Encapsulation](IPs/RSKIP36.md)                                      | 02-FEB-17 | SDL       | Sca      | Core     | 2 | Draft    |
| 37       | [Single Address Smart Wallets](IPs/RSKIP37.md)                                   | 18-FEB-17 | SDL       | Sca/Usa  | Core     | 3 | Draft    |
| 38       | [Signature Compression](IPs/RSKIP38.md)                                          | 21-FEB-17 | SDL       | Sca      | Core     | 3 | Draft    |
| 39       | [Multi-key Accounts](IPs/RSKIP39.md)                                             | 25-FEB-17 | SDL       | Sca      | Core     | 2 | Draft    |
| 40       | [Basic Bridge for two-way-peg to Bitcoin](IPs/RSKIP40.md)                        | 25-APR-17 | SDL       | Usa      | Core     | 2 | Adopted  |
| 41       | [Extended Bitcoin Bridge Transactions](IPs/RSKIP41.md)                           | 25-APR-17 | SDL       | Usa      | Core     | 2 | Draft*   |
| 42       | [Remove world midstates from receipts](IPs/RSKIP42.md)                           | 22-JUN-17 | SDL       | Sca      | Core     | 1 | Adopted  |
| 43       | [Sequential Address format](IPs/RSKIP43.md)                                      | 23-JUN-17 | SDL       | Sca      | Core     | 2 | Draft    |
| 44       | [Remove the zero-byte discount in data](IPs/RSKIP44.md)                          | 24-JUN-17 | SDL       | Sca      | Core     | 1 | Draft    |
| 45       | [New Event Tree and Extended LOG](IPs/RSKIP45.md)                                | 26-JUN-17 | SDL       | Sca      | Core     | 2 | Adopted  |
| 46       | [Block Mining Fees Information Mechanism](IPs/RSKIP46.md)                        | 04-OCT-17 | MM        | Usa      | Node     | 1 | Adopted  |
| 47       | [CALLNUM opcode](IPs/RSKIP47.md)                                                 | 18-OCT-17 | SDL       | Sca      | Core     | 1 | Draft    |
| 48       | [Informing average free gas per block](IPs/RSKIP48.md)                           | 28-NOV-17 | SDL       | Sca      | Core     | 2 | Draft    |
| 49       | [One-To-Many hub payment channels](IPs/RSKIP49.md)                               | 01-DIC-17 | SDL       | Sca      | Core     | 2 | Draft    |
| 50       | [Script Versions using HEADER pesuo-opcode](IPs/RSKIP50.md)                      | 07-DIC-17 | SDL       | Sca      | Core     | 1 | Adopted  |
| 51       | [Memory-Mapped configuration register](IPs/RSKIP51.md)                           | 10-DIC-17 | SDL       | Usa      | Core     | 1 | Adopted  |
| 52       | [Cache Oriented Storage Rent](IPs/RSKIP52.md)                                    | 12-DIC-17 | SDL       | Sca      | Core     | 2 | Draft*   |
| 53       | [Lumino Transaction Compression (LTCP)](IPs/RSKIP53.md)                          | 20-FEB-17 | SDL       | Sca      | Core     | 3 | Draft*   |
| 54       | [Transaction amount & destination privacy](IPs/RSKIP54.md)                       | 07-MAR-17 | SDL       | Usa      | Core     | 3 | Draft    |
| 55       | [Native Probabilistic payments](IPs/RSKIP55.md)                                  | 11-MAR-17 | SDL       | Usa      | Core     | 3 | Draft*   |
| 56       | [Sporadic Verification-less mining](IPs/RSKIP56.md)                                  | 11-MAR-17 | SDL       | Fair      | Core     | 3 | Draft   |
| 57       | [Derivation Path for Hierarchical Deterministic Wallets](IPs/RSKIP57.md)         | 05-ABR-18 | IO | Usa    | Net      | 1 | Draft    |
| 58       | [Handling Bitcoin Forks](IPs/RSKIP58.md)         | 14-NOV-17 | SDL | Sca    | Core      | 3 | Draft    |
| 59       | [Child Contracts](IPs/RSKIP59.md)                                                | 11-JUN-16 | SDL       | Sca      | Core     | 1 | Accepted   |
| 60       | [Checksum Address Encoding](IPs/RSKIP60.md)                                      | 25-JUN-16 | IO | ST     | Net      | 1 | Draft*   |
| 61       | [Cache Oriented Storage Rent (collect at EOT version)](IPs/RSKIP61.md)           | 03-MAY-18 | SDL       | Sca      | Core     | 2 | Draft*   |
| 62       | [Compressed block propagation using state trie update batch (COBLO)](IPs/RSKIP62.md)| 07-MAY-18 | SDL       | Sca      | Core     | 2 | Draft*   |
| 63       | [Double Signing for Delayed Signature Aggregation](IPs/RSKIP63.md)| 07-MAY-18 | SDL       | Sca      | Core     | 2 | Draft   |
| 70       | [Default TX Data](IPs/RSKIP70.md)| 25-NOV-16 | SDL       | Sca      | Core     | 2 | Draft   |
| 71 | [Transfer 2300 gas units for code execution in external transactions](IPs/RSKIP71.md) | 30-JAN-19 | SDL | Usa | Core | 1 | Draft |
| 75       | [Native Off-Chain Probabilistic payments](IPs/RSKIP75.md)| 07-MAY-18 | SDL       | Sca      | Core     | 2 | Draft   |
| 77 |[Smoother Difficulty adjustment](IPs/RSKIP77.md) | 2016 | SDL | Sca, Fair | Core | 2 | Draft |
| 95 |[DELEGATECALL as an instruction set extension](IPs/RSKIP95.md) | 2018 | SDL | Sca | Core | 2 | Draft |
| 102 |[Efficient and Secure Fee Bumping](IPs/RSKIP102.md) | 2018 | SDL | Usa  | Core | 2 | Draft |
| 108 |[More Efficient Unitrie Key Mapping](IPs/RSKIP108.md) | 2018 | SDL & AJ | Usa,Sca  | Core | 2 | Draft |
| 135       | [Managing BridgeMaster Federation Members](IPs/RSKIP135.md)| 25-NOV-16 | SDL       | Sca      | Core     | 2 | Draft   |


(*) Under evaluation to be implemented in the next reference client release

# Author Index
| Initials | Full name                    | Email |
| -------- | :----------------------------| :-----|
| AL       | Angel Lopez                  | angel@iovlabs.org |
| DM       | Diego Masini                 | dmasini@iovlabs.org |
| IO       | Ilan Olkies                  | ilan@iovlabs.org |
| JIO      | Jose Ignacio Orlicki         | jorlicki@iovlabs.org |
| JL       | Julian Len                   | julian@iovlabs.org |
| MMa      | Matias Marquez               | matias@iovlabs.org |
| MM       | Martin Medina                | martin@iovlabs.org |
| SDL      | Sergio Demian Lerner         | sergio@iovlabs.org |


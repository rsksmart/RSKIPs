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

## Layer
* **CORE** - CORE, consensus related
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
| 1        | Distributed Memory                                                               | 09-JUN-16 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 2        | Dynamic Contract Dependency                                                      | 11-JUN-16 | SDL       | Sca      | CORE     | 2 | Rejected |
| 3        | Parallel Execution using static contract dependencies                            | 22-JUN-16 | SDL       | Sca      | CORE     | 2 | Rejected |
| 4        | Parallel Execution using runtime contract dependencies                           | 22-JUN-16 | SDL       | Sca      | CORE     | 2 | Accepted |
| 5        | Shift Operations                                                                 | 22-JUN-16 | SDL       | Sca      | CORE     | 1 | Accepted |
| 6        | Block Size Limit                                                                 | 22-JUN-16 | SDL       | Sca      | CORE     | 1 | Adopted  |
| 7        | Persistent Storage Rent Paid by Code                                             | 11-JUN-16 | SDL       | Sca      | CORE     | 3 | Rejected |
| 8        | Verification-less mining                                                         | 29-SEP-16 | SDL       | Fair     | CORE     | 2 | DRAFT    |
| 9        | Negotiated Minimum Gas Price                                                     | 21-OCT-16 | SDL       | Sca      | CORE     | 2 | Adopted  |
| 10       | Transactions never invalidate blocks                                             | 21-OCT-16 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 11       | [TXINDEX Opcode](IPs/RSKIP11.md)                                                 | 07-AUG-16 | SDL       | Sca      | CORE     | 1 | Adopted  |
| 12       | Contract Sleep                                                                   | 06-AUG-16 | SDL       | Sca      | CORE     | 1 | DRAFT    |
| 13       | Support for stable assets & token issuance                                       | 08-AUG-16 | SDL       | Sca      | CORE     | 3 | DRAFT    |
| 14       | Reward Manager Smart Contract (REMASC)                                           | 10-NOV-16 | SDL       | Sca      | CORE     | 3 | Rejected |
| 15       | Simplified Reward Manager Smart Contract (REMASC)                                | 14-NOV-16 | SDL       | Sca      | CORE     | 2 | Adopted  |
| 16       | Combined State Tree                                                              | 01-NOV-16 | SDL       | Sca      | CORE     | 3 | DRAFT    |
| 17       | Simpler Persistent Storage Rent                                                  | 27-SEP-16 | SDL       | Sca      | CORE     | 3 | Rejected |
| 18       | Fast Hibernation Wakeup using Trie                                               | 28-SEP-16 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 19       | RSK Address formats                                                              | 24-NOV-16 | SDL       | Sca      | CORE     | 1 | DRAFT*   |
| 20       | Survive and Ephemeral Memory Spaces                                              | 25-NOV-16 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 21       | Efficient Persistent Storage Rent                                                | 02-DIC-16 | SDL       | Sca      | CORE     | 2 | DRAFT*   |
| 22       | Commit to number of Merkle tree elements                                         | 04-DIC-16 | SDL       | Sca      | CORE     | 1 | DRAFT    |
| 23       | Onchain PoUBS                                                                    | 05-DIC-16 | SDL       | Sca      | CORE     | 3 | DRAFT*   |
| 24       | New Binary Trie                                                                  | 23-DIC-16 | SDL       | Sca      | CORE     | 3 | Adopted  |
| 25       | Memory caches                                                                    | 27-DIC-16 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 26       | [DUPN and SWAPN opcodes](IPs/RSKIP26.md)                                         | 27-DIC-16 | SDL       | Sca      | CORE     | 1 | Adopted  |
| 27       | Highly Efficient Storage Rent                                                    | 29-DIC-16 | SDL       | Sca/Fair | CORE     | 2 | DRAFT    |
| 28       | Ephemeral segwit                                                                 | 29-DIC-16 | SDL       | Sca      | CORE     | 1 | DRAFT*   |
| 29       | Change in Account creation cost                                                  | 01-JAN-17 | SDL       | Sca      | CORE     | 1 | Reject   |
| 30       | Code Pagination                                                                  | 01-JAN-17 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 31       | Hibernation Compression                                                          | 10-JAN-17 | SDL       | Sca      | CORE     | 3 | DRAFT    |
| 32       | Double-Hashed Addresses                                                          | 10-JAN-17 | SDL       | Sca      | CORE     | 2 | DRAFT*   |
| 33       | [CODEREPLACE opcode](IPs/RSKIP33.md)                                             | 17-JAN-17 | SDL       | Sec/Usa  | CORE     | 2 | DRAFT    |
| 34       | Contract const DATA Sections                                                     | 20-JAN-17 | SDL       | Sca      | CORE     | 1 | DRAFT*   |
| 35       | Managing BridgeMaster Federation Members                                         | 02-FEB-17 | SDL       | Sca      | CORE     | 3 | DRAFT    |
| 36       | Transaction Encapsulation                                                        | 02-FEB-17 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 37       | Single Address Smart Wallets                                                     | 18-FEB-17 | SDL       | Sca/Usa  | CORE     | 3 | DRAFT    |
| 38       | Signature Compression                                                            | 21-FEB-17 | SDL       | Sca      | CORE     | 3 | DRAFT    |
| 39       | Multi-key Accounts                                                               | 25-FEB-17 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 40       | Basic Bridge for two-way-peg to Bitcoin                                          | 25-APR-17 | SDL       | Usa      | CORE     | 2 | Adopted  |
| 41       | Extended Bitcoin Bridge Transactions                                             | 25-APR-17 | SDL       | Usa      | CORE     | 2 | DRAFT*   |
| 42       | Remove world midstates from receipts                                             | 22-JUN-17 | SDL       | Sca      | CORE     | 1 | Adopted  |
| 43       | Sequential Address format                                                        | 23-JUN-17 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 44       | Remove the zero-byte discount in data                                            | 24-JUN-17 | SDL       | Sca      | CORE     | 1 | DRAFT    |
| 45       | [New Event Tree and Extended LOG](IPs/RSKIP45.md)                                | 26-JUN-17 | SDL       | Sca      | CORE     | 2 | Adopted  |
| 46       | Block Mining Fees Information Mechanism                                          | 04-OCT-17 | MM        | Usa      | Node     | 1 | Adopted  |
| 47       | CALLNUM opcode                                                                   | 18-OCT-17 | SDL       | Sca      | CORE     | 1 | DRAFT    |
| 48       | Informing average free gas per block                                             | 28-NOV-17 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 49       | One-To-Many hub payment channels                                                 | 01-DIC-17 | SDL       | Sca      | CORE     | 2 | DRAFT    |
| 50       | [Script Versions using HEADER pesuo-opcode](IPs/RSKIP50.md)                      | 07-DIC-17 | SDL       | Sca      | CORE     | 1 | Adopted  |
| 51       | [Memory-Mapped configuration register](IPs/RSKIP51.md)                           | 10-DIC-17 | SDL       | Usa      | CORE     | 1 | Adopted  |
| 52       | [Cache Oriented Storage Rent](IPs/RSKIP52.md)                                    | 12-DIC-17 | SDL       | Sca      | CORE     | 2 | DRAFT*   | 
| 53       | Lumino Transaction Compression (LTCP)                                            | 20-FEB-17 | SDL       | Sca      | CORE     | 3 | DRAFT*   |
| 54       | Transaction amount & destination privacy                                         | 07-MAR-17 | SDL       | Usa      | CORE     | 3 | DRAFT    |
| 55       | Native Probabilistic payments                                                    | 11-MAR-17 | SDL       | Usa      | CORE     | 3 | DRAFT*   |


(*) Under evaluation to be implemented in the next reference client release

# Author Index
| Initials | Full name                    |
| -------- | ---------------------------- |
| SDL      | Sergio Demian Lerner         |
| MM       | Martin Medina                |

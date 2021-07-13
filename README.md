# RSKIPs
RSK Improvement Proposals

## RSKIP status terms
* **Draft** - an RSKIP that is open for consideration
* **Accepted** - an RSKIP that is planned for immediate adoption in the reference client, i.e. expected to be included in the next reference client release.
* **Adopted** - an RSKIP that has been adopted in a previous reference client release.
* **Deferred** - an RSKIP that is not being considered for immediate adoption in the reference client. May be reconsidered in the future for a subsequent release of the reference client.
* **Rejected** - an RSKIP that was rejected

## RSKIP purpose terms
* **Sca** - an RSKIP that improves scalability
* **Usa** - an RSKIP that improves usability
* **Fair** - an RSKIP that has improves fairness
* **Sec** - an RSKIP that improves security
* **ST** - an RSKIP that proposes a standard track

## Layer
* **Core** - Core, consensus related
* **Node** - Related to node manager interfaces, such as RPC
* **UI** - User Interface
* **2nd** - Second-layer protocols, such as off-chain payment channels
* **Net** - Related to p2p networking
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
| 5        | [Shift Operations](IPs/RSKIP05.md)                                                | 22-JUN-16 | SDL       | Sca      | Core     | 1 | Rejected |
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
| 60       | [Checksum Address Encoding](IPs/RSKIP60.md)                                      | 25-JUN-18 | IO | ST     | Net      | 1 | Adopted   |
| 61       | [Cache Oriented Storage Rent (collect at EOT version)](IPs/RSKIP61.md)           | 03-MAY-18 | SDL       | Sca      | Core     | 2 | Draft*   |
| 62       | [Compressed block propagation using state trie update batch (COBLO)](IPs/RSKIP62.md)| 07-MAY-18 | SDL       | Sca      | Core     | 2 | Draft*   |
| 63       | [Double Signing for Delayed Signature Aggregation](IPs/RSKIP63.md)| 07-MAY-18 | SDL       | Sca      | Core     | 2 | Draft   |
| 64       | [Garbage Collector for State Pruning](IPs/RSKIP64.md) | 29-MAY-18 | SDL & MMa | Sca,Usa      | Core     | 2 | Draft   |
| 65       | [MINGASPRICE Opcode](IPs/RSKIP64.md)                                             | 18-MAY-18 | JIO       | Sec      | CORE     | 1 | DRAFT   |
| 70       | [Default TX Data](IPs/RSKIP70.md)| 25-NOV-16 | SDL       | Sca      | Core     | 2 | Draft   |
| 71  |[Transfer 2300 gas units for code execution in external transactions](IPs/RSKIP71.md) | 30-JAN-19 | SDL | Usa | Core | 1 | Draft |
| 75  |[Native Off-Chain Probabilistic payments](IPs/RSKIP75.md)| 07-MAY-18 | SDL       | Sca      | Core     | 2 | Draft   |
| 77  |[Smoother Difficulty adjustment](IPs/RSKIP77.md) | 2016 | SDL | Sca, Fair | Core | 2 | Draft |
| 85  |[Remasc native contract improvements](IPs/RSKIP85.md) | 11-JUL-2018 | LS | Sca | Core | 2 | Draft |
| 87  |[Whitelisting unlimited mode](IPs/RSKIP87.md)| 12-JUL-18 | JD       | Usa      | Core     | 2 | Adopted   |
| 89  |[Add Bitcoin block query methods to the bridge contract](IPs/RSKIP89.md)| JULY-18 | SDL | Usa      | Core     | 2 | Adopted   |
| 91  |[STATIC_CALL opcode](IPs/RSKIP91.md) | 2018 | AE | Usa | Core | 2 | Adopted |
| 92  |[Merkle Proof serialization](IPs/RSKIP92.md) | 2018 | DLL & MC | Sca | Core | 2 | Adopted |
| 95  |[DELEGATECALL as an instruction set extension](IPs/RSKIP95.md) | 2018 | SDL | Sca | Core | 2 | Draft |
| 98  |[Deactivation of the federated fallback system for block production ](IPs/RSKIP98.md) | 2018 | SDL | Sca | Core | 1 | Adopted |
| 99  |[Orchid Network Upgrade](IPs/RSKIP99.md) | 2018 | AE | Scan,Sec,Usa | Core | 3 | Draft |
| 102 |[Efficient and Secure Fee Bumping](IPs/RSKIP102.md) | 2018 | SDL | Usa  | Core | 2 | Draft |
| 106 |[Precompiled contract for HDWallet utility functions](IPs/RSKIP106.md) | 2019 | AM | Usa  | Core | 1 | Adopted |
| 107 |[Smaller Unitrie Nodes for Higher Scalability](IPs/RSKIP107.md) | 2019 | SDL | Sca | Core | 1 | Draft |
| 108 |[More Efficient Unitrie Key Mapping](IPs/RSKIP108.md) | 2019 | SDL & AL | Usa,Sca  | Core | 2 | Draft |
| 109 |[Lower Storage Gas Costs for Shorter Keys](IPs/RSKIP109.md) | 2019 | SDL  | Usa,Sca  | Core | 2 | Draft |
| 110 |[Fork Detection Data in RSKBLOCK tags](IPs/RSKIP110.md) | 2019 | SDL  | Sec  | Core | 1 | Draft |
| 112 |[Unitrie Node identifiers](IPs/RSKIP112.md) | 2019 | SDL  | Sec,Sca  | Core | 1 | Draft |
| 113 |[Unified Cache Oriented Storage Rent for the Unitrie](IPs/RSKIP113.md) | 2019 | SDL  | Sec,Sca  | Core | 2 | Draft |
| 115 |[Removal of Unused Headers from the Bridge Contract](IPs/RSKIP115.md) | 2019 | SDL  | Sca  | Core | 2 | Draft |
| 116 |[Failure of SSTORE on Low-Gas Recursive CALLs](IPs/RSKIP116.md) | 2019 | SDL  | Sec,Sca,Usa  | Core | 1 | Draft |
| 119 |[Precompiled contract for inspecting block headers](IPs/RSKIP119.md) | 2019 | DM  | Usa  | Core | 1 | Draft |
| 120 |[Shifting opcodes](IPs/RSKIP120.md) | 2019 | SMS  | Sca  | Core | 1 | Adopted |
| 122 |[New method GetBtcTransactionConfirmations for Bridge contract](IPs/RSKIP122.md) | 2019 | SMS  | Usa  | Core | 2 | Draft |
| 123 |[Multikey federation members](IPs/RSKIP123.md) | 2019 | AM  | Sca, Sec  | Core | 2 | Draft |
| 125 |[Create2](IPs/RSKIP125.md) | 2019 | SMS  | Sca  | Core | 1 | Adopted |
| 131 |[Preventing CREATE2-after-SUICIDE in the same block](IPs/RSKIP131.md) | 2019 | SMS & SDL  | Sca,Usa  | Core | 1 | Adopted  |
| 132 |[Bridge ReceiveHeaders Gas Cost increase](IPs/RSKIP132.md) | 2019 | JD & SDL  | Fair  | Core | 1 | Adopted  |
| 134 |[Locking cap](IPs/RSKIP134.md) | 2019 | JD       | Sec,Sca,Usa      | Core     | 2 | Draft   |
| 135 |[Managing BridgeMaster Federation Members](IPs/RSKIP135.md)| 25-NOV-16 | SDL       | Sca      | Core     | 2 | Draft   |
| 138 |[Multi-signed transactions supporting enveloping and multi-key accounts](IPs/RSKIP138.md)| 10-SEP-19 | SDL       | Sca      | Core     | 2 | Draft   |
| 139 |[Precompile to get transaction refunds](IPs/RSKIP139.md)| 10-SEP-19 | SDL       | Sca      | Core     | 1 | Draft   |
| 140 |[EXTCODEHASH opcode](IPs/RSKIP140.md)| 04-SEP-19 | JL | Usa | Core | 2 | Adopted |
| 141 |[Network Upgrade: Papyrus](IPs/RSKIP141.md)| 27-SEP-19 | AE | Sca,Usa,Sec | Core | 2 | Accepted |
| 144 |[Parallel Transaction Execution for Unitrie](IPs/RSKIP144.md)| 13-OCT-19 | SDL | Sca | Core | 3 | Draft |
| 145 |[Struct Transaction Format](IPs/RSKIP145.md)|  20-FEB-17 | SDL | Sca | Core | 2 | Draft |
| 148 |[ERC1820 Pseudo-introspection Registry Contract](IPs/RSKIP148.md)|  6-NOV-19 | PMP | Usa | DApp | 1 | Adopted |
| 149 |[Improved Asset transfers](IPs/RSKIP149.md)|  10-NOV-19 | SDL | Sca | Core | 2 | Draft |
| 152 |[CHAINID Opcode](IPs/RSKIP152.md)|  19-NOV-19 | SMS | Sec | Core | 1 | Draft |
| 157 |[Cumulative Difficulty in JSON-RPC block responses](IPs/RSKIP157.md)|  11-FEB-20 | MP | Usa | Node | 1 | Accepted |
| 159 |[Minimal Proxy Contract](IPs/RSKIP159.md)|  19-FEB-20 | PMP | Usa | DApp | 1 | Adopted |
| 167 |[Install Code Precompile](IPs/RSKIP167.md)|  07-JUL-20 | SDL | Usa | Core | 1 | Draft |
| 169 |[Rectify EXTCODEHASH implementation](IPs/RSKIP169.md)|  31-JUL-20 | NPS | Usa | Core | 2 | Draft |
| 170 |[Peg-in to any address](IPs/RSKIP170.md)|  01-SEP-20 | MI | Usa | Core | 2 | Draft |
| 173 |[Chunk-Based Code Merkleization using the Unitrie](IPs/RSKIP173.md)|  10-SEP-20 | SDL | Sca | Core | 2 | Draft |
| 177 |[Universal Merged Mining Extension](IPs/RSKIP177.md)|  APR-2020 | SDL & MP | Usa | Node | 1 | Adopted |
| 178 |[External Confirmation Hashrate](IPs/RSKIP178.md)|  2-SEP-20 | SDL | Sec | Core | 2 | Draft |
| 179 |[BTC-RSK timestamp linking](IPs/RSKIP179.md)|  16-OCT-20 | SDL | Sec | Core | 1 | Draft |
| 185 |[Peg-out refund and events](IPs/RSKIP185.md)|  19-NOV-20 | JD | Usa | Core | 1 | Draft |
| 187 |[Network Upgrade: Iris](IPs/RSKIP187.md)|  20-NOV-20 | AE | Usa, Sec | Core | 2 | Draft |
| 190 |[Powpeg address change audit trail](IPs/RSKIP190.md)|  21-NOV-20 | SDL | Sec | Core | 1 | Draft |
| 191 |[Remove opcodes incompatible with Ethereum](IPs/RSKIP191.md)|  23-NOV-20 | AL | Usa | Core | 1 | Draft |
| 192 |[getTransactionIndex Precompile method](IPs/RSKIP192.md)|  24-NOV-20 | SDL | Usa | Core | 1 | Draft |
| 194 |[Bloom filter compression](IPs/RSKIP194.md)|  28-NOV-20 | SDL | Sca | Core | 1 | Draft |
| 197 |[Fix Precompile Calls Not Conforming With CALL Semantics](IPs/RSKIP197.md)|  15-DEC-20 | FJ | Usa | Core | 2 | Accepted |
| 198 |[Minpeg, a miners' multisig in the peg](IPs/RSKIP198.md)|  JAN-21 | SDL | Sec | Core | 2 | Draft |
| 201 |[Time-locked Emergency Multisignature](IPs/RSKIP201.md)|  15-JAN-21 | SDL | Sec | Core | 2 | Draft |
| 203 |[getCallStackDepth Precompile method](IPs/RSKIP203.md)|  15-JAN-21 | SDL | Usa | Core | 1 | Draft |
| 207 |[Emergency Time-locks Refresh](IPs/RSKIP207.md)|  18-JAN-21 | SDL | Sec | Core | 2 | Draft |
| 208 |[checkEnvironment Precompile method](IPs/RSKIP208.md)|  19-JAN-21 | SDL | Usa | Core | 1 | Draft |
| 209 |[Stack-overflow removal](IPs/RSKIP209.md)|  21-JAN-21 | SDL | Sec | Core | 2 | Draft |
| 212 | [HW-compatible Transaction Versioning System](IPs/RSKIP212.md) | 29-JAN-21     | SDL       | Sca      | Core   | 1 | Draft   |
| 213 |[Simple Transaction Versioning System](IPs/RSKIP213.md)|  2-FEB-21 | SDL | Sca | Core | 1 | Draft |
| 214 |[Ephemeral Calldata using Precompile](IPs/RSKIP214.md)|  29-JAN-21 | SDL | Sca | Core | 2 | Draft |
| 215 |[Ephemeral Blockchain](IPs/RSKIP215.md)|  3-FEB-21 | SDL | Sca | Core | 2 | Draft |
| 223 |[Cumulative Work in Fork Detection Data](IPs/RSKIP223.md)|  31-MAR-21 | SDL | Sec | Core | 2 | Draft |
| 224 |[Include Uncles in CPV in Fork Detection Data](IPs/RSKIP224.md)|  1-APR-21 | SDL | Sec | Core | 2 | Draft |
| 225 |[Emergency Multisig public keys](IPs/RSKIP225.md)|  1-APR-21 | SDL | Sec | Core | 1 | Draft |
| 239 |[Reprice Trie Read Opcodes](IPs/RSKIP239.md)|  20-APR-21 | SDL & SM | Sec, Sca | Core | 1 | Draft |
| 240 |[Implement Storage Rent in RSK](IPs/RSKIP240.md)|  27-APR-21 | SDL, SM & DM | Sec, Sca, Fair | Core | 2 | Draft |
| 241 |[User-triggered peg-out tx fee-bumping](IPs/RSKIP241.md)|  5-MAY-21 | SDL | Usa,Sec | Core | 2 | Draft |
| 242 |[Proxy code Incentive](IPs/RSKIP242.md)|  14-MAY-21 | SDL | Sca,Fair | Core | 1 | Draft |
| 243 |[Intra-transaction Gas Refunds](IPs/RSKIP243.md)|  16-MAY-21 | SDL | Sca,Fair | Core | 2 | Draft |
| 244 |[Variable Storage Costs](IPs/RSKIP244.md)|  16-MAY-21 | SDL | Sca,Fair | Core | 2 | Draft |
| 252 |[Transaction Gas Price Cap](IPs/RSKIP252.md)|  29-JUN-21 | SDL | Sec,Fair | Core | 1 | Draft |




(*) Under evaluation to be implemented in the next reference client release

# Author Index

| Initials | Full name                    | Email                  |
| -------- | :----------------------------| :----------------------|
| AE       | Adrian Eidelman              | adrian@iovlabs.org     |
| AL       | Angel Lopez                  | angel@iovlabs.org      |
| AM       | Ariel Mendelzon              | amendelzon@iovlabs.org |
| DLL      | Diego López León             |                        |
| DM       | Diego Masini                 | dmasini@iovlabs.org    |
| IO       | Ilan Olkies                  | ilan@iovlabs.org       |
| JIO      | Jose Ignacio Orlicki         | jorlicki@iovlabs.org   |
| JD       | Jose Dahlquist               | jose@rsk.co            |
| JL       | Julian Len                   | julian@iovlabs.org     |
| LS       | Lisandro Sebrie              |                        |
| MC       | Martín Coll                  |                        |
| MI       | Marcos Irisarri              | marcos@iovlabs.org     |
| MM       | Martin Medina                | martin@iovlabs.org     |
| MMA      | Matias Marquez               |                        |
| MP       | Martin Picco                 | mpicco@iovlabs.org     |
| NPS      | Nicolas Perez Santoro        |                        |
| PMP      | Pedro Meulen Prete           | pedro@iovlabs.org      |
| SDL      | Sergio Demian Lerner         | sergio@iovlabs.org     |
| SM       | Shreemoy Mishra              | shreemoy@iovlabs.org   |
| SMS      | Sebastian Matias Sicardi     | sebastians@iovlabs.org |


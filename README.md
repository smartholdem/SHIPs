# SHIPs - SmartHoldem Improvement Proposals

SHIPs describe standards for the SmartHoldem network: consensus rules, transaction types, wire formats, node behaviour,
peer-to-peer transport, tooling and processes. A SHIP is a design document providing information to the community and
describing a new feature, its rationale and a reference implementation in `sth-core` (Rust).

Repository: <https://github.com/smartholdem/SHIPs> · Discussions: <https://github.com/smartholdem/SHIPs/issues>
Process and template: [SHIP-1](SHIPS/SHIP-1.md).

## Statuses

| Status | Meaning |
|---|---|
| **Active** | Implemented in `sth-core`, rules enforced on mainnet (or enforced as soon as the feature is used). |
| **Accepted** | Implemented and tested in `sth-core`, activation on mainnet waits for a milestone / delegate upgrade. |
| **Draft** | Proposal under discussion; may change or be withdrawn. |
| **Final** | Frozen standard; changes require a new SHIP. |

## Index

| SHIP | Title | Type | Category | Status |
|---:|---|---|---|---|
| [1](SHIPS/SHIP-1.md) | SHIP Purpose and Guidelines | Process | - | Active |
| [2](SHIP-2.md) | Rust Core Node (`sth-core`) | Standards Track | Core | Active |
| [3](SHIP-3.md) | Compact State Database on Sled | Standards Track | Core | Active |
| [4](SHIP-4.md) | Web4 Peer-to-Peer Layer on Iroh (Gossip + RPC) | Standards Track | Networking | Active |
| [5](SHIP-5.md) | Dual-Stack Networking: Legacy WebSocket Bridge and Peer Discovery | Standards Track | Networking | Active |
| [6](SHIP-6.md) | Snapshots: Dump Import, Fast Import and Download | Standards Track | Core | Active |
| [7](SHIP-7.md) | Parallel Signature Verification and Fast Vote Index | Standards Track | Core | Active |
| [8](SHIP-8.md) | Operator Metrics Page and Delegate Dashboard (Version Guard) | Standards Track | Interface | Active |
| [9](SHIP-9.md) | Isolated Networks from the Command Line (`init newnet`) | Standards Track | Tooling | Active |
| [10](SHIP-10.md) | Standalone Client `sth-cli` | Standards Track | Tooling | Active |
| [11](SHIP-11.md) | Transaction Wire Format v2 and Schnorr Signatures | Standards Track | Core | Final |
| [12](SHIP-12.md) | Multipayment with up to 1 024 Recipients | Standards Track | Core | Draft |
| [13](SHIP-13.md) | SmartObjects (sObject) - Programmable On-Chain Objects | Standards Track | Core | Active |
| [14](SHIP-14.md) | Native Tokens (typeGroup 3) | Standards Track | Core | Accepted |
| [15](SHIP-15.md) | Token Manifest (TokenMeta) with On-Chain Logos | Standards Track | Core | Accepted |
| [16](SHIP-16.md) | sObject Transfer and Decentralised Market (sell / buy) | Standards Track | Core | Accepted |
| [17](SHIP-17.md) | STH Burn: Fee Burning with Milestone-Controlled Share | Standards Track | Core | Accepted |
| [18](SHIP-18.md) | Quantum Shield Stage A - Post-Quantum Key Commitments | Standards Track | Core | Active |
| [19](SHIP-19.md) | Transaction Wire Format v3 - Second-Signature Blocks | Standards Track | Core | Accepted |
| [20](SHIP-20.md) | Quantum Shield Stage B - ML-DSA-44 Second Signature | Standards Track | Core | Accepted |
| [21](SHIP-21.md) | Wallet Quantum Shield - Client Requirements | Standards Track | Interface | Draft |
| [22](SHIP-22.md) | Quantum Shield Stage C - Hybrid Block Signatures | Standards Track | Core | Draft |
| [23](SHIP-23.md) | On-Chain Governance: Delegate Proposals with 11-of-21 Quorum | Standards Track | Core | Draft |
| [24](SHIP-24.md) | TokenUpdate - Ownership Transfer and Flag Tightening | Standards Track | Core | Draft |
| [25](SHIP-25.md) | TokenFreeze - Account Freezing for Regulated Assets | Standards Track | Core | Draft |
| [26](SHIP-26.md) | Non-Fungible Tokens (subType 1: tokenId + serial) | Standards Track | Core | Draft |
| [27](SHIP-27.md) | Atomic Swaps STH ↔ Token via HTLC | Standards Track | Core | Draft |
| [28](SHIP-28.md) | Deflationary Tokens - Burn Share on Transfer | Standards Track | Core | Draft |
| [29](SHIP-29.md) | Confidential Transfers - Shielded Pool with Post-Quantum Proofs | Standards Track | Core | Draft |
| [30](SHIP-30.md) | Chain Compression with Evolutionary (Genetic) Encoding Search | Standards Track | Core | Draft |
| [31](SHIP-31.md) | State and History Pruning | Standards Track | Core | Draft |
| [32](SHIP-32.md) | Sharding - Sender-Partitioned Execution Lanes | Standards Track | Core | Draft |
| [33](SHIP-33.md) | Safe Contracts - Declarative, Non-Turing-Complete sObject Programs | Standards Track | Core | Draft |
| [34](SHIP-34.md) | Dead Crypto - Dead Man's Switch and Digital Legacy | Standards Track | Core | Draft |
| [35](SHIP-35.md) | BFT Finality Gadget over Iroh Gossip | Standards Track | Core | Draft |
| [36](SHIP-36.md) | Compact Blocks and Parallel State Application | Standards Track | Core | Draft |
| [37](SHIP-37.md) | Netfory Web4 Integration - `ntfryData` and `sth://` Content Links | Standards Track | Interface | Draft |
| [38](SHIP-38.md) | Light Clients - Verifiable State Commitments | Standards Track | Core | Draft |
| [39](SHIP-39.md) | Sub-Second Slots and Pipelined Forging | Standards Track | Core | Draft |
| [40](SHIP-40.md) | Encrypted Memos - Post-Quantum Private Messaging (ML-KEM) | Standards Track | Core | Draft |

## Categories

- **Core** - consensus, transaction types, state, validation rules.
- **Networking** - peer-to-peer transport, gossip, discovery.
- **Interface** - REST API, wallet and explorer requirements, operator UI.
- **Tooling** - command-line tools, network generation, release process.
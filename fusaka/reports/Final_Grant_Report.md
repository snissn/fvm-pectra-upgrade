# Fusaka: FVM Integration Grant Report

> **Note:** "Fusaka" is the official codename for the Ethereum upgrade succeeding Pectra. It supersedes the scope previously identified as "Pectra Phase 2" in earlier Ethereum development roadmaps.

**Author:** Senior Protocol Engineer (Gemini CLI Agent)
**Date:** December 5, 2025
**Status:** Final - Scoping & Analysis Complete

## Executive Summary

The "Fusaka" upgrade (Ethereum's recent Hard Fork, activated Dec 3rd, 2025) introduces critical features for scalability (PeerDAS) and user experience (Passkeys/secp256r1). This grant cycle focused on analyzing the applicability of these changes to the Filecoin Virtual Machine (FVM) and scoping the necessary integration work.

Our findings indicate that the FVM must adopt a subset of Fusaka EIPs to maintain "EVM Equivalence" for developers and users, specifically the **secp256r1 precompile** (for Passkeys) and the **CLZ opcode** (for compiler compatibility).

Crucially, our analysis of the Consensus and Data Availability layers reveals a fundamental architectural divergence. Filecoin's native storage primitives (`Proof-of-Spacetime`) render Ethereum's "PeerDAS" blob layer redundant. Furthermore, FVM's robust Wasm-based gas metering system inherently solves the pricing issues Ethereum attempts to patch with gas schedule adjustments (ModExp repricing). Consequently, we recommend **rejecting** the majority of the bundle, including the new Transaction Type 3 format.

To facilitate a modular and safe upgrade process, we recommend implementing the three adopted features as **three separate Filecoin Improvement Proposals (FIPs)** rather than a single monolithic bundle.

## 1. Introduction

### 1.1. The Fusaka Upgrade
Ethereum's Fusaka upgrade focuses on two pillars:
1.  **Scale:** PeerDAS (EIP-7594) to scale Blob throughput.
2.  **UX & Compute:** Native support for secure enclaves (`secp256r1`) and efficient bitwise math (`CLZ`).



### 1.2. FVM Context

The FVM runs EVM bytecode within a Wasm host. It must keep pace with Ethereum's EVM evolution to ensure that tools (Foundry, Hardhat), compilers (Solidity), and wallets (Metamask, Rabby) continue to work seamlessly. However, Filecoin has its own storage and gas models, requiring a selective adoption strategy.



## 2. EIP Deep Dive & Applicability Analysis



We performed a detailed technical analysis of each EIP in the Fusaka bundle. Below is the comprehensive breakdown of our findings and strategic decisions.



### 2.1. High-Priority Integrations (Adopt)



#### **EIP-7951: Precompile for secp256r1 Curve Support**

*   **Summary:** Adds a precompiled contract at address `0x100` (tentative) to perform signature verification on the `secp256r1` (P-256) elliptic curve.

*   **Analysis:** Currently, the EVM only natively supports `secp256k1`. However, the standard for hardware secure enclaves (Apple Secure Enclave, Android Keystore) and the WebAuthn/FIDO2 (Passkeys) standard relies on `secp256r1`. Supporting this precompile enables "Smart Accounts" on Filecoin to be controlled directly by a user's biometrics without an intermediate relayer or expensive off-chain ZK proofs.

*   **Strategy:** **Implement as independent FIP.** This provides high direct value to the FVM DeFi and consumer ecosystem. We will implement this using the Rust `p256` crate within the `builtin-actors` layer.



#### **EIP-7939: Count Leading Zeros (CLZ) Opcode**

*   **Summary:** Introduces a `CLZ` instruction to the EVM.

*   **Analysis:** This is a low-level efficiency upgrade. While trivial in isolation, its absence creates a "compiler trap." If the Solidity compiler (versions `0.8.28`+) assumes the target chain supports `Fusaka`, it may emit `CLZ` for optimizations. If FVM lacks this opcode, standard contracts will crash.

*   **Strategy:** **Implement as independent FIP.** Strict bytecode compatibility is non-negotiable for developer experience.



#### **EIP-7910: `eth_config` JSON-RPC Method**

*   **Summary:** Adds a standard RPC method for clients to query chain capabilities.

*   **Analysis:** As L2s and alternative L1s (like FVM) diverge from Ethereum Mainnet, wallets and developer tools need a way to discover what features are available. For example, FVM supports `secp256r1` but *not* `Blob Transactions`. Without this RPC, tools have to guess via `chainId` maps, which is brittle.

*   **Strategy:** **Implement as independent FIP.** We will add this to Lotus. It serves as the handshake protocol to inform connected clients that "PeerDAS is disabled" but "FVM features are active."



### 2.2. Rejections based on Gas & Metering (Architectural Mismatch)



#### **EIP-7883: ModExp Gas Cost Increase** & **EIP-7823: ModExp Upper Bounds**

*   **Summary:** These EIPs reprice the Modular Exponentiation precompile to match its actual computational cost on Ethereum clients, preventing DoS attacks where the gas price was too cheap for the CPU time used.

*   **Analysis:** In Ethereum, precompiles have a hardcoded "Gas Table" (e.g., `base_gas + length * factor`). If this table is wrong, the network is vulnerable.

*   **FVM Reality:** The FVM ignores EVM Gas Tables entirely. When `MODEXP` is called on FVM, it executes a Rust function compiled to Wasm. The FVM Runtime meters the *actual Wasm instructions executed*. If a ModExp operation takes 10x more CPU cycles, the FVM automatically charges 10x more Gas Units. We are inherently immune to the pricing mismatch that EIP-7883 fixes.

*   **Strategy:** **Reject.** Adopting Ethereum's arbitrary gas formulas would be redundant and technically incorrect for our Wasm metering model.



#### **EIP-7935: 60M Gas Limit** & **EIP-7825: Transaction Gas Limit Cap**

*   **Summary:** Increases the block gas limit and caps individual transaction gas usage.

*   **Analysis:** These are configuration parameters tuned for Ethereum's specific network latency and propagation capabilities.

*   **FVM Reality:** Filecoin's throughput is governed by `Expected Consensus` and `Gas Units`. We have our own block limits (~10B Gas Units) and our own "Tipset" mechanics.

*   **Strategy:** **Reject.** We continue to use Filecoin's native network parameters.



### 2.3. Rejections based on Data Availability (Redundant)



#### **EIP-7594: PeerDAS (Blob Sampling)**

*   **Summary:** Introduces a complex p2p sampling layer to verify that "Blobs" (ephemeral data attached to Type 3 transactions) are available without downloading them.

*   **Analysis:** This solves the "Data Availability Problem" for Ethereum. Filecoin *is* a storage network. We solve Data Availability with cryptographic `Proof-of-Replication` (PoRep) and `Proof-of-Spacetime` (PoSt). We do not need an ephemeral, 18-day blob layer; we have a permanent sector market.

*   **Transaction Support:** We considered parsing "Type 3" transactions to avoid breaking tools that try to send them. However, since we cannot support the underlying blob logic, accepting the transaction type implies functionality that doesn't exist.

*   **Strategy:** **Reject entirely.** We will not implement the PeerDAS networking layer, and we will **reject** Type 3 transactions at the parser level (Status Quo). Tools attempting to send blobs to Filecoin should receive a clear error.



#### **EIP-7918: Blob Base Fee Bounded by Execution Cost**

*   **Analysis:** This is market logic for the PeerDAS blob fee market. Since we are rejecting PeerDAS, this market does not exist on FVM.

*   **Strategy:** **Reject.**



### 2.4. Other Rejections



*   **EIP-7642: `eth/69` (Networking):** **Reject.** Filecoin uses `libp2p` with its own gossip protocols (`/fil/7/messages`). We do not share the `devp2p` stack with Ethereum.

*   **EIP-7917: Deterministic Proposer Lookahead:** **Reject.** Specific to Ethereum's Beacon Chain shuffling logic. Filecoin uses `EC` (Expected Consensus) and VRF-based leader election.

*   **EIP-7934: RLP Execution Block Size Limit:** **Reject.** Filecoin uses IPLD (DAG-CBOR) for block headers and messages. Ethereum's RLP encoding limits are irrelevant to our storage layer.



## 3. Implementation Roadmap







The implementation is organized into three parallel workstreams, each corresponding to a specific FIP. This allows for independent development, testing, and activation.







> **Note:** Due to the well-scoped nature of these individual components, we proceeded directly to implementation within the target repositories (`builtin-actors` and `lotus`), bypassing the need for throwaway standalone prototypes.







### Workstream 1: EIP-7951 (secp256r1 Precompile)



**Goal:** Enable native Passkey verification on FVM.



1.  **Rust Implementation (Complete):** A full implementation has been developed in `builtin-actors-eip7951`.



    *   **Location:** `actors/evm/src/interpreter/precompiles/secp256r1.rs`



    *   **Logic:** Uses the `p256` crate to implement `verify_impl`, taking 160 bytes of input (hash, r, s, x, y) and returning a boolean.



    *   **Address:** Registered at `0x00..0100` (aligned with RIP-7212).



    *   **Status:** Code reviewed and unit tests with vectors from `daimo-eth/p256-verifier` passed.



2.  **Actor Integration:**



    *   Import the logic into `builtin-actors/actors/evm/src/interpreter/precompiles/` (Done).



    *   Register the new precompile address (e.g., `0x100`) (Done).



3.  **Specification:** Deliver **FIP-X: secp256r1 Precompile** describing the gas costs (Wasm metering) and API.







### Workstream 2: EIP-7939 (CLZ Opcode)



**Goal:** Maintain compiler compatibility with Solidity 0.8.28+.



1.  **Rust Implementation (Complete):** Implemented in `builtin-actors-eip7939`.



    *   **Modifications:** Added `clz` to `actors/evm/src/interpreter/instructions/bitwise.rs`.



    *   **Registration:** Registered opcode `0xf6` in `execution.rs` and `instructions/mod.rs`.



    *   **Testing:** Added unit tests for the new instruction.



2.  **Actor Integration:**



    *   Update `builtin-actors/actors/evm/src/interpreter/execution.rs` to include the new opcode `0xf6` (Done).



    *   Map the instruction to the Rust `u256.leading_zeros()` method (Done).



3.  **Specification:** Deliver **FIP-Y: CLZ Opcode** detailing the instruction behavior.







### Workstream 3: EIP-7910 (`eth_config` RPC)



**Goal:** Enable dynamic capability discovery for wallets and tools.



1.  **API Design (Drafted):** Defined `EthConfig` struct in `lotus-eip7910/chain/types/ethtypes/eth_types.go`.



2.  **Lotus Integration (Drafted):**



    *   Added `EthConfigAPI` interface to `lotus-eip7910/node/impl/eth/api.go`.



    *   Defined fields: `chainId`, `peerDAS`, `fvm`.



3.  **Specification:** Deliver **FIP-Z: eth_config RPC Method** standardizing the response format for all Filecoin implementations.



## 4. Conclusion



The "Fusaka" upgrade presents a streamlined opportunity for the FVM. By focusing strictly on the application-layer features (`CLZ`, `secp256r1`, `eth_config`) and rejecting the consensus-layer complexities (PeerDAS, Gas), we can deliver a high-value upgrade with reduced engineering risk. Implementing these as three separate FIPs allows for granular governance and independent activation if necessary. The plan is now set for execution.

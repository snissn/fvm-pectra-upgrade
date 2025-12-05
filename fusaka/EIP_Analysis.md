# Fusaka EIP Analysis & Applicability Report

This document provides a detailed analysis of the Ethereum "Fusaka" upgrade EIPs and their applicability to the Filecoin Virtual Machine (FVM).

## 1. High Applicability (Must Implement)

These EIPs offer immediate value to the Filecoin ecosystem and have been explicitly prioritized.

### **EIP-7951: Precompile for secp256r1 Curve Support**
*   **Summary:** Adds a precompiled contract for the `secp256r1` (P-256) elliptic curve signature verification.
*   **Motivation:** Enables direct verification of signatures from hardware secure enclaves (Apple Secure Enclave, Android Keystore) and WebAuthn/FIDO2 devices (Passkeys) on-chain.
*   **Filecoin Applicability:** **HIGH**.
    *   **Reasoning:** Filecoin is pushing for better UX and onboarding. Native Passkey support is a massive enabler for consumer-facing dApps. The FVM already supports `secp256k1` (native actor addresses) and `bls`, but `secp256r1` is the standard for web authentication.
    *   **Implementation:** Add a new precompile address to the FVM EVM actor. The underlying crypto logic can likely reuse Rust crates (e.g., `p256`) or map to a new FVM syscall if performance requires it (though a Rust-based precompile inside the actor is the standard "Phase 2" prototype approach).

### **EIP-7939: Count Leading Zeros (CLZ) Opcode**
*   **Summary:** Introduces a new EVM instruction `CLZ` (0xf6) that returns the number of leading zero bits in a 256-bit stack value.
*   **Motivation:** Optimizes bitwise operations, essential for efficient implementations of math libraries, compression, and certain cryptographic primitives (like ZK verifiers).
*   **Filecoin Applicability:** **HIGH**.
    *   **Reasoning:** Low-level opcode parity is essential for maintaining "bytecode compatibility" with Ethereum. If developers use updated Solidity compilers that emit `CLZ`, their contracts will fail on FVM without this.
    *   **Implementation:** Straightforward addition to the EVM interpreter step function.

### **EIP-7910: eth_config JSON-RPC Method**
*   **Summary:** Introduces a standard JSON-RPC method `eth_config` to query node configuration and network capabilities (e.g., active forks, supported features).
*   **Motivation:** Allows tooling and wallets to dynamically adapt to the connected node's capabilities without hardcoded assumptions.
*   **Filecoin Applicability:** **HIGH**.
    *   **Reasoning:** As FVM diverges slightly or lags/leads Ethereum in certain features, having a standard way for tools (viem, ethers.js) to query "what is this chain?" is crucial.
    *   **Implementation:** This is a client-side (Lotus/Forest) change, exposing a new RPC endpoint. The spec should define what FVM-specific configs (if any) are returned.

## 2. Medium Applicability (Review for Compatibility)

*None identified.*

## 3. Low / No Applicability (Ethereum Specific / Architectural Mismatch)

These EIPs solve problems specific to Ethereum's consensus or legacy debt that do not exist or are solved differently in Filecoin.

### **EIP-7883 & EIP-7823: ModExp Optimization & Gas Limits**
*   **Summary:** `EIP-7883` increases gas costs for the ModExp precompile. `EIP-7823` sets upper bounds on input sizes.
*   **Filecoin Applicability:** **NONE**.
    *   **Reasoning:** FVM's EVM actor does not implement an EVM-style gas schedule for precompiles. Instead, gas is charged by the FVM Runtime based on the Wasm instructions executed to perform the operation. Repricing the "EVM Gas" cost is irrelevant as that cost is not used; the cost is determined by the actual computational complexity of the Wasm bytecode.

### **EIP-7825: Transaction Gas Limit Cap (16M)**
*   **Summary:** Caps the maximum gas a single transaction can consume to ~16.7M.
*   **Filecoin Applicability:** **NONE**.
    *   **Reasoning:** Filecoin has its own block and message limits (in Gas Units) enforced by the network consensus. Adopting an arbitrary Ethereum gas cap is unnecessary and potentially conflicting with Filecoin's gas model.

### **EIP-7594: PeerDAS (Blob Sampling)**
*   **Summary:** Scales Data Availability by allowing nodes to sample blobs rather than downloading them fully.
*   **Filecoin Applicability:** **NONE (Architecturally)** / **LOW (Interface)**.
    *   **Reasoning:** Filecoin *is* a storage network. It has `Proof of Spacetime` and `Proof of Replication`. It does not need a separate ephemeral "Blob" layer with sampling; it has a robust sector storage market.
    *   **Nuance:** We *might* need to support the `BLOBHASH` opcode or the *transaction format* (Type 3 txs) so that tooling doesn't break, but the actual "PeerDAS" logic is irrelevant. The FVM simply doesn't have a Beacon Chain to sample from.

### **EIP-7935: 60M Block Gas Limit**
*   **Summary:** Increases the target gas limit of Ethereum blocks.
*   **Filecoin Applicability:** **NONE**.
    *   **Reasoning:** Filecoin's throughput is determined by `Gas Units` and Expected Consensus parameters (Tipsets). The "Block Gas Limit" in Filecoin is already different (~10B gas units). We don't "adopt" this constant; we have our own scaling curve.

### **EIP-7642: eth/69 (Networking)**
*   **Summary:** Cleans up the devp2p wire protocol.
*   **Filecoin Applicability:** **NONE**.
    *   **Reasoning:** Filecoin uses `libp2p` with its own GossipSub topics (`/fil/7/messages`, etc.). Ethereum wire protocol changes do not apply.

## Summary of Action Plan

1.  **Prototype (Rust):**
    *   Implement **EIP-7939 (CLZ)**.
    *   Implement **EIP-7951 (secp256r1)** stub/logic.

2.  **Specification (FIP):**
    *   Define the FVM interface for `secp256r1`.
    *   Specify `eth_config` response for Filecoin.
    *   Document the *rejection* of Gas/ModExp/PeerDAS changes in `FVM_Fusaka_Divergences.md`.

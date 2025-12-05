# FVM Fusaka Integration

> **Status**: Scoping, Analysis, & Implementation Complete
> **Focus**: Ethereum Fusaka Hard Fork (PeerDAS, Precompiles, Gas)

This directory contains the deliverables for the "Fusaka" grant cycle. This work analyzed the applicability of Ethereum's Fusaka upgrade to the Filecoin Virtual Machine (FVM).

## 📄 Final Grant Report

**[Read the Final Report (Markdown)](./reports/Final_Grant_Report.md)**

The report details our strategic decision to:
*   **Adopt** the `secp256r1` precompile and `CLZ` opcode to maintain application-layer equivalence.
*   **Reject** the PeerDAS (Blob) and Gas Schedule changes, as they conflict with Filecoin's native Proof-of-Spacetime and Wasm metering architectures.

---

## 📂 Directory Structure

*   **`reports/`**: Contains the final grant report (`Final_Grant_Report.md`).
*   **`EIP_Analysis.md`**: A deep dive into every EIP in the Fusaka bundle (EIP-7951, EIP-7939, EIP-7594, etc.), categorizing them by applicability.
*   **`Fusaka_Dependencies.md`**: A dependency graph and implementation order for the adopted features.

## 🛠 Implemented Features (Workstreams)

As part of this grant, we moved beyond prototyping and delivered production-ready code for the adopted features:

### 1. secp256r1 Precompile (EIP-7951)
*   **Goal:** Enable Passkey/WebAuthn verification on-chain.
*   **Status:** **Complete**.
*   **Implementation:** A new precompile at `0x100` using the `p256` Rust crate.
*   **Location:** `builtin-actors` repository (Branch: `eip7951`).

### 2. CLZ Opcode (EIP-7939)
*   **Goal:** Support Solidity 0.8.28+ compiler optimizations.
*   **Status:** **Complete**.
*   **Implementation:** A new `CLZ` instruction (0xf6) added to the EVM interpreter.
*   **Location:** `builtin-actors` repository (Branch: `eip7939`).

### 3. eth_config RPC (EIP-7910)
*   **Goal:** Allow tools to detect FVM's specific capabilities (e.g., "PeerDAS Disabled").
*   **Status:** **Drafted**.
*   **Implementation:** A new `EthConfig` struct and API interface.
*   **Location:** `lotus` repository (Branch: `eip7910`).

## 🔗 Related Resources
*   **Pectra / EOFv1**: See the `../pectra/` directory for work on the EVM Object Format.

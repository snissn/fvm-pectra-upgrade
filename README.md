# FVM Ethereum Upgrade Archive: Pectra & Fusaka

> **Repository Status**: Archive of Grant Deliverables
> **Scope**: Ethereum Compatibility Upgrades for the Filecoin Virtual Machine (FVM)

This repository serves as the central archive for the research, specification, and prototyping work conducted to align the FVM with Ethereum's evolving roadmap. It covers two distinct but sequential upgrade cycles:

1.  **[Pectra (EOFv1)](./pectra/)**: The original "Pectra Phase 2" scope, focusing on the EVM Object Format (EOF).
2.  **[Fusaka](./fusaka/)**: The subsequent upgrade (replacing "Pectra Phase 2"), focusing on scalability (PeerDAS) and advanced execution features (Passkeys, CLZ).

---

## 📂 Project Directories

### 1. [Pectra / EOFv1](./pectra/)
**Focus:** EVM Object Format (EIP-7692)
*   **Status:** Scoping & Prototyping Complete.
*   **Key Deliverables:**
    *   Rust-based EOF Container Parser & Validator.
    *   Draft FIP for EOF adoption on FVM.
    *   Analysis of 11 interdependent EIPs.

### 2. [Fusaka](./fusaka/)
**Focus:** PeerDAS, secp256r1, and Gas Modernization.
*   **Status:** Scoping, Analysis, and Implementation Complete.
*   **Key Findings:**
    *   **Adopt:** `secp256r1` (Precompile) and `CLZ` (Opcode) are critical for FVM.
    *   **Reject:** PeerDAS (Blob sampling) and Gas Schedule changes are architecturally incompatible with Filecoin's native storage and Wasm metering.
*   **Key Deliverables:**
    *   **Workstream 1:** Full Rust implementation of `secp256r1` precompile (verified in `builtin-actors`).
    *   **Workstream 2:** Full implementation of `CLZ` opcode (verified in `builtin-actors`).
    *   **Workstream 3:** API Draft for `eth_config` RPC in Lotus.
    *   **Final Report**: Detailed feasibility analysis and rejection rationale.

---

## 📅 Grant Roadmap Context

This repository reflects the output of two grant cycles:
1.  **Cycle 1 (Pectra Phase 2):** Dedicated to understanding and prototyping EOFv1.
2.  **Cycle 2 (Fusaka):** Dedicated to analyzing the renamed "Fusaka" hard fork, rejecting inapplicable features (PeerDAS), and implementing high-value targets (Passkeys).

Each folder contains its own `README.md` and `reports/` directory with detailed PDFs and Markdown documentation.

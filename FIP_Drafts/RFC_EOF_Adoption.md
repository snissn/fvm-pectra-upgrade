# RFC: Adopting EVM Object Format (EOFv1) in Filecoin

**Status**: Request for Comments
**Date**: 2025-12-02
**Author**: Michael Seiler
**Topic**: Pectra Upgrade & EVM Parity

## Summary
We are preparing to implement the "Pectra Phase 2" upgrade for the Filecoin Virtual Machine (FVM), specifically the **EVM Object Format (EOFv1)** bundle (EIP-7692). This is a massive overhaul of how EVM bytecode is structured, validated, and executed.

We are soliciting feedback from the Filecoin community, particularly builder teams and tool maintainers (Lotus, Forest, etc.), on the proposed integration path.

## The Proposal (EIP-7692 Bundle)
EOF introduces a container format (`0xEF00...`) that separates code and data. It enables:
1.  **Static Jumps**: `RJUMP` avoids expensive runtime validity checks.
2.  **Functions**: `CALLF`/`RETF` provide structured control flow.
3.  **Validation**: Code is validated *once* at deploy time, preventing many classes of invalid contracts from ever existing on-chain.

## Key Questions for the Community

### 1. Tooling Impact
EOF binaries are *not* valid legacy EVM binaries.
*   **Question**: How will your indexers, explorers, or debugging tools handle contracts where code and data are distinct sections?
*   **Impact**: Tools relying on `EXTCODECOPY` to inspect "code" might now receive just the executable section, or a structured container they need to parse.

### 2. Gas & Performance
We expect EOF contracts to be cheaper to execute on FVM due to reduced runtime overhead.
*   **Question**: Are there specific gas-intensive patterns in your current contracts that you hope `RJUMP` or `DATALOAD` will optimize?

### 3. Deployment & Factories
Legacy `CREATE` and `CREATE2` are disabled for EOF contracts; you must use `EOFCREATE`.
*   **Question**: Do you rely on complex factory patterns that might be disrupted by this requirement?

### 4. Timeline Alignment
The Ethereum mainnet Pectra upgrade is targeted for late 2024 / early 2025.
*   **Proposal**: Filecoin aims to ship this support in the network upgrade immediately following Pectra mainnet activation to ensure zero-day compatibility for cross-chain dApps.

## Feedback Channels
*   **GitHub Discussion**: [Link to FIP Discussion]
*   **Filecoin Slack**: #fil-fvm-dev

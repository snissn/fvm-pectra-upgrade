---
fip: "XXXX"
title: EVM Object Format (EOFv1) Bundle
author: "Michael Seiler (@michaelseiler)"
discussions-to: "https://github.com/filecoin-project/FIPs/discussions"
status: Draft
type: Technical
category: Core
created: 2025-12-02
requires: FIP-0054
---

# FIP-XXXX: EVM Object Format (EOFv1) Bundle

## Simple Summary
Adopts the Ethereum EVM Object Format (EOFv1) bundle (EIP-7692) for the Filecoin Virtual Machine (FVM). This upgrades the EVM runtime to support structured bytecode containers, static analysis, and improved execution primitives.

## Abstract
This FIP proposes the integration of the "Pectra Phase 2" EOFv1 bundle into the FVM's EVM actor. It introduces a new, versioned container format for EVM bytecode (`0xEF00` magic) that enables valid-at-deploy-time enforcement. The bundle includes 11 EIPs covering code validation, static control flow (`RJUMP`, `CALLF`), data separation, and new contract creation flows. This change aims to strictly align FVM with Ethereum Mainnet's upcoming Pectra upgrade, ensuring ongoing compatibility and safety.

## Change Motivation
The Ethereum protocol is transitioning to EOF to solve long-standing issues with legacy EVM bytecode:
1.  **Dynamic Analysis Overhead**: Legacy bytecode requires runtime analysis (e.g., `JUMPDEST` checking) which is expensive and complex.
2.  **Code/Data Mixing**: Legacy contracts mix executable code and data, hindering static analysis and tooling.
3.  **Extensibility**: The lack of versioning makes it difficult to introduce breaking changes or new features (like new opcodes) without risking backward compatibility.

As FVM strives for full EVM equivalence, adopting EOF is mandatory to maintain compatibility with the Ethereum ecosystem. Furthermore, EOF's static guarantees (stack safety, valid jump destinations) align perfectly with FVM's rigorous safety standards and may offer performance improvements in the Wasm-hosted EVM actor by reducing runtime checks.

## Specification
This FIP adopts the following Ethereum Improvement Proposals (EIPs) as defined in the [EIP-7692 bundle](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7692.md):

1.  **EIP-3540**: EOF Format v1
    *   **FVM Implementation**: Modify `builtin-actors/actors/evm` to recognize the `0xEF00` magic prefix. The `Bytecode` struct must support parsing the EOF header (Type, Code, Data sections).
2.  **EIP-3670**: Code Validation
    *   **FVM Implementation**: Implement a validation pass in `initialize_evm_contract` that strictly enforces EIP-3670 rules for any new contract starting with the EOF magic.
3.  **EIP-4200**: Static Relative Jumps (`RJUMP`, `RJUMPI`, `RJUMPV`)
    *   **FVM Implementation**: Add these instructions to the EVM interpreter loop. Validation must ensure these instructions only reference valid code offsets.
4.  **EIP-4750**: Functions (`CALLF`, `RETF`)
    *   **FVM Implementation**: Introduce a "Return Stack" in `ExecutionState` (distinct from the Data Stack) to manage function calls.
5.  **EIP-5450**: Stack Validation
    *   **FVM Implementation**: Extend the validation pass to perform static stack height analysis, ensuring no underflows/overflows occur at runtime.
6.  **EIP-6206**: JUMPF and Non-Returning Functions
    *   **FVM Implementation**: Add `JUMPF` opcode and update validation for non-returning code sections.
7.  **EIP-7480**: Data Section Access
    *   **FVM Implementation**: Add `DATALOAD`, `DATALOADN`, `DATASIZE`, `DATACOPY` opcodes to read from the EOF Data section.
8.  **EIP-0663**: SWAPN, DUPN, EXCHANGE
    *   **FVM Implementation**: Add these stack manipulation opcodes (only enabled within EOF contracts).
9.  **EIP-7069**: Revamped CALL Instructions
    *   **FVM Implementation**: Add `EXTCALL`, `EXTDELEGATECALL`, `EXTSTATICCALL` which abstract away gas semantics (using the 63/64 rule by default).
10. **EIP-7620**: EOF Contract Creation
    *   **FVM Implementation**: Add `EOFCREATE` and `RETURNCODE`. `CREATE` and `CREATE2` will be disallowed for EOF code.
11. **EIP-7698**: Creation Transaction
    *   **FVM Implementation**: Ensure Filecoin's creation method can accept and process EOF initcode containers.

### FVM-Specific Adjustments
*   **Gas Model**: FVM uses Wasm metering. The *relative* gas costs of EOF instructions should match Ethereum's schedule to prevent arbitrage, but the absolute "gas" consumed is determined by the Wasm instruction count. Since `RJUMP` avoids `JUMPDEST` table lookups, it will naturally be cheaper in Wasm cycles, preserving the intent of the EIP.
*   **Legacy Support**: Existing EVM contracts (legacy format) must continue to function exactly as before. The EOF path is strictly opt-in via the container format.

## Design Rationale
*   **Monolithic Bundle**: Adopting EIP-7692 as a single unit is necessary because the EIPs are highly interdependent (e.g., `JUMPF` relies on Functions, which rely on the Format). Partial adoption would fragment the developer experience.
*   **EVM Actor Integration**: All changes are contained within the `builtin-actors` EVM actor. This isolates risk from the core FVM logic (`ref-fvm`).

## Backwards Compatibility
This FIP is a breaking change for the **EVM Actor** but is backward compatible for **existing contracts**.
*   **Legacy Contracts**: Contracts already deployed (or new contracts deployed with legacy bytecode) are unaffected.
*   **New Contracts**: New contracts can opt-in to EOFv1.
*   **Tooling**: Client tooling (Lotus, etc.) may need updates to support inspecting EOF containers (e.g., separate code/data fields).

## Test Cases
*   **Ethereum Tests**: Port the official Ethereum Execution Spec Tests (EEST) for EOF to the FVM test harness.
*   **FVM Integration Tests**: Create new test vectors in `builtin-actors/integration_tests` specifically for:
    *   Deploying an EOF contract via `Init` actor.
    *   Validating correct rejection of malformed EOF containers.
    *   Testing `RJUMP` vs `JUMP` gas behavior.

## Security Considerations
*   **Validation Complexity**: The validation pass (EIP-3670/5450) is complex and runs on-chain during deployment. It must be strictly bounded in complexity (linear time) to prevent DoS attacks.
*   **Stack Depth**: EOF introduces a return stack. FVM must enforce limits on this stack to prevent recursion-based stack overflow attacks in the Wasm host.

## Incentive Considerations
N/A

## Product Considerations
Enables the deployment of next-generation Solidity contracts (compiled with EOF support) on Filecoin, ensuring parity with EVM tooling and Layer 2s.

## Implementation
Tracking in `builtin-actors` repository.

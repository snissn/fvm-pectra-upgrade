# Pectra Phase 2 — EVM Object Format (EOFv1)

## Goal
Begin work on [EIP-7692](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7692.md), which bundles a major upgrade to EVM bytecode structure and validation. EOFv1 introduces structured containers, static analysis, functions, and safer execution primitives — critical for advanced contract safety and VM extensibility. Maintain close alignment between builder feedback and protocol roadmap by coordinating across FVM engineering, ecosystem partners, and external contributors.

Given the size and complexity of the EOFv1 bundle—which spans over 10 interdependent EIPs—work in this grant cycle will focus on scoping, analysis, FIP translation, and early prototyping. Implementation priority will be based on technical feasibility, community readiness, and ecosystem demand. This allows us to make meaningful progress toward long-term adoption without overcommitting to full support for all components in a single cycle.

## Available Resources (Read-Only)
The following local repositories in `./multistage-execution/` are available for reference, context analysis, and feasibility checking:
- **`ref-fvm`**: Reference implementation of the Filecoin Virtual Machine (Rust). Key for assessing architectural compatibility and gas models.
- **`builtin-actors`**: System actors implementation. Critical for understanding how EOF might interact with actor code and upgrades.
- **`lotus`**: The Filecoin node reference implementation. Used for integration feasibility and node-level impact analysis.
- **`FIPs`**: Filecoin Improvement Proposals repository. Reference for drafting new specifications and understanding existing standards.

## EOF-related EIPs in scope:

- [EIP-3540: EOF Format v1](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3540.md)
- [EIP-3670: Code Validation](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3670.md)
- [EIP-4200: Static Relative Jumps](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4200.md)
- [EIP-4750: Functions](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4750.md)
- [EIP-5450: Stack Validation](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-5450.md)
- [EIP-6206: JUMPF and Non-Returning Functions](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-6206.md)
- [EIP-7480: Data Section Access Instructions](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7480.md)
- [EIP-0663: SWAPN, DUPN and EXCHANGE Instructions](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-0663.md)
- [EIP-7069: Revamped CALL Instructions](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7069.md)
- [EIP-7620: EOF Contract Creation](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7620.md)
- [EIP-7698: Creation Transaction](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7698.md)

## Execution Plan

### Phase 1: Scoping & Technical Analysis
- [x] **EIP Deep Dive**: Conduct a detailed technical review of all 11 scoped EIPs to understand their interactions and edge cases.
- [x] **FVM Compatibility Assessment**: Analyze `./multistage-execution/ref-fvm` to map EOFv1 structures (e.g., container formats, static jumps) to the existing architecture.
- [x] **Actor Impact Analysis**: Review `./multistage-execution/builtin-actors` to evaluate potential backward compatibility issues or required upgrades.
- [x] **Dependency Mapping**: Create a dependency graph for the EIPs to determine the optimal implementation order.
- [x] **Gas Model Analysis**: Evaluate the gas implications of new instructions and validation passes against the `./multistage-execution/ref-fvm` execution environment.

**Phase 1 Findings Summary:**
- **Architecture**: `ref-fvm` is a generic Wasm host. EOF support must be implemented entirely within the **EVM Actor** in `builtin-actors` (specifically `actors/evm`).
- **Gas Model**: The EVM actor does not use a virtual gasometer for basic instructions; it relies on FVM's Wasm gas metering. EOF's static jumps (`RJUMP`) will naturally be cheaper than dynamic `JUMP`s because they avoid the runtime `JUMPDEST` bitmap check, aligning with EOF's efficiency goals without requiring explicit gas schedule tuning.
- **Dependencies**: Implementation should proceed in the order: Core Structure (Format/Validation) -> Control Flow (Jumps/Functions) -> Extended Validation -> Features.
- **Compatibility**: `initialize_evm_contract` and `load_bytecode` in the EVM actor are the key integration points for EOF container parsing and validation.


### Phase 2: FIP Translation & Specification
- [x] **Draft FIP-EIP Mappings**: Created a master FIP draft (`FIP_Drafts/FIP_EOF_Bundle.md`) that bundles the 11 EOF EIPs for FVM adoption.
- [x] **Identify FVM Divergences**: Documented key implementation differences in `FIP_Drafts/FVM_EOF_Divergences.md` (Gas, Creation, Stacks).
- [x] **Community RFCs**: Drafted `FIP_Drafts/RFC_EOF_Adoption.md` to solicit ecosystem feedback on tooling and gas impacts.

**Phase 2 Deliverables:**
- **FIP-XXXX**: A comprehensive specification for upgrading the FVM EVM actor to support EOFv1.
- **Divergence Report**: A technical guide for implementers highlighting where FVM semantics differ from the EVM spec.
- **RFC**: A communication artifact to align with the community.

### Phase 3: Early Prototyping & Architecture
- [x] **Container Format Prototype (EIP-3540)**: Developed a Rust prototype for parsing EOFv1 containers, including header and section extraction.
- [x] **Validation Logic Stub (EIP-3670)**: Implemented a basic validator function enforcing EIP-3670 rules (forbidden opcodes, PUSH data integrity, section ordering).
- [x] **Instruction Set Expansion**: Prototyped `RJUMP` and `RJUMPI` opcode behavior within a simulated EVM step function, demonstrating relative jump logic.
- [x] **Integration Feasibility Test**: Assessed `lotus` integration complexity, identifying minimal impact on core block processing, but moderate impact on transaction handling (new transaction types for EOF creation) and tooling updates.

**Phase 3 Findings Summary:**
- **Container Format**: A Rust prototype (`pectra/prototype/eof/src/lib.rs`) can successfully parse EOF headers and extract sections.
- **Validation**: Early validation can effectively catch malformed EOF contracts (forbidden opcodes, PUSH data issues, basic section ordering/counts).
- **Instruction Prototyping**: The `RJUMP` and `RJUMPI` opcodes demonstrate a straightforward implementation, with relative jumps efficiently handled at the opcode level.
- **Lotus Impact**: Core `lotus` changes appear minimal, as it primarily interfaces with the FVM. The main efforts will be in transaction type handling and ensuring `lotus` tooling correctly interprets EOF bytecode.


### Phase 4: Reporting & Coordination
- [x] **Builder Feedback Loop**: Drafted a strategy document (`pectra/reports/Builder_Feedback_Strategy.md`) outlining key discussion areas and an approach for engaging FVM builders.
- [x] **Progress Report**: Compiled a mid-cycle report (`pectra/reports/Progress_Report.md`) summarizing findings from Phase 1, 2, and 3, referencing all generated artifacts.
- [x] **Final Grant Report**: Prepared a polished, comprehensive summary of the project (`pectra/reports/Final_Grant_Report.md`), suitable for submission.

**Phase 4 Deliverables:**
- **Builder Feedback Strategy**: A plan for engaging the FVM ecosystem.
- **Progress Report**: Documentation of findings and status up to prototyping.
- **Final Grant Report**: A comprehensive summary of the entire grant cycle's work.


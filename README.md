# FVM EOFv1 Integration (Project Pectra)

> **Status**: Research & Prototyping Phase
> **Focus**: Ethereum Pectra Upgrade Alignment

This repository contains the comprehensive research, technical analysis, specification drafts, and early prototypes for integrating the **EVM Object Format (EOFv1)** bundle (EIP-7692) into the Filecoin Virtual Machine (FVM).

## 📖 Project Goal: Fulfilling the Grant Mandate

This project was initiated with the specific mandate to:
> *"Begin work on [EIP-7692]... work in this grant cycle will focus on **scoping, analysis, FIP translation, and early prototyping**. Implementation priority will be based on technical feasibility..."*

This repository serves as the definitive artifact resolving this goal. We have successfully:
1.  **Scoped** the 11 interdependent EIPs required for EOFv1.
2.  **Analyzed** the FVM architecture (`ref-fvm` and `builtin-actors`) to determine compatibility and implementation strategies.
3.  **Translated** Ethereum specs into actionable FIP drafts and technical reports.
4.  **Prototyped** the core container format and validation logic in Rust to prove technical feasibility.

**[Read the Final Grant Report](pectra/reports/Final_Grant_Report.md)** for a detailed breakdown of how every grant requirement was met.

### What is EOF?
Legacy EVM bytecode is unstructured, mixing code and data, which complicates analysis and optimization. **EOF (EIP-3540)** introduces a versioned container format (`0xEF00...`) that separates code and data sections. This enables:
*   **Static Validation**: Code is validated *once* at deployment time (EIP-3670), rejecting invalid contracts before they run.
*   **Efficient Execution**: New static jump instructions (`RJUMP`) replace dynamic jumps, removing the need for runtime `JUMPDEST` analysis.
*   **Structured Control Flow**: Introduction of Functions (`CALLF`, `RETF`) and a dedicated return stack.

---

## 📂 Repository Structure

This repository is organized into research, specification, and engineering artifacts:

### 1. Research & Analysis
*   **[`EIP_Analysis.md`](EIP_Analysis.md)**: A deep-dive technical summary of all 11 scoped EIPs, explaining their mechanics and impact.
*   **[`EOF_Dependencies.md`](EOF_Dependencies.md)**: A dependency graph visualizing the optimal implementation order (Format -> Control Flow -> Validation -> Features).
*   **[`GEMINI.md`](GEMINI.md)**: The project execution log and task tracker.

### 2. Specifications (FIP Drafts)
Located in **[`FIP_Drafts/`](FIP_Drafts/)**:
*   **`FIP_EOF_Bundle.md`**: The master **Filecoin Improvement Proposal** draft proposing the adoption of the entire EOFv1 bundle.
*   **`FVM_EOF_Divergences.md`**: A critical technical report detailing where FVM implementation *must* differ from Ethereum (e.g., Wasm-based gas metering, Account Abstraction interactions, Contract Creation flow).
*   **`RFC_EOF_Adoption.md`**: A "Request for Comments" document designed to solicit feedback from the Filecoin builder community regarding tooling and workflow impacts.

### 3. Engineering Prototype
Located in **[`prototype/eof/`](prototype/eof/)**:
A Rust-based proof-of-concept that implements the core EOF logic:
*   **Container Parsing**: Fully parses the EOFv1 header, Type, Code, and Data sections.
*   **Validation**: Implements **EIP-3670** validation rules, checking for forbidden opcodes (`INVALID`, `SELFDESTRUCT`, `JUMP`, `JUMPI`), PUSH data integrity, and section ordering.
*   **Simulation**: Contains a basic step-function simulator for the new **EIP-4200** instructions (`RJUMP`, `RJUMPI`), demonstrating how relative offsets work.

### 4. Project Reports
Located in **[`pectra/reports/`](pectra/reports/)**:
*   **[`Final_Grant_Report.md`](pectra/reports/Final_Grant_Report.md)**: **<-- START HERE.** The comprehensive summary of the entire grant cycle's deliverables and findings.
*   **`Progress_Report.md`**: Mid-cycle status update.
*   **`Builder_Feedback_Strategy.md`**: A strategic plan for engaging the ecosystem.

---

## 🛠 In-Scope EIPs

This project covers the following Ethereum Improvement Proposals:

| EIP | Description | Impact |
| :--- | :--- | :--- |
| **EIP-3540** | EOF - EVM Object Format v1 | Core container structure. |
| **EIP-3670** | EOF - Code Validation | Deploy-time bytecode verification. |
| **EIP-4200** | EOF - Static Relative Jumps | `RJUMP`, `RJUMPI`, `RJUMPV`. |
| **EIP-4750** | EOF - Functions | `CALLF`, `RETF`, Return Stack. |
| **EIP-5450** | EOF - Stack Validation | Static stack height analysis. |
| **EIP-6206** | EOF - JUMPF | Tail call optimization. |
| **EIP-7480** | EOF - Data Section Access | `DATALOAD`, `DATACOPY`. |
| **EIP-0663** | SWAPN, DUPN, EXCHANGE | New stack manipulation opcodes. |
| **EIP-7069** | Revamped CALL Instructions | `EXTCALL`, `RETURNDATALOAD`. |
| **EIP-7620** | EOF Contract Creation | `EOFCREATE`, `RETURNCODE`. |
| **EIP-7698** | EOF - Creation Transaction | Deploying EOF via transaction. |

---

## 🚀 Getting Started

### Prerequisites
*   **Rust**: Ensure you have the latest stable Rust toolchain installed.

### Running the Prototype
To see the EOF parser and validator in action:

1.  Navigate to the prototype directory:
    ```bash
    cd prototype/eof
    ```
2.  Run the test suite:
    ```bash
    cargo test
    ```

This will execute unit tests that:
*   Parse valid EOF byte streams.
*   Reject invalid headers (magic numbers, versions).
*   Validate code sections against EIP-3670 rules.
*   Simulate execution of static jumps.

---

## 🔍 Key Findings & Divergences

*   **Architecture**: `ref-fvm` requires minimal changes. The bulk of the work lies within the **EVM Actor** (`builtin-actors`).
*   **Gas Model**: The FVM's Wasm metering is naturally aligned with EOF. Static jumps are cheaper to execute in Wasm than dynamic jumps, preserving the economic incentives of EOF without complex gas schedule hacks.
*   **Contract Creation**: The FVM's actor creation flow (via `Init` actor) will need to accommodate the specific semantics of `EOFCREATE` and new transaction types.

## 📅 Roadmap

*   **Phase 1**: Scoping & Technical Analysis (✅ Completed)
*   **Phase 2**: FIP Translation & Specification (✅ Completed)
*   **Phase 3**: Early Prototyping & Architecture (✅ Completed)
*   **Phase 4**: Reporting & Coordination (✅ Completed)
*   **Next Steps**: Production implementation in `builtin-actors` repository.

## 📄 License

[MIT](LICENSE)

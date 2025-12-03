# Pectra Phase 2 — EVM Object Format (EOFv1) Integration on FVM

This repository contains the research, analysis, specifications, and prototypes for integrating the **EVM Object Format (EOFv1)** bundle (EIP-7692) into the Filecoin Virtual Machine (FVM).

## Project Overview

The goal of this project is to align the FVM with the upcoming Ethereum Pectra upgrade, specifically focusing on the EOFv1 bundle. This involves scoping 11 interdependent EIPs, translating them into Filecoin Improvement Proposals (FIPs), and prototyping the necessary changes within the FVM architecture.

**Key Objectives:**
*   Analyze the technical impact of EOFv1 on FVM.
*   Draft FIPs for adopting the EOF bundle.
*   Prototype the container format, validation logic, and new instructions.
*   Engage the community for feedback.

## Repository Structure

*   **`GEMINI.md`**: The main project tracking file, outlining goals, scope, and task status across all phases.
*   **`EIP_Analysis.md`**: A detailed technical deep-dive into each of the 11 EIPs in the EOFv1 bundle.
*   **`EOF_Dependencies.md`**: A dependency graph illustrating the optimal implementation order for the EIPs.
*   **`FIP_Drafts/`**:
    *   `FIP_EOF_Bundle.md`: The master FIP draft proposing the adoption of the EOFv1 bundle.
    *   `FVM_EOF_Divergences.md`: A report detailing technical divergences between EVM and FVM implementations (e.g., gas, creation).
    *   `RFC_EOF_Adoption.md`: A Request for Comments document for community engagement.
*   **`prototype/eof/`**: A Rust-based prototype demonstrating:
    *   **Container Parsing**: Parsing of the `0xEF00` EOF header and sections.
    *   **Validation**: Implementation of EIP-3670 validation rules (forbidden opcodes, etc.).
    *   **Simulation**: Basic execution simulation for new `RJUMP` and `RJUMPI` instructions.
*   **`pectra/reports/`**:
    *   `Builder_Feedback_Strategy.md`: Strategy for gathering ecosystem feedback.
    *   `Progress_Report.md`: Mid-cycle progress summary.
    *   `Final_Grant_Report.md`: Comprehensive final report of the grant cycle work.

## Getting Started with the Prototype

To explore the EOF prototype:

1.  Navigate to the prototype directory:
    ```bash
    cd prototype/eof
    ```
2.  Run the tests to see parsing and validation in action:
    ```bash
    cargo test
    ```

## License

[MIT](LICENSE) (or standard Filecoin project license)

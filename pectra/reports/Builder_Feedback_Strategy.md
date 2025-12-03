# Builder Feedback Strategy for EOFv1 Adoption on FVM

## Objective
To actively engage key FVM ecosystem builders, developers, and tool maintainers to gather critical feedback on the proposed EVM Object Format (EOFv1) bundle integration. This feedback will inform the FIP finalization, refine implementation priorities, and ensure a smooth transition.

## Key Discussion Areas & Questions

### 1. Tooling & Infrastructure Impact (EIP-3540, EIP-7480)
*   **Context**: EOF introduces a structured container (`0xEF00` magic) with separate code and data sections. `EXTCODESIZE`, `EXTCODECOPY` behavior changes for EOF. New `DATALOAD`/`DATACOPY` opcodes provide explicit data access.
*   **Questions for Builders**:
    *   How will current FVM tools (explorers, indexers, debuggers, wallets, RPC endpoints) handle and interpret EOF-formatted bytecode?
    *   What are the anticipated challenges in adapting tooling to parse and display EOF contracts?
    *   Are there existing features in your tools that rely on the legacy code/data model that will need significant refactoring?
    *   How do you currently inspect contract bytecode and data? Will the new `DATALOAD` paradigm simplify or complicate this for you?

### 2. Contract Development & Deployment Workflow (EIP-7620, EIP-7698)
*   **Context**: Legacy `CREATE`/`CREATE2` are disallowed for EOF. New `EOFCREATE`/`RETURNCODE` are used for deployment. EOF contracts are deployed via specialized creation transactions.
*   **Questions for Builders**:
    *   How will your contract development workflows (compilation, deployment scripts, testing frameworks) adapt to the new EOF creation mechanisms?
    *   Do you rely on specific `CREATE`/`CREATE2` behaviors (e.g., address predictability) that need careful consideration in the EOF context?
    *   What are the implications for existing contract factory patterns?
    *   What level of support would you need from the FVM team (e.g., libraries, SDK updates) to facilitate EOF contract deployment?

### 3. Performance & Gas Model (EIP-4200, EIP-5450, EIP-7069)
*   **Context**: EOF promises gas efficiency through static jumps (`RJUMP`, `RJUMPI`), static stack validation, and revamped calls (`EXTCALL`). FVM's gas model is based on Wasm execution.
*   **Questions for Builders**:
    *   Are there specific high-gas-cost contract patterns that you hope EOF will optimize?
    *   How critical is exact EVM gas parity vs. relative efficiency gains for your applications?
    *   What are your expectations for performance improvements (or potential regressions) with EOF contracts on FVM?

### 4. General Compatibility & Future-Proofing
*   **Context**: EOF is a major step towards a more robust and extensible EVM. Maintaining compatibility with Ethereum's roadmap is paramount.
*   **Questions for Builders**:
    *   Are there any critical EOF features or related EIPs (beyond the initial bundle) that are essential for your roadmap?
    *   What are your biggest concerns or questions regarding the FVM's adoption of EOFv1?

## Engagement Channels
*   **Targeted Workshops/Demos**: Schedule dedicated sessions with key FVM ecosystem partners (e.g., wallet providers, RPC services, major dApp teams).
*   **Public Forums**: Leverage the `FIPs` GitHub discussions, Filecoin Discord channels (#fvm-dev), and community calls.
*   **Surveys/Questionnaires**: Distribute structured questionnaires to gather quantitative feedback.

## Next Steps
1.  Finalize the FIP draft (`FIP_Drafts/FIP_EOF_Bundle.md`).
2.  Publish the RFC (`FIP_Drafts/RFC_EOF_Adoption.md`) to a public forum (e.g., FIPs GitHub discussions).
3.  Initiate direct outreach to FVM ecosystem partners based on this strategy.

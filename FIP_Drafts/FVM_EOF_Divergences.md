# FVM vs. EVM Divergences for EOFv1

## 1. Gas Metering & Execution
*   **EVM**: Uses a distinct "gas schedule" where every opcode has a specific cost (e.g., `JUMP` = 8, `RJUMP` = 2).
*   **FVM**: Uses Wasm instruction metering ("Fuel"). The cost of an EVM opcode is the aggregate cost of the Wasm instructions required to interpret it.
*   **Divergence**: While we aim for *relative* parity, we cannot guarantee exact gas cost equivalence.
    *   *Mitigation*: `RJUMP` in FVM will naturally be cheaper than `JUMP` because it skips the `valid_jump_destination` check (which involves vector lookups). This preserves the *incentive* of EOF even if the exact numbers differ.
    *   *Action*: We must ensure `RJUMP` implementation is indeed more efficient in Rust/Wasm than `JUMP`.

## 2. Contract Creation & Address Derivation
*   **EVM**: `EOFCREATE` uses specific salt and initcode hash rules.
*   **FVM**: Contract creation usually goes through the `Init` actor or `EAM` (Ethereum Address Manager).
*   **Divergence**: The `EOFCREATE` opcode in the EVM actor must correctly interface with FVM's `send` syscall to the EAM or Init actor. The address derivation logic in EAM might need to be aware of the new EOF initcontainer hash rules to ensure addresses match Ethereum's expectations (especially for `CREATE2` equivalents).

## 3. Code & Data Separation
*   **EVM**: Code is strictly read-only and separate from Data.
*   **FVM**: `ref-fvm` loads code as a Wasm module (for native actors) or a byte array (for EVM).
*   **Divergence**: The EVM actor currently loads the entire bytecode into memory. With EOF, we might want to optimize this.
    *   *Future Optimization*: Use FVM's blockstore to load only the *Code* section into memory for execution, and load *Data* section pages lazily via `DATALOAD`, avoiding large memory allocations for contracts with huge data sections.

## 4. Stack Limits
*   **EVM**: Enforces a 1024 item data stack and a 1024 item return stack.
*   **FVM**: The Wasm host has its own stack limits (`max_wasm_stack`).
*   **Divergence**: We must ensure that the explicit `Vec` used for the EOF Return Stack in the EVM actor does not allow a user to exceed the Wasm memory limits or cause a panic. The validation pass (EIP-5450) provides strong guarantees here, but the *implementation* of the stack in Rust must be robust.

## 5. Syscalls
*   **EVM**: `EXTCALL` etc. have specific return status codes (0, 1, 2).
*   **FVM**: Syscalls return `ExitCode`.
*   **Divergence**: The EVM actor's interpreter loop must map FVM `ExitCode`s (e.g., `SysErrorIllegalArgument`) to the correct EOF status codes (e.g., `1` for Revert or `2` for Failure) when handling `EXTCALL` results.

# EIP Analysis Report for Pectra Phase 2 (EOFv1)

## EIP-3540: EOF Format v1

### Summary
EIP-3540 introduces the EVM Object Format (EOF) v1, a new, extensible, and versioned container format for EVM bytecode. Its primary goals are to:
- Enable once-off validation of contract code at deployment time, reducing execution overhead.
- Facilitate future EVM changes and opcode deprecation/introduction.
- Separate code and data within contracts, which benefits static analysis and layer-2 solutions.

### Key Aspects
- **Magic Number**: EOF contracts are identified by a `0xEF00` magic byte sequence at their beginning. Legacy contracts remain unaffected.
- **Structure**: An EOF container consists of a header (magic, version, section kinds/sizes, terminated by 0x00) and a body (containing code, type metadata, nested containers, and arbitrary data sections).
- **Validation Rules**: Specifies rules for valid EOF containers, including version constraints, section structure, and overall size limits (`MAX_INITCODE_SIZE` from EIP-3860).
- **Execution Semantics Changes**: Rejects certain opcodes (`CODESIZE`, `CODECOPY`, `EXTCODESIZE`, etc.) for EOF contracts. `CALL`, `DELEGATECALL`, `STATICCALL` are also rejected, with replacements deferred to other EIPs. `DELEGATECALL` from an EOF contract to a non-EOF contract is disallowed.
- **Backward Compatibility**: Designed to be a breaking change but avoids affecting existing contracts due to the magic number.

## EIP-3670: Code Validation

### Summary
EIP-3670 introduces mandatory code validation for EOF-formatted contracts at creation time. This validation ensures the integrity and well-formedness of the bytecode, preventing deployment of malformed or insecure contracts.

### Key Aspects
- **Creation-Time Validation**: All EOF contracts undergo validation upon deployment.
- **Invalid Instruction Rejection**: Contracts containing undefined instructions or deprecated opcodes (e.g., `CALLCODE`, `SELFDESTRUCT`) are rejected.
- **PUSH-Data Validation**: Ensures that `PUSH` instructions are correctly formed and do not have truncated data.
- **Consensus Integration**: By moving code validity checks into consensus, it simplifies future EVM upgrades and allows for easier introduction of new instructions.

## EIP-4200: Static Relative Jumps

### Summary
EIP-4200 introduces three new EVM jump instructions (`RJUMP`, `RJUMPI`, `RJUMPV`) for EOF-formatted contracts. These instructions facilitate static analysis, reduce gas costs, and enable relocatable code by using relative offsets for jump destinations.

### Key Aspects
- **New Instructions**:
    - `RJUMP` (0xe0): Unconditional relative jump.
    - `RJUMPI` (0xe1): Conditional relative jump.
    - `RJUMPV` (0xe2): Relative jump with a jump table and a fallback.
- **Relative Addressing**: Jump destinations are specified as signed 16-bit immediate values relative to the current program counter, making code relocatable.
- **Validation**: These instructions are only valid for EOF1 formatted code. Validation ensures jump targets are within code bounds and point to instructions.
- **Gas Efficiency**: The static nature of these jumps allows for lower gas costs (e.g., `RJUMP` at 2 gas) due to upfront validation.
- **No `JUMPDEST`**: Eliminates the need for `JUMPDEST` markers, further saving gas.
- **Backward Compatibility**: Designed to be compatible with EIP-3540 and not affect legacy bytecode.

## EIP-4750: Functions

### Summary
EIP-4750 introduces structured function calls for EOF-formatted contracts by allowing multiple code sections, each acting as a function/subroutine. This enables a more organized control flow and improves code analysis. It introduces new opcodes `CALLF` and `RETF` and a separate return stack to manage execution state.

### Key Aspects
- **Multiple Code Sections**: EOF contracts can contain several code sections, each representing a function.
- **`CALLF` (0xe3)**: New opcode for calling a function. Takes a 16-bit `target_section_index` as an immediate argument. Pushes current execution state (section index, program counter) to a dedicated return stack.
- **`RETF` (0xe4)**: New opcode for returning from a function. Pops execution state from the return stack to resume previous execution.
- **Return Stack**: A separate stack (limited to 1024 items) is introduced to manage function calls and returns, isolating function stack frames.
- **Removal of Dynamic Jumps**: `JUMP` and `JUMPI` opcodes are disallowed in EOF. `JUMPDEST` becomes a `NOP` and `PC` becomes invalid, moving towards more structured control flow.
- **Improved Analysis**: Isolating function stacks and disallowing dynamic jumps enhance static analysis capabilities.
- **Backward Compatibility**: Only applies to EOF1 contracts, ensuring no impact on legacy bytecode.

## EIP-5450: Stack Validation

### Summary
EIP-5450 introduces static analysis to validate the operand stack behavior of EOF-formatted contracts at deployment time. This ensures that contracts cannot underflow or overflow the stack during execution, eliminating the need for many expensive runtime checks.

### Key Aspects
- **Static Analysis**: A linear pass over the code at deployment time calculates minimum and maximum stack height bounds for each instruction.
- **Underflow/Overflow Prevention**: Validates that no instruction can cause a stack underflow or overflow during execution.
- **Reduced Runtime Checks**: By ensuring stack safety statically, many runtime stack checks can be removed, improving execution efficiency.
- **Unreachable Code Detection**: Helps in identifying and preventing the deployment of contracts containing unreachable instructions.
- **Two-Phase Validation**: Integrates with existing instruction validation (EIP-3670, EIP-4200, EIP-4750) and adds operand stack validation.
- **Backward Compatibility**: Applies only to EOF1 contracts.

## EIP-6206: JUMPF and Non-Returning Functions

### Summary
EIP-6206 introduces the `JUMPF` instruction for tail-call optimizations within EOF functions and extends the type section format to declare functions as non-returning. This aims to improve gas efficiency and simplify stack validation for specific control flow patterns.

### Key Aspects
- **`JUMPF` (0xe5)**: New instruction for tail calls. It performs an unconditional relative jump to a target code section without modifying the return stack. Costs 5 gas.
- **Non-Returning Functions**: The type section format is extended to allow declaring a function as non-returning (e.g., using `0x80` for `outputs`). This signifies that the function will not return control to its caller (e.g., it always `REVERT`s or `STOP`s).
- **Validation**:
    - `JUMPF` target index must be valid.
    - Stack validation ensures output consistency or targets a non-returning function.
    - `CALLF` cannot target a non-returning section.
    - A section is non-returning if it contains no `RETF` or `JUMPF` to returning sections.
- **Optimization**: Allows compilers to generate more efficient code by avoiding unnecessary return stack operations for tail calls and by treating non-returning functions as terminal.
- **Backward Compatibility**: Applies only to EOF1 contracts.

## EIP-7480: Data Section Access Instructions

### Summary
EIP-7480 introduces new instructions to explicitly access the data section of an EOF container. This reinforces the separation of code and data, which is a core tenet of EOFv1, and provides a safer, more structured way to interact with contract data. These new instructions replace the functionality of older bytecode introspection opcodes for EOF contracts.

### Key Aspects
- **New Instructions**:
    - `DATALOAD` (0x4a): Loads a 32-byte word from the data section at a given offset.
    - `DATALOADN` (0x4b): Loads a 32-byte word from the data section at a specific index.
    - `DATASIZE` (0x4c): Pushes the size of the data section onto the stack.
    - `DATACOPY` (0x4d): Copies a block of bytes from the data section to memory.
- **Explicit Data Access**: Provides clear and explicit mechanisms for contracts to interact with their own data section.
- **Code/Data Separation**: Further enforces the distinction between executable code and immutable data within EOF containers.
- **Replacement for Deprecated Opcodes**: For EOF contracts, these instructions replace the functionality of `CODECOPY`, `CODESIZE`, and `EXTCODECOPY` when interacting with the contract's own data.
- **Validation**: Rules are defined for the valid use of these instructions, ensuring data access is within bounds and properly handled.
- **Backward Compatibility**: These new instructions only apply to EOF1 formatted contracts and do not affect legacy bytecode.

## EIP-0663: SWAPN, DUPN and EXCHANGE Instructions

### Summary
EIP-0663 proposes the introduction of new stack manipulation instructions (`SWAPN`, `DUPN`, and `EXCHANGE`) to address the limitations of existing `SWAP` and `DUP` opcodes, which can only access the top 16 items of the EVM stack. The goal is to enable easier access to deeper stack items, thereby simplifying compiler implementations for higher-level language constructs and improving stack scheduling algorithms.

### Key Aspects
- **Motivation**: Overcome the 16-item limit of current `SWAP` and `DUP` instructions to access deeper stack elements (the EVM stack has a depth of 1024 items).
- **New Instructions**:
    - `SWAPN`: Swaps an item at a specified depth with the top of the stack.
    - `DUPN`: Duplicates an item at a specified depth to the top of the stack.
    - `EXCHANGE`: Swaps two items at specified depths on the stack.
- **Benefits**: Simplifies compiler design for languages targeting the EVM, especially for managing function calls and local variables that might reside deeper in the stack. Improves optimization opportunities for stack-aware compilers.
- **Status**: (Based on web search) This EIP was proposed in 2017. While discussions and refinements have occurred, its official status (e.g., Final, Accepted) within the EIP process is not explicitly stated, and the original GitHub link for the EIP document is currently unavailable. It may have been superseded or integrated into other EIPs, particularly in the context of EOFv1.
- **Backward Compatibility**: Likely intended to be backward compatible for legacy code if implemented, but primarily relevant for new bytecode formats and compiler optimizations.

## EIP-7069: Revamped CALL Instructions

### Summary
EIP-7069 introduces new call instructions (`EXTCALL`, `EXTDELEGATECALL`, `EXTSTATICCALL`) and `RETURNDATALOAD` for EOF-formatted contracts. These instructions aim to simplify external contract calls, remove gas observability, and improve future EVM extensibility.

### Key Aspects
- **New Call Instructions**:
    - `EXTCALL`: External call.
    - `EXTDELEGATECALL`: External delegate call.
    - `EXTSTATICCALL`: External static call.
- **Simplified Call Semantics**: Removes the ability to specify a gas limit directly; gas is instead managed by the "63/64th rule" (EIP-150). Simplifies "stipend" rules.
- **`RETURNDATALOAD`**: A new instruction to load a 32-byte word from the return data buffer onto the stack, complementing `RETURNDATACOPY`.
- **Status Codes**: Call instructions return an extensible status code (`0` for success, `1` for revert, `2` for failure) instead of a boolean, providing more granular error handling.
- **Removed Gas Observability**: A key motivation is to eliminate gas observability, which has been a source of complexity and issues in EVM development.
- **Backward Compatibility**: These new instructions are only available within EOF code and are undefined in legacy bytecode.

## EIP-7620: EOF Contract Creation

### Summary
EIP-7620 introduces `EOFCREATE` and `RETURNCODE` instructions to manage contract creation within the EOF framework. This EIP is crucial for enabling factory contracts to deploy other EOF contracts, replacing the legacy `CREATE` and `CREATE2` opcodes which are incompatible with EOF's design principles, particularly regarding code observability.

### Key Aspects
- **New Instructions**:
    - `EOFCREATE` (0xec): Initiates the creation of a new EOF contract. It consumes gas, reads an `initcontainer_index`, and pops `salt`, `input_offset`, `input_size`, and `value` from the stack to execute the specified initcode container.
    - `RETURNCODE` (0xee): Used during the initcode execution to specify which container should be deployed as the new contract, and optionally to append auxiliary data.
- **Replacement for Legacy Creation**: `CREATE` and `CREATE2` are made obsolete for EOF contracts. Attempts to use them with EOF initcode will result in deployment failure without consuming gas or updating the caller's nonce.
- **Code Observability**: Aligns with EOF's goal of removing code observability by providing a structured and validated way to create contracts, where the deployed code is explicitly defined.
- **Data Section Lifecycle**: The EIP details how the data section is handled during contract creation, including the appending of auxiliary data.
- **Backward Compatibility**: Introduced alongside EIP-3540, ensuring it only affects EOF contracts and maintains compatibility with legacy contract creation.

## EIP-7698: Creation Transaction

### Summary
EIP-7698 enables the deployment of EOF contracts directly through creation transactions, building upon the EOF format (EIP-3540) and EOF contract creation mechanisms (EIP-7620). This EIP defines the transaction type and processing rules necessary for the initial on-chain presence of EOF contracts.

### Key Aspects
- **Transaction Type**: Introduces a new transaction type or extends existing ones to support EOF contract creation.
- **Initcode Container**: Specifies that the transaction payload contains an EOF initcode container, which is responsible for deploying the final EOF runtime code.
- **Parsing and Validation**: Details the process for parsing the EOF header of the initcode container, and the subsequent validation of the initcontainer.
- **Execution**: Describes how the initcontainer is executed to produce the final EOF contract code.
- **Motivation**: The primary motivation is to allow the initial deployment of EOF contracts on the Ethereum blockchain, making the entire EOF ecosystem functional from the transaction layer.
- **Backward Compatibility**: Designed to be fully compatible with existing legacy transactions and contract creation processes, only affecting transactions explicitly opting into the EOF creation mechanism.

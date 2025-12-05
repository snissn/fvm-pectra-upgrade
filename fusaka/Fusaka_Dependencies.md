# Fusaka Implementation Dependencies

This document maps the dependency graph for implementing the Fusaka upgrade on FVM.

**Note on Gas Changes:**
FVM gas accounting is based on Wasm execution metering handled by the FVM Runtime. Ethereum's gas schedule changes (EIP-7883 ModExp repricing, EIP-7935 60M limit, etc.) are **not applicable** and will not be implemented. We rely on the native FVM gas model.

```mermaid
graph TD
    Fusaka[Fusaka Upgrade Bundle] --> EVM_Actor
    Fusaka --> Client_RPC
    Fusaka --> FVM_Runtime

    subgraph EVM_Actor [Builtin-Actors: EVM]
        CLZ[EIP-7939: CLZ Opcode]
        Secp256r1_Precompile[EIP-7951: secp256r1 Precompile]
    end

    subgraph FVM_Runtime [FVM Runtime / Syscalls]
        Crypto_Syscall[New Syscall: verify_secp256r1?]
        style Crypto_Syscall stroke-dasharray: 5 5
        Note_Syscall[Optional: Can be pure Rust in Actor initially]
    end

    subgraph Client_RPC [Lotus / Forest]
        Eth_Config[EIP-7910: eth_config RPC]
    end

    %% Dependencies
    Secp256r1_Precompile -.->|Optional Optimization| Crypto_Syscall
    CLZ -->|Requires| Bytecode_Interpreter_Update
```

## Implementation Order

1.  **Phase A: Core Logic (Prototype)**
    *   Implement `CLZ` logic (Rust).
    *   Implement `secp256r1` verification logic (Rust, `p256` crate).
2.  **Phase B: Actor Integration (Spec)**
    *   Define Precompile Address for `secp256r1`.
    *   *Note: ModExp gas changes are skipped as FVM relies on Wasm metering.*
3.  **Phase C: Client Integration**
    *   Spec out `eth_config` response fields.
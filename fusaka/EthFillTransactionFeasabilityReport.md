# Feasibility Report: Implementing `eth_fillTransaction` in Lotus

**Date:** December 10, 2025  
**Project:** Lotus (Filecoin Implementation)  
**Subject:** Detailed Feasibility and Implementation Plan for `eth_fillTransaction`

---

## 1. Executive Summary

This report analyzes the feasibility of implementing the `eth_fillTransaction` JSON-RPC method within the Lotus codebase. The `eth_fillTransaction` method is a standard Ethereum JSON-RPC endpoint used to populate missing fields in a transaction object—specifically `nonce`, `gas` (limit), and gas pricing fields (`gasPrice` or `maxFeePerGas`/`maxPriorityFeePerGas`)—preparing it for signing and broadcasting.

**Conclusion:** Implementing `eth_fillTransaction` in Lotus is **highly feasible** and low-risk. The Lotus architecture already possesses distinct, robust APIs for every component required by this method (nonce management, gas estimation, and fee suggestion). The work primarily involves creating an orchestration layer to weave these existing components together.

---

## 2. Specification & Behavior Analysis

### 2.1 The `eth_fillTransaction` Spec

The `eth_fillTransaction` method accepts a partial transaction object and returns a fully populated, ready-to-sign transaction object.

*   **RPC Method:** `eth_fillTransaction`
*   **Parameters:** `[ transactionObject ]`
*   **Returns:** `transactionObject` (populated)

#### Expected Logic:
1.  **Validation:** Ensure the `from` address is present.
2.  **Nonce:** If the `nonce` field is missing (null), query the transaction pool or state for the next available nonce for the sender.
3.  **Transaction Type:** Determine if the transaction should be Legacy (Type 0) or EIP-1559 (Type 2). This often depends on whether `maxFeePerGas` is provided or if the chain supports EIP-1559. Lotus supports EIP-1559 natively.
4.  **Gas Pricing:**
    *   If EIP-1559 fields (`maxFeePerGas`, `maxPriorityFeePerGas`) are missing, fetch current network base fee and tip estimates.
    *   If Legacy `gasPrice` is missing, estimate it.
5.  **Gas Limit:** If the `gas` field is missing, perform a `eth_estimateGas` simulation using the (now partially filled) transaction details.
6.  **Return:** Construct the final object.

### 2.2 Survey of Reference Implementations (Geth)

In Go-Ethereum (Geth), `eth_fillTransaction` serves as a utility helper. It does not sign the transaction; it merely "fills in the blanks."

*   **Geth's Approach:** It utilizes the internal `eth.API` backend.
*   **Nonce:** Uses `GetPoolNonce` to find the pending nonce, preventing nonce collisions if multiple txs are queued.
*   **Gas:** It performs a binary search or execution trace (via `DoEstimateGas`) to find the optimal limit.
*   **Defaults:** It applies default values if the user provides none (e.g., ensuring `value` defaults to 0 if nil).

---

## 3. Lotus Feasibility Analysis

Lotus already implements the Filecoin EVM (FEVM), which maps Ethereum concepts to Filecoin's underlying actors and message passing.

### 3.1 Existing Capabilities Mapping

| Requirement | Lotus Equivalent | Feasibility Status |
| :--- | :--- | :--- |
| **Nonce Retrieval** | `MpoolAPI.MpoolGetNonce(ctx, address)` | **Ready**. Returns the next nonce considering pending messages. |
| **Gas Price (1559)** | `EthGasAPI.EthMaxPriorityFeePerGas` + BaseFee from TipSet | **Ready**. Lotus fully supports EIP-1559 fee markets. |
| **Gas Estimation** | `EthGasAPI.EthEstimateGas` | **Ready**. Existing logic simulates execution to determine gas usage. |
| **Data Structures** | `chain/types/ethtypes` | **Ready**. `EthTx` and `EthCall` exist, though a new input struct is needed (see Section 4). |

### 3.2 Architectural Fit

The implementation naturally belongs in the `EthGasAPI` or `EthTransactionAPI` interface implementations (`node/impl/eth/gas.go` or similar). Since `EthEstimateGas` resides in `node/impl/eth/gas.go`, placing `EthFillTransaction` there minimizes code duplication and circular dependencies, as filling a transaction often requires estimating gas.

---

## 4. Implementation Plan

The implementation requires changes in three specific areas:
1.  **Type Definitions**: To handle "optional" fields correctly.
2.  **API Interface**: Exposing the method.
3.  **Core Logic**: The orchestration function.

### 4.1 Step 1: Define `EthFillTransactionArgs`

Standard `EthTx` structs in Lotus often use value types (e.g., `EthUint64` for `Gas`). In JSON-RPC, a missing field unmarshals to `0`. However, a user might *explicitly* want `Gas: 0` (unlikely but possible) vs "Please estimate gas". More importantly, `Nonce: 0` is valid.

We need a struct where these fields are pointers to distinguish `nil` (missing) from `0`.

**File:** `chain/types/ethtypes/eth_types.go`

```go
// Add this new struct
type EthFillTransactionArgs struct {
    From                 EthAddress  `json:"from"` // Required
    To                   *EthAddress `json:"to"`
    Gas                  *EthUint64  `json:"gas"`  // Pointer to allow nil check
    GasPrice             *EthBigInt  `json:"gasPrice"`
    MaxFeePerGas         *EthBigInt  `json:"maxFeePerGas"`
    MaxPriorityFeePerGas *EthBigInt  `json:"maxPriorityFeePerGas"`
    Value                *EthBigInt  `json:"value"`
    Data                 EthBytes    `json:"data"`
    Nonce                *EthUint64  `json:"nonce"` // Pointer to allow nil check (0 is valid)
    ChainID              *EthUint64  `json:"chainId"`
}
```

### 4.2 Step 2: Update API Interfaces

**File:** `node/impl/eth/api.go`

We need to add the method signature to the `EthGasAPI` interface (or `EthTransactionAPI`).

```go
type EthGasAPI interface {
    // ... existing methods ...
    EthEstimateGas(ctx context.Context, p jsonrpc.RawParams) (ethtypes.EthUint64, error)
    
    // NEW METHOD
    EthFillTransaction(ctx context.Context, p jsonrpc.RawParams) (*ethtypes.EthTx, error)
}
```

### 4.3 Step 3: Core Implementation

**File:** `node/impl/eth/gas.go`

This is where the logic resides. We will create the function that orchestrates the filling.

```go
func (e *ethGas) EthFillTransaction(ctx context.Context, p jsonrpc.RawParams) (*ethtypes.EthTx, error) {
    // 1. Parse Parameters
    params, err := jsonrpc.DecodeParams[ethtypes.EthFillTransactionArgs](p)
    if err != nil {
        return nil, xerrors.Errorf("decoding params: %w", err)
    }

    // 2. Setup the Result Object (Start with what we have)
    res := &ethtypes.EthTx{
        From: params.From,
        To:   params.To,
        Input: params.Data,
    }
    
    // Handle Value
    if params.Value != nil {
        res.Value = *params.Value
    } else {
        res.Value = ethtypes.EthBigInt(big.Zero())
    }

    // 3. Fill Nonce
    if params.Nonce != nil {
        res.Nonce = *params.Nonce
    } else {
        // Convert EthAddress to Filecoin Address
        faddr, err := params.From.ToFilecoinAddress()
        if err != nil {
             return nil, err
        }
        // Fetch Nonce from Mpool
        nonce, err := e.messagePool.MpoolGetNonce(ctx, faddr)
        if err != nil {
            return nil, xerrors.Errorf("failed to get nonce: %w", err)
        }
        res.Nonce = ethtypes.EthUint64(nonce)
    }

    // 4. Fill Gas Pricing (EIP-1559 Default)
    // Note: A robust implementation checks if the user requested Legacy (provided GasPrice)
    // vs EIP-1559. Assuming EIP-1559 for Filecoin:
    if params.MaxFeePerGas != nil {
        res.MaxFeePerGas = params.MaxFeePerGas
    }
    if params.MaxPriorityFeePerGas != nil {
        res.MaxPriorityFeePerGas = params.MaxPriorityFeePerGas
    }
    
    // If pricing is missing, calculate it
    if res.MaxFeePerGas == nil || res.MaxPriorityFeePerGas == nil {
        // Get Tip
        tip, err := e.EthMaxPriorityFeePerGas(ctx)
        if err != nil {
            return nil, err
        }
        
        // Get BaseFee
        ts := e.chainStore.GetHeaviestTipSet()
        baseFee := ts.Blocks()[0].ParentBaseFee
        
        if res.MaxPriorityFeePerGas == nil {
            res.MaxPriorityFeePerGas = &tip
        }
        if res.MaxFeePerGas == nil {
            // formula: baseFee + tip
            mf := big.Add(baseFee, big.Int(tip))
            mfEth := ethtypes.EthBigInt(mf)
            res.MaxFeePerGas = &mfEth
        }
    }
    res.Type = ethtypes.EIP1559TxType // Explicitly mark as Type 2

    // 5. Fill Gas Limit
    if params.Gas != nil {
        res.Gas = *params.Gas
    } else {
        // Construct an EstimateGas call
        // We need to map our partially filled tx to the EthCall struct used by EstimateGas
        estimateArgs := ethtypes.EthCall{
            From:  &res.From,
            To:    res.To,
            Value: res.Value,
            Data:  res.Input,
            // We use the pricing we just calculated to ensure estimation is accurate
            // (though pricing rarely affects gas used, only affordability)
        }
        
        // Marshal params to match EthEstimateGas signature
        // Or call internal estimation logic directly if available to avoid JSON overhead
        // For simplicity here, we assume calling the internal logic:
        
        // NOTE: e.EthEstimateGas takes RawParams, so we might need to mock that 
        // or extract the internal logic of EthEstimateGas into a reusable function `estimateGasInternal`.
        // Refactoring `EthEstimateGas` to extract `estimateGasInternal(ctx, tx, blk)` is recommended.
        
        estimated, err := e.estimateGasInternal(ctx, estimateArgs, nil) 
        if err != nil {
            return nil, xerrors.Errorf("failed to estimate gas: %w", err)
        }
        res.Gas = estimated
    }

    // 6. Chain ID
    if params.ChainID != nil {
        res.ChainID = *params.ChainID
    } else {
        // Hardcode or fetch from config
        res.ChainID = ethtypes.EthUint64(buildconstants.Eip155ChainId)
    }

    return res, nil
}
```

### 4.4 Refactoring Recommendation

To make Step 5 (Gas Estimation) clean, the existing `EthEstimateGas` function in `node/impl/eth/gas.go` should be refactored. Currently, it decodes JSON params and runs logic.

**Refactor:**
Extract the core logic into:
`func (e *ethGas) estimateGasInternal(ctx context.Context, tx ethtypes.EthCall, blkParam *ethtypes.EthBlockNumberOrHash) (ethtypes.EthUint64, error)`

Then both `EthEstimateGas` and `EthFillTransaction` can call this internal helper.

---

## 5. Risks and Considerations

1.  **Performance Overhead:** `EthFillTransaction` calls `EthEstimateGas`, which triggers a VM simulation. This is computationally expensive. It should be rate-limited or cached if exposed publicly.
2.  **Nonce Race Conditions:** `MpoolGetNonce` returns the *current* valid nonce. If the user calls `fillTransaction` but doesn't sign/send immediately, and sends another transaction in the meantime, the filled nonce will become stale (invalid). This is a standard behavior for this RPC, but users should be aware.
3.  **Legacy vs EIP-1559:** Filecoin is natively EIP-1559. The implementation should default to Type 2 transactions unless the user strictly provides Legacy `GasPrice` but no EIP-1559 fields.

## 6. Conclusion

Implementing `eth_fillTransaction` is a straightforward engineering task involving:
1.  Defining a pointer-based input struct.
2.  Orchestrating `MpoolGetNonce`, fee estimation, and gas estimation.
3.  Refactoring `EthEstimateGas` slightly for reusability.

This enhancement will significantly improve the developer experience for Ethereum tools (like Hardhat, Foundry, or ethers.js) interacting with the Lotus Filecoin node, as these tools often rely on the node to provide sensible defaults.

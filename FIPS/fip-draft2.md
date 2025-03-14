---
fip: "<to be assigned>"
title: Add Support for EIP-2537 (BLS12-381 Precompiles) in the Filecoin EVM runtime
author: Aarav Mehta (aaravm), Michael Seiler (snissn)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/1135
status: Draft
type: Technical
category: Core
created: 2024-05-01
---

# FIP-<to be assigned>: Add Support for EIP-2537 (BLS12-381 Precompiles) in the Filecoin EVM runtime

## Simple Summary
This proposal adds seven new precompiles to the Filecoin Ethereum Virtual Machine (FEVM) to enable efficient operations over the BLS12-381 elliptic curve, including cryptographic primitives required for modern signature schemes and zero-knowledge proofs. These operations provide 120+ bits of security while maintaining compatibility with Ethereum's EIP-2537 standard.

## Abstract
We introduce seven precompiles supporting fundamental cryptographic operations on the BLS12-381 curve:
1. Point additions in G1/G2 groups
2. Multi-scalar multiplications (MSM) in G1/G2 
3. Pairing checks
4. Field element to curve mappings

These operations enable efficient BLS signature verification and aggregation while providing stronger security guarantees than existing BN254 precompiles. Gas costs align with computational complexity using Pippenger's algorithm optimizations for MSM operations.

## Change Motivation
The Filecoin network requires modern cryptographic primitives to support:
1. Secure storage proofs with future-proof security margins
2. Cross-chain interoperability with Ethereum's evolving ecosystem
3. Efficient aggregation techniques for large-scale decentralized storage networks

Existing BN254-based precompiles only provide 80-bit security, making them vulnerable to advances in quantum computing. BLS12-381's 120+ bit security margin addresses this while enabling new storage-related use cases like:
- Aggregate signatures for batch storage deals
- Zero-knowledge proofs for privacy-preserving storage verification
- Cross-chain bridges with Ethereum-based DeFi protocols

## Specification

### Precompile Addresses & Gas Costs

| Operation               | Address | Base Gas Cost |
|-------------------------|---------|---------------|
| G1 Addition             | 0x0b    | 375           |
| G1 Multi-scalar Mult    | 0x0c    | 12,000 + MSM discount |
| G2 Addition             | 0x0d    | 600           |
| G2 Multi-scalar Mult    | 0x0e    | 22,500 + MSM discount |
| Pairing Check           | 0x0f    | 37,700 + 32,600*k |
| Map Fp → G1             | 0x10    | 5,500         |
| Map Fp2 → G2            | 0x11    | 23,800        |

### Input/Output Encodings
- **Field Elements**: 64 bytes for Fp, 128 bytes for Fp2 (BigEndian)
- **Curve Points**: 128 bytes for G1, 256 bytes for G2 (concatenated coordinates)
- **Scalars**: 32-byte BigEndian integers
- **Pairing Inputs**: 384 bytes per (G1, G2) pair

### Critical Operations
1. **MSM Discounts**: Apply Pippenger's algorithm discounts using EIP-2537 tables
2. **Subgroup Checks**: Mandatory for pairing operations and MSM inputs
3. **Mapping Functions**: Implement IETF-compliant hash-to-curve (RFC 9380)

## Design Rationale
BLS12-381 was selected for its:
1. **Security**: 128-bit security against classical attacks
2. **Efficiency**: Optimal pairing performance vs. alternatives
3. **Ecosystem Alignment**: Direct compatibility with Ethereum's EIP-2537

The 7-precompile structure balances implementation complexity with operational flexibility. Gas costs were calculated using worst-case cycle counts from reference implementations optimized for WebAssembly execution environments.

## Backwards Compatibility
No breaking changes - existing contracts remain unaffected. New precompiles occupy previously unused addresses. Subgroup checks follow EIP-2537 validation rules to prevent invalid curve attacks.

## Test Cases
Test vectors include:
1. Edge Cases: Zero points, identity elements
2. Pairing Bilinearity: e(aP,bQ) = e(abP,Q)
3. MSM Correctness: ∑scalar_i*point_i verifications
4. Mapping Consistency: Field elements → valid curve points

Reference implementations must pass all EIP-2537 test vectors plus Filecoin-specific:
- Gas cost boundary checks
- Max-length input validations (10,000+ MSM elements)

## Security Considerations
1. **Timing Attacks**: Non-constant-time operations permitted for performance
2. **Gas Metering**: Strict input length validation prevents DoS via malformed inputs
3. **Subgroup Enforcement**: Critical for pairing correctness and G1/G2 safety
4. **Precompile Isolation**: Each operation runs in constrained WASM environment

## Incentive Considerations
1. **Fair Pricing**: MSM discounts prevent overpayment for bulk operations
2. **Storage Proofs**: Enables cost-effective batch verification for sector commitments
3. **Network Effects**: Compatibility with Ethereum tooling attracts developer talent

## Product Considerations
Enables:
1. **BLS Storage Markets**: Aggregate signatures for high-throughput deal-making
2. **ZK Storage Proofs**: Pairing-based proofs for compact replication arguments
3. **Cross-Chain Assets**: Trust-minimized bridges using BLS threshold signatures

## Implementation
Reference implementation in Rust:
- Integrates blst & arkworks-rs libraries
- WASM runtime optimizations for FVM
- Gas cost tables verified against Ethereum mainnet benchmarks

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

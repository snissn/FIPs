---
fip: "<to be assigned>"
title: Add Support for EIP-2537 (BLS12-381 Precompiles) in the Filecoin EVM runtime
author: Aarav Mehta (aaravm), Michael Seiler (snissn)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/1135
status: Draft
type: Technical
category: Core
created: 2025-03-13
---

# FIP-<to be assigned>: Add Support for EIP-2537 (BLS12-381 Precompiles) in the Filecoin EVM runtime

## Simple Summary
This proposal introduces a set of precompiles in the FEVM for efficient cryptographic operations on the BLS12-381 elliptic curve. These precompiles enable high-performance BLS signature verification and aggregation, enhancing compatibility with Ethereum tooling and supporting advanced cryptographic protocols on Filecoin.

## Abstract
This FIP proposes the addition of precompiled functions in the FEVM to support BLS12-381 curve operations, matching the functionality defined in [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537). These precompiles provide primitives for point addition, multi-scalar multiplication (MSM), field-to-curve mapping, and pairing checks, facilitating secure and performant BLS signature schemes. This enhancement increases the cryptographic capabilities of FEVM and aligns with Ethereum ecosystem standards.

## Change Motivation
BLS12-381 precompiles are essential for supporting cryptographic primitives that require high security (≥120-bit) and efficient signature aggregation. These operations are fundamental to threshold signatures, decentralized identities, zk-rollups, and consensus mechanisms. Currently, the FEVM lacks native support for BLS12-381 operations, creating a performance and compatibility gap with Ethereum-based applications and protocols that rely on this curve. This FIP bridges that gap, bringing parity with Ethereum and enabling new cryptographic use cases on Filecoin.

## Specification

This FIP introduces a set of BLS12-381 elliptic curve operations via precompiled contracts, matching the functionality and encoding rules defined in [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537). These precompiles are intended for cryptographic operations including point addition, multi-scalar multiplication, field-to-curve mapping, and pairing checks, using the BLS12-381 curve.

The following precompile addresses are proposed:

| Operation                  | Address | Description                                     |
|---------------------------|---------|-------------------------------------------------|
| `BLS12_G1ADD`             | `0x0b`  | Adds two G1 points (128-byte input, 64-byte each) |
| `BLS12_G1MSM`             | `0x0c`  | Multi-scalar multiplication over G1             |
| `BLS12_G2ADD`             | `0x0d`  | Adds two G2 points                              |
| `BLS12_G2MSM`             | `0x0e`  | Multi-scalar multiplication over G2             |
| `BLS12_PAIRING_CHECK`     | `0x0f`  | Performs a pairing check over (G1, G2) pairs    |
| `BLS12_MAP_FP_TO_G1`      | `0x10`  | Maps an Fp element to a point in G1             |
| `BLS12_MAP_FP2_TO_G2`     | `0x11`  | Maps an Fp2 element to a point in G2            |

Each operation uses the same ABI and data layout as described in EIP-2537, including:
- **Big-endian byte encoding** of points, scalars, and field elements.
- **Subgroup checks** where required (e.g. for MSMs and pairings).
- **Canonical encoding and validation rules** for input field elements.
- **Deterministic failure modes** on malformed input (e.g. invalid length, non-curve points).

The implementation ensures full compatibility with Ethereum tooling and semantics, enabling cross-chain cryptographic applications and reuse of existing test vectors and infrastructure.

## Design Rationale
The precompile design is adapted directly from Ethereum’s EIP-2537 to maximize compatibility and interoperability. The explicit support for MSMs reduces gas costs compared to sequential MUL/ADD operations. Mapping functions are included to support signature schemes like BLS that require field-to-curve mapping. Gas schedules are tuned to reflect worst-case computation while preventing DDoS vectors. Subgroup checks are enforced where necessary to ensure cryptographic correctness.

## Backwards Compatibility
This FIP introduces new precompile addresses and does not affect existing contract behavior. Contracts not using these addresses remain unaffected. The precompiles are fully opt-in and backwards-compatible.

## Test Cases

A broad suite of tests is provided with the implementation that validates the behavior of the BLS12-381 precompiles defined in [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537). These tests ensure both correctness and safety across standard, edge, and invalid input scenarios.

#### The following categories of tests are implemented:

### **Success Case Coverage**

These verify correctness and adherence to cryptographic properties:

#### **G1ADD / G2ADD**
- Point addition with valid G1/G2 inputs
- Addition with identity point (0)
- Doubling (`P + P`)
- Subtraction (`P + (-P)` → 0)
- Commutativity checks

#### **G1MSM / G2MSM**
- Weighted sums with valid `(point, scalar)` pairs
- Scalars: `0`, `1`, and random values
- Mix of valid points, including points at infinity
- Multiple pairs and varying lengths

#### **MAP_FP_TO_G1 / MAP_FP2_TO_G2**
- Mapping various valid field elements (from clean byte strings)
- Verifies resulting points lie on curve and in correct subgroup

#### **PAIRING**
- Pairing identity: `e(0, 0) = 1`, `e(P, Q) * e(P, -Q) = 1`
- Bilinearity: `e(aP, bQ) = e(P, Q)^{ab}`
- Zero contributions: `e(P, 0) = e(0, Q) = 1`
- Multi-pair checks validating associativity and multiplicative accumulation
- Mix of G1 and G2 in normal and negated forms

### **Failure Case Coverage**

These verify rejection of malformed, incomplete, or semantically invalid inputs:

#### **General Input Errors (All Opcodes)**
- Empty input
- Short input
- Extra bytes (long input)
- Invalid top bytes
- Field elements with illegal encodings
- Invalid curve points
- Point not in correct subgroup
- Mismatched input sizes (e.g., unaligned MSM pairs)

### **Edge Case Behaviors (Covered across modules)**
- Point at infinity as operand in MSM or pairing
- Scalar = 0 in MSM (yields zero contribution)
- Double vs. add equivalence in G1ADD/G2ADD
- Mixed representations and ordering (commutative symmetry)
- Subgroup validation logic invoked only *after* full field element validation

This comprehensive suite ensures correct implementation of the BLS12-381 arithmetic under realistic usage and malicious input conditions, supporting conformance with EIP-2537.

## Security Considerations

This FIP introduces a set of precompiled contracts to perform cryptographic operations over the BLS12-381 elliptic curve, consistent with Ethereum’s [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537). These include G1/G2 addition, multi-scalar multiplication (MSM), mapping from field elements to curve points, and bilinear pairing checks. The implementation leverages the well-audited `blst` library and mirrors the design and operational constraints of the Ethereum specification.

### Surface and Context

The BLS precompiles expose powerful cryptographic primitives used in aggregate signature verification, threshold cryptography, and zero-knowledge systems. As such, correctness and safety are critical. These operations act on untrusted input and therefore must defend against malformed data, invalid encoding, and subgroup-related attacks.

### Threat Model and Risks

The following threat vectors and risk areas are addressed:

#### 1. **Curve Membership and Subgroup Checks**
- Operations involving scalar multiplication and pairings **must** perform subgroup checks to prevent cofactor-related attacks and ensure that only points in the prime-order subgroups are used.
- Our implementation adheres to the subgroup-check requirements laid out in EIP-2537, applying them **after** curve membership and field validity checks to ensure consistency and avoid leaking information about malformed inputs.

#### 2. **Input Validation**
- All precompiles validate the length and structure of input bytes prior to deserialization.
- Inputs failing length checks, field modulus checks, or invalid encoding formats are rejected early with clear failure paths.
- Particular care is taken in mapping precompiles to avoid processing non-canonical field elements or incorrect byte representations.

#### 3. **Constant-Time Operations**
- All cryptographic operations are performed using the `blst` library, which is designed to run in constant time to mitigate timing-based side-channel attacks.
- This guarantees uniform execution time regardless of input values, preventing information leakage via timing differences.

#### 4. **Point at Infinity Handling**
- All operations explicitly handle identity elements and correctly return zero points where expected (e.g., scalar multiplication by zero, or adding a point to its negation).
- Pairings involving points at infinity are treated as identity elements but still undergo validation to ensure that no malformed input is accepted.

#### 5. **Error Handling and Deterministic Fails**
- The implementation provides deterministic and transparent error behavior for all malformed inputs.
- Inputs that are malformed in ways that could lead to unpredictable behavior (e.g., truncated input, invalid field elements) result in failure with no side effects.

#### 6. **Compatibility and Cross-Chain Consistency**
- Matching Ethereum’s EIP-2537 ensures consistency for cross-chain cryptographic applications and tooling.
- Test vectors and behaviors are aligned with Ethereum’s reference implementation to support shared ecosystem tooling and audits.

### Implementation Safeguards

- The implementation includes extensive unit tests covering edge cases, invalid inputs, and critical functional properties (e.g., bilinearity of pairing, subgroup correctness).
- All tests are built against known-good vectors and checked for conformance against Ethereum-compatible expectations.
- The use of precompiles restricts these operations to controlled execution contexts, reducing the attack surface compared to userland implementations.

## Incentive Considerations
Enabling efficient BLS operations enhances support for use cases like decentralized storage verification, threshold signing, and zk-rollup integration. These use cases directly improve Filecoin’s reliability, performance, and application ecosystem. While no direct protocol incentive changes are introduced, better cryptographic primitives contribute to Filecoin’s long-term network utility.

## Product Considerations
BLS precompiles improve developer experience by enabling advanced applications (e.g., zk-proof aggregation, BLS-based voting, and signature-based storage proofs). This upgrade positions Filecoin as a capable and attractive platform for cryptographically advanced dApps and aligns with emerging Ethereum tooling standards.

## Implementation
Reference implementation will be integrated into the `builtin-actors` FEVM runtime. A pull request with a working precompile set and test coverage will be submitted following FIP acceptance.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).




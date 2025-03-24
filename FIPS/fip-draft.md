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
This FIP mirrors [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537) and introduces seven precompiled functions for the following BLS12-381 curve operations:

| Operation                     | Address | Description                                   |
|-----------------------------|---------|-----------------------------------------------|
| BLS12_G1ADD                 | 0x0b    | G1 point addition                             |
| BLS12_G1MSM                 | 0x0c    | G1 multi-scalar multiplication                |
| BLS12_G2ADD                 | 0x0d    | G2 point addition                             |
| BLS12_G2MSM                 | 0x0e    | G2 multi-scalar multiplication                |
| BLS12_PAIRING_CHECK        | 0x0f    | Pairing check between G1/G2 point pairs       |
| BLS12_MAP_FP_TO_G1         | 0x10    | Map Fp element to G1                          |
| BLS12_MAP_FP2_TO_G2        | 0x11    | Map Fp2 element to G2                         |

Detailed gas cost schedules, input/output encoding formats, subgroup checks, and behavior on invalid input match the original EIP-2537 specification and are to be fully implemented within FEVM.

## Design Rationale
The precompile design is adapted directly from Ethereum’s EIP-2537 to maximize compatibility and interoperability. The explicit support for MSMs reduces gas costs compared to sequential MUL/ADD operations. Mapping functions are included to support signature schemes like BLS that require field-to-curve mapping. Gas schedules are tuned to reflect worst-case computation while preventing DDoS vectors. Subgroup checks are enforced where necessary to ensure cryptographic correctness.

## Backwards Compatibility
This FIP introduces new precompile addresses and does not affect existing contract behavior. Contracts not using these addresses remain unaffected. The precompiles are fully opt-in and backwards-compatible.

## Test Cases
Test vectors and functional properties will follow the EIP-2537 test suite, including:
- Point addition and multiplication associativity
- Pairing bilinearity and non-degeneracy
- MSM performance benchmarks
- Subgroup validation

A full suite of test vectors and performance benchmarks will be published alongside implementation.

## Security Considerations
BLS precompiles introduce cryptographic attack surfaces if implemented incorrectly. This FIP mandates subgroup checks for MSM and pairing operations to prevent rogue key attacks. Precompiles must return errors on invalid encoding or input. Gas burning on errors follows EVM precompile conventions. Constant-time operations are not strictly required but recommended.

## Incentive Considerations
Enabling efficient BLS operations enhances support for use cases like decentralized storage verification, threshold signing, and zk-rollup integration. These use cases directly improve Filecoin’s reliability, performance, and application ecosystem. While no direct protocol incentive changes are introduced, better cryptographic primitives contribute to Filecoin’s long-term network utility.

## Product Considerations
BLS precompiles improve developer experience by enabling advanced applications (e.g., zk-proof aggregation, BLS-based voting, and signature-based storage proofs). This upgrade positions Filecoin as a capable and attractive platform for cryptographically advanced dApps and aligns with emerging Ethereum tooling standards.

## Implementation
Reference implementation will be integrated into the `builtin-actors` FEVM runtime. A pull request with a working precompile set and test coverage will be submitted following FIP acceptance.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).




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
This proposal introduces a suite of precompiles for performing BLS12-381 curve operations within the Filecoin Ethereum Virtual Machine (FEVM). These precompiles enable efficient cryptographic operations—including point addition, multi-scalar multiplication, pairing checks, and field-to-curve mappings—supporting BLS signature verification and public key aggregation with over 120 bits of security.

## Abstract
This FIP proposes to add seven new precompiles to the FEVM that implement operations on the BLS12-381 elliptic curve. The precompiles cover the following functionalities:
- **G1 Addition and MSM:** Operations for point addition and multi-scalar multiplication (MSM) on the base curve (G1).
- **G2 Addition and MSM:** Similar operations on the twist curve (G2) defined over a quadratic extension.
- **Pairing Check:** A pairing precompile that verifies the aggregation of BLS signatures by checking the pairing equation.
- **Field-to-Curve Mapping:** Mapping functions for converting field elements in Fp and Fp2 into corresponding curve points in G1 and G2 respectively.

Each operation is assigned a dedicated precompile address and comes with a carefully defined gas cost schedule that factors in both fixed and variable costs (e.g., discount tables for MSMs). The precompiles follow a strict ABI for encoding inputs and outputs, ensuring interoperability between different FEVM implementations. In addition, error conditions are explicitly defined, with any erroneous input resulting in a complete burn of the provided gas, thereby aligning with the FEVM’s existing security and consistency models.

## Change Motivation
The current FEVM precompiles include support for BN254 curve operations, which offer only 80 bits of security. As cryptographic best practices evolve, there is a pressing need for operations over the BLS12-381 curve to achieve higher security levels (120+ bits). BLS signatures, favored for their aggregation and scalability properties, require efficient implementations that are infeasible to perform solely via smart contract code. By integrating these precompiles, the FEVM will be able to efficiently perform BLS signature verification and related operations, enhancing both security and performance for a wide range of applications—from decentralized storage proofs to complex multi-signature schemes.

## Specification

### Precompile Operations and Constants

Seven distinct precompiles will be added with the following designated operations and addresses:

| Operation              | Precompile Address | Description |
|------------------------|--------------------|-------------|
| **BLS12_G1ADD**        | `0x0b`             | Point addition in G1 (base field curve) with a fixed cost (e.g., 375 gas). |
| **BLS12_G1MSM**        | `0x0c`             | Multi-scalar multiplication (MSM) in G1 with a gas cost computed via a discount formula. |
| **BLS12_G2ADD**        | `0x0d`             | Point addition in G2 (curve over quadratic extension) with a fixed cost (e.g., 600 gas). |
| **BLS12_G2MSM**        | `0x0e`             | MSM in G2 with a variable gas cost using a discount table. |
| **BLS12_PAIRING_CHECK**| `0x0f`             | Aggregates pairing checks over pairs of (G1, G2) points with gas cost proportional to the number of pairs. |
| **BLS12_MAP_FP_TO_G1** | `0x10`             | Maps an Fp element to a G1 curve point with a fixed cost (e.g., 5500 gas). |
| **BLS12_MAP_FP2_TO_G2**| `0x11`             | Maps an Fp2 element to a G2 curve point with a fixed cost (e.g., 23800 gas). |

### Input/Output Encoding and ABI Details

- **Field Elements:** 
  - Fp elements are encoded as 64-byte big-endian numbers (with the top 16 bytes zeroed).
  - Fp2 elements are encoded by concatenating two Fp encodings (totaling 128 bytes).

- **Point Encoding:** 
  - G1 points are encoded as the concatenation of two Fp elements (128 bytes total).
  - G2 points are encoded as the concatenation of two Fp2 elements (256 bytes total).
  - The “point of infinity” is encoded as all-zero bytes (128 for G1, 256 for G2).

- **Scalar Encoding:** 
  - Scalars are encoded as 32-byte big-endian numbers.

- **ABI for Operations:** 
  - Each precompile’s ABI is strictly defined. For example, G1 addition expects 256 bytes (two G1 points) and outputs 128 bytes, while MSM operations accept a variable-length input and output a single point.

- **Error Handling:** 
  - Any malformed or invalid input (e.g., incorrect length or non-curve points) must cause the precompile to return an error and burn all supplied gas.

### Gas Schedule
Gas costs are defined relative to existing precompiles (e.g., EcRecover) with both fixed and variable components:
- **Fixed Costs:** For operations like G1/G2 addition and field-to-curve mappings.
- **Variable Costs:** For MSM and pairing operations, gas is computed based on the number of point-scalar pairs or (G1, G2) pairs. Discount tables (with defined caps) ensure that the cost scales sub-linearly with the number of inputs.
- Detailed pseudocode is provided to compute the gas for MSM and pairing checks, ensuring consistency across FEVM implementations.

## Design Rationale
The decision to include dedicated precompiles for BLS12-381 operations is driven by the need for high-security cryptographic primitives and efficiency in signature verification. By using multi-scalar multiplication (MSM) via Pippenger's algorithm, significant gas savings are realized over iterative multiplication and addition. Additionally, combining both single point operations and MSM into a unified interface avoids redundant precompiles, reducing complexity. The design also mirrors similar efforts in Ethereum (EIP-2537), thereby aligning Filecoin’s FEVM with prevailing cryptographic standards and ensuring interoperability.

## Backwards Compatibility
These precompiles are additive and do not modify the existing FEVM opcode set. Contracts not using the new BLS precompiles will continue to operate normally. However, all inputs must now adhere to stricter encoding rules (e.g., zero-checks for “point of infinity”) and explicit subgroup validations are required for MSM and pairing operations. Implementations must handle these cases consistently to avoid consensus issues.

## Test Cases
Test cases should cover:
1. **Valid Operations:**  
   - Correct encoding and successful execution of G1/G2 point addition.
   - Valid MSM operations with multiple pairs.
   - Correct mapping from Fp and Fp2 to G1 and G2.
   - Successful pairing checks that satisfy the equation.
2. **Error Conditions:**  
   - Malformed input lengths.
   - Invalid field element encodings.
   - Points not on the curve or failing subgroup checks.
   - Edge cases with the “point of infinity.”
3. **Benchmarking:**  
   - Performance comparisons for varying input sizes for MSM and pairing operations.

## Security Considerations
While following the specification minimizes security risks, potential issues include:
- **Non-constant Time Operations:**  
  The precompiles are not required to be constant time; implementations must document any side-channel risks.
- **Error Handling:**  
  All errors result in gas being burned, preventing potential exploits from partial execution.
- **Subgroup Checks:**  
  Robust subgroup validation is mandatory to prevent invalid point attacks.
- **Input Validation:**  
  Strict checks on input encoding prevent malformed data from being processed.

## Incentive Considerations
The introduction of these precompiles enhances cryptographic efficiency on the FEVM, reducing gas costs for signature verification and public key aggregation. This encourages the use of BLS signatures in decentralized applications, ultimately benefiting the network by reducing computation overhead and incentivizing the development of more secure and scalable smart contracts.

## Product Considerations
By supporting BLS12-381 operations natively, Filecoin improves its standing as an advanced, secure, and interoperable EVM-compatible platform. These precompiles facilitate sophisticated cryptographic protocols, such as aggregate signature schemes and threshold cryptography, that are critical for decentralized storage and other blockchain applications. This improvement will attract developers who rely on high-security cryptography, expanding the ecosystem of applications on Filecoin.

## Implementation
Reference implementations of the BLS precompiles are available in Rust:
- One implementation integrates with OpenEthereum, leveraging existing EIP-196 code as a base.
- A dedicated implementation for the FEVM (e.g., in Geth) is under active development.
These implementations follow the detailed pseudocode for gas calculation, error handling, and the precise arithmetic operations defined in this FIP.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


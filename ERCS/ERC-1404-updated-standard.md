---
eip: 1404
title: Transfer Restriction Interface Extension
description: A transfer-restriction interface with machine-readable codes for fungible and non-fungible tokens.
author: CMTA, Ryan Sauge (@rya-sge)
discussions-to: https://ethereum-magicians.org/t/erc-1404-extended-transfer-restriction-interface/0
status: Draft
type: Standards Track
category: ERC
created: 2026-04-22
requires: 20, 165, 721, 1155
---

## Abstract

This EIP defines an extension of [ERC-1404](./eip-1404.md) for exposing transfer restriction logic across [ERC-20](./eip-20.md), [ERC-721](./eip-721.md), and [ERC-1155](./eip-1155.md) tokens. It enables clients to pre-check transfer eligibility and retrieve machine-readable restriction codes alongside human-readable messages. The interface supports optional spender-aware restriction detection and supports [ERC-165](./eip-165.md) discovery.

## Motivation

Tokens across major standards may be subject to transfer restrictions due to regulation, compliance rules, lockups, blacklists, or issuer-defined policies.

When restrictions are enforced only by reverts, wallets and off-chain systems have no standardized machine-readable reason for failure. This causes poor UX and inconsistent integration behavior.

Related standards such as [ERC-7943](./eip-7943.md) and [ERC-7551](./eip-7551.md) define transfer eligibility checks, but typically rely on boolean outcomes. This EIP extends the original [ERC-1404](./eip-1404.md) code-and-message approach to newer token models and spender/operator transfer flows.

## Specification

### Design Principles

- Restriction detection MUST be callable without modifying state.
- Restriction code `0` MUST indicate an unrestricted transfer.
- Non-zero restriction codes are implementation-defined.
- Spender-aware detection is OPTIONAL.

### Interfaces

```solidity
pragma solidity ^0.8.0;

interface IERC1404Fungible {
    function detectTransferRestriction(
        address from,
        address to,
        uint256 value
    ) external view returns (uint256);

    function messageForTransferRestriction(
        uint256 restrictionCode
    ) external view returns (string memory);
}

interface IERC1404FungibleSpender {
    function detectTransferRestrictionFrom(
        address spender,
        address from,
        address to,
        uint256 value
    ) external view returns (uint256);
}

interface IERC1404NFT {
    function detectTransferRestriction(
        address from,
        address to,
        uint256 tokenId,
        uint256 amount
    ) external view returns (uint256);

    function messageForTransferRestriction(
        uint256 restrictionCode
    ) external view returns (string memory);
}

interface IERC1404NFTSpender {
    function detectTransferRestrictionFrom(
        address spender,
        address from,
        address to,
        uint256 tokenId,
        uint256 amount
    ) external view returns (uint256);
}
```

### ERC-165

Implementations MAY support [ERC-165](./eip-165.md) discovery for any interface above.

If [ERC-165](./eip-165.md) is implemented, `supportsInterface(bytes4)` SHOULD return `true` for each implemented ERC-1404 extended interface.

### Transfer Semantics

An implementation:

- MUST behave as the underlying token standard when restriction code is `0`.
- MUST block restricted transfers when a non-zero code applies.
- Detection and enforcement logic MUST be consistent.
- Restriction codes are metadata and MUST NOT be used as a sole authorization primitive.
- SHOULD prefer custom errors over revert strings.
- SHOULD include the restriction code in revert data where possible.

If spender-aware interfaces are implemented, delegated/operator transfer paths SHOULD use the spender-aware check.

## Rationale

This design keeps restriction logic machine-readable and wallet-friendly:

- Numeric codes are cheap and composable.
- Message lookup allows user-facing explanations.
- Spender-aware checks cover `transferFrom` and operator flows.
- The interface remains additive to existing token standards.

## Backwards Compatibility

The standard is additive for [ERC-20](./eip-20.md), [ERC-721](./eip-721.md), and [ERC-1155](./eip-1155.md).

Existing integrations remain compatible with base token behavior. Integrators that detect this extension can run pre-flight restriction checks and display restriction messages.

## Test Cases

1. `detectTransferRestriction(...)` returns `0` for unrestricted transfers.
2. Restricted transfers return non-zero codes.
3. `messageForTransferRestriction(code)` returns deterministic output for each used non-zero code.
4. Actual transfer execution enforces the same policy as detection logic.
5. If spender-aware interfaces are implemented, delegated/operator transfer paths use the spender-aware restriction check.
6. If [ERC-165](./eip-165.md) is implemented, `supportsInterface` reports implemented interfaces correctly.

## Reference Implementation

Reference implementation: TBD.

## Security Considerations

Off-chain checks are advisory only; transfer functions remain the source of truth.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

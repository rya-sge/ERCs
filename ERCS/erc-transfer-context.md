---
eip: 0
title: Transfer Context Hook for Compliance Rules
description: A transfer-hook context interface for fungible and non-fungible token compliance integrations.
author: Ryan Sauge (@rya-sge)
discussions-to: https://ethereum-magicians.org/t/transfer-context-hook-for-compliance-rules/0
status: Draft
type: Standards Track
category: ERC
created: 2026-03-03
requires: 165
---

## Abstract

This ERC defines a single, standardized entrypoint that token contracts can call to notify external compliance rules about a successful transfer. It introduces two dedicated interfaces: one for fungible tokens and one for non-fungible/multi-token standards. A unified context struct allows rules to receive the full transfer intent (including optional operator/spender and arbitrary data) without requiring multiple hook signatures.

## Motivation

Tokens that integrate external compliance logic often need to call rule contracts during `transfer`, `transferFrom`, or `safeTransferFrom`. Without a standard hook, rules must expose several signatures and token developers must add bespoke integrations. This ERC provides:

- A unique, consistent entrypoint for transfer hooks.
- A clear separation between fungible and non-fungible flows.
- A minimal context struct that carries the relevant data for rule evaluation or post-transfer state updates.

This standard reduces integration complexity, avoids signature sprawl, and improves interoperability between tokens and rule engines.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", and "MAY" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Transfer Context (Fungible)

Tokens representing fungible balances (e.g. [ERC-20](./eip-20.md)) MUST use the following interface:

```solidity
interface IERCXXXXFungibleTransferContext {
    struct FungibleTransferContext {
        bytes4 selector;  // Selector of the originating token function, or bytes4(0) if unavailable or unused.
        address sender;   // Address that initiated the transfer (msg.sender in the token contract).
        address from;     // Token sender.
        address to;       // Token recipient.
        uint256 value;    // Amount transferred.
        bytes data;       // Optional data from the originating call.
    }

    error TransferContext_InvalidSelector(bytes4 selector);

    function transferred(FungibleTransferContext calldata ctx) external;
}
```

### Transfer Context (Non-Fungible / Multi-Token)

Tokens representing non-fungible or multi-token balances (e.g. [ERC-721](./eip-721.md), [ERC-1155](./eip-1155.md)) MUST use the following interface:

```solidity
interface IERCXXXXNonFungibleTransferContext {
    struct MultiTokenTransferContext {
        bytes4 selector;  // Selector of the originating token function, or bytes4(0) if unavailable or unused.
        address sender;   // Address that initiated the transfer (msg.sender in the token contract).
        address from;     // Token sender.
        address to;       // Token recipient.
        uint256 value;    // Amount transferred (1 for ERC-721).
        uint256 tokenId;  // Token identifier.
        bytes data;       // Optional data from the originating call.
    }

    error TransferContext_InvalidSelector(bytes4 selector);

    function transferred(MultiTokenTransferContext calldata ctx) external;
}
```

### Token Requirements

- A token that supports this ERC MUST call `transferred(...)` exactly once for each successful transfer.
- Token transfer execution and hook invocation MUST be atomic: if the transfer does not succeed, the transaction MUST revert.
- A token that supports this ERC SHOULD call `transferred(...)` exactly once for each successful burn (removing tokens from circulation) and mint (creating new tokens).
- `sender` MUST always be the address that called the token function (i.e. `msg.sender` in the token contract). For direct transfers this equals `from`; for operator-based transfers this is the operator/spender.
- For ERC-721 transfers, `value` MUST be `1`.
- `selector` MUST be the selector of the originating token function (`transfer`, `transferFrom`, `safeTransferFrom`, etc.) when any configured hook requires it. Tokens MAY set `selector` to `bytes4(0)` when no configured hook requires it. If the token developer does not know whether configured hooks use `selector`, they SHOULD pass the correct selector.
- `data` SHOULD carry any calldata payload associated with the transfer (e.g. [ERC-1155](./eip-1155.md) `data` or [ERC-777](./eip-777.md) `data`). If no data is available, `data` SHOULD be empty.
- `from` MUST be `address(0)` when minting, and `to` MUST be `address(0)` when burning, if those flows are integrated into the same hook.

### [ERC-165](./eip-165.md) Support

Hook contracts implementing this ERC MUST implement [ERC-165](./eip-165.md) so tokens can detect supported interfaces before calling the hook.

- A hook contract that supports the fungible interface MUST return `true` for `supportsInterface(type(IERCXXXXFungibleTransferContext).interfaceId)`.
- A hook contract that supports the multi-token interface MUST return `true` for `supportsInterface(type(IERCXXXXNonFungibleTransferContext).interfaceId)`.
- Hook contracts MUST return `false` for `supportsInterface(0xffffffff)`.
- Tokens MAY call `supportsInterface(...)` before invoking `transferred(...)` to verify compatibility. If `supportsInterface(...)` returns `false`, the token SHOULD treat the hook as not supporting the interface and handle accordingly.

### Hook Requirements

- Hook contracts SHOULD treat `sender != from && to != address(0) && from != address(0)`  as a transfer with allowance.
- Hook contracts SHOULD treat `to == address(0) && sender == from` as a self burn.
- Hook contracts SHOULD treat `to == address(0) && sender != from` as a burn with the sender as operator.
- Hook contracts that apply selector-dependent logic MUST revert with `TransferContext_InvalidSelector` when `selector` is `bytes4(0)` or unrecognized.
- Hook contracts that do not use `selector` SHOULD ignore it and MUST NOT revert solely because `selector` is `bytes4(0)`.

## Rationale

Providing a single entrypoint for transfer hooks reduces gas and code duplication. The `sender` field allows a rule to distinguish direct transfers from operator-based transfers without having to expose two public signatures. The `selector` is included to allow future extensibility for additional transfer-like flows without changing the interface. The `data` field lets tokens carry ancillary payloads without creating new hook signatures.

### Comparison with [ERC-3643](./eip-3643.md) `ICompliance`

[ERC-3643](./eip-3643.md)'s `ICompliance` defines multiple functions (`canTransfer`, `transferred`, `created`, `destroyed`) and requires the compliance contract to expose the full interface. This ERC is narrower in scope: it defines a single, optional hook that tokens can call to notify external rules about a successful transfer. It does not replace [ERC-3643](./eip-3643.md) compliance checks or the RuleEngine model; it is a lightweight, uniform notification mechanism that can coexist with [ERC-3643](./eip-3643.md) integrations.

#### Advantages of This ERC

- **Single entrypoint:** one hook signature per token type (fungible vs multi-token).
- **Lower integration friction:** token developers do not need to wire multiple overloads.
- **Forward-compatible context:** `selector` and `data` allow future extensions without breaking the interface.
- **Optional adoption:** can be added alongside ERC-3643 without changing token semantics.

Separating fungible and non-fungible interfaces preserves clarity while keeping each context minimal.

## Backwards Compatibility

This ERC is fully backwards compatible with [ERC-20](./eip-20.md), [ERC-721](./eip-721.md), and [ERC-1155](./eip-1155.md). Tokens may adopt it without changing existing transfer semantics; the hook is additive.

## Test Cases

1. A compliant token calls `transferred(...)` exactly once for each successful transfer.
2. `sender` equals the caller of the token transfer function (`msg.sender` in token context).
3. For [ERC-721](./eip-721.md), `value` is `1`.
4. Hook implementations that require selector validation revert with `TransferContext_InvalidSelector` on invalid selector input.
5. Hook implementations that do not use selector do not revert solely because `selector == bytes4(0)`.
6. If [ERC-165](./eip-165.md) is implemented, `supportsInterface` reports implemented interfaces correctly.

## Reference Implementation

See `src/rules/interfaces/ITransferContext.sol` for a reference interface used in this repository.

## Security Considerations

- Tokens should call the hook before updating balances and total supply.
- Hook contracts should be designed to avoid reentrancy side effects.
- Hook contracts should protect the function through access control or assume they can be called by any token unless additional access control is enforced.
- Tokens should guard against misconfigured hook addresses that could revert or consume excessive gas.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

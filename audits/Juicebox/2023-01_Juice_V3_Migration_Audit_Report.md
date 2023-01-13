# Audit Report - Juice V3 Migration

|             |                                                                           |
| ----------- | ------------------------------------------------------------------------- |
| **Date**    | January 2023                                                              |
| **Auditor** | Bernd Artmüller ([@berndartmueller](https://twitter.com/berndartmueller)) |
| **Status**  | **Final**                                                                 |

# Table of Contents

- [Scope](#scope)
- [License](#license)
- [Disclaimer](#disclaimer)
- [Severity classification](#severity-classification)
- [Summary](#summary)
- [Findings](#findings)
- [Appendix](#appendix)

# Scope

The review focused on the commit hash [`c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464`](https://github.com/jbx-protocol/juice-v3-migration/tree/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464) of the public https://github.com/jbx-protocol/juice-v3-migration GitHub repository. The list of files in scope can be found in [Appendix 1](#Appendix-1).

# License

This audit report is licensed under a [Creative Commons - Attribution-NoDerivatives 4.0 International (CC BY-ND 4.0)](https://creativecommons.org/licenses/by-nd/4.0/) license.

# Disclaimer

_This audit report should not be taken as a guarantee that all vulnerabilities have been identified and addressed. While every effort has been made to thoroughly review the smart contracts for potential vulnerabilities, it is not possible to absolutely ensure the complete absence of vulnerabilities. The audit is a time- and resource-limited engagement. As such, it is not possible to guarantee that all vulnerabilities will be identified or that the smart contracts will be completely secure following the audit._

_Copyright of this report remains with the author._

# Severity classification

| Severity         | Description                                                                                                                                                                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ![critical]      | A **directly** exploitable security vulnerability that leads to stolen/lost/locked/compromised assets or catastrophic denial of service.                                                                                                       |
| ![high]          | A security vulnerability or bug that can affect the correct functioning of the system, lead to incorrect states or denial of service. It may not be directly exploitable or may require certain, external conditions in order to be exploited. |
| ![medium]        | Assets not at direct risk, but the function of the protocol or its availability could be impacted, or leak value with a hypothetical attack path with stated assumptions, but external requirements.                                           |
| ![minor]         | A violation of common best practices or incorrect usage of primitives, which may not currently have a major impact on security, but may do so in the future or introduce inefficiencies.                                                       |
| ![informational] | Non-critical comments, recommendations or potential optimizations, not relevant to security. Code maintainers should use their own judgment as to whether to address such issues.                                                              |

# Summary

The audited contracts contain **0 critical** issues, **0 high severity** issues, **1 medium** severity issue, **6 minor** issues and **6 informational** issues.

| #   | Title                                                                                                                                                                                                  | Severity         | Status      |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------- | ----------- |
| 1   | [Locked V1 tokens are not considered and can prevent migration](#1-locked-v1-tokens-are-not-considered-and-can-prevent-migration--))                                                                   | ![Medium]        | ![resolved] |
| 2   | [The total supply calculation in `JBV3Token.totalSupply` can potentially revert](#2-the-total-supply-calculation-in-jbv3tokentotalsupply-can-potentially-revert--))                                    | ![minor]         | ![ack]      |
| 3   | [The total supply calculation in `JBV3Token.totalSupply` is vulnerable to overflows](#3-the-total-supply-calculation-in-jbv3tokentotalsupply-is-vulnerable-to-overflows--))                            | ![minor]         | ![ack]      |
| 4   | [Migrating V1 and V2 tokens is susceptible to overflows leading to incorrectly minted V3 tokens](#4-migrating-v1-and-v2-tokens-is-susceptible-to-overflows-leading-to-incorrectly-minted-v3-tokens--)) | ![minor]         | ![ack]      |
| 5   | [Deploying and attaching a misconfigured V3 token is irreversible](#5-deploying-and-attaching-a-misconfigured-v3-token-is-irreversible--))                                                             | ![minor]         | ![ack]      |
| 6   | [Migrating V2 tokens with a custom token contract is potentially unsafe](#6-migrating-v2-tokens-with-a-custom-token-contract-is-potentially-unsafe--))                                                 | ![minor]         | ![resolved] |
| 7   | [Erroneous migration of either V1 or V2 tokens will prevent migration of the other](#7-erroneous-migration-of-either-v1-or-v2-tokens-will-prevent-migration-of-the-other--))                           | ![minor]         | ![ack]      |
| 8   | [Deprecated import for OpenZeppelin's `ERC20Permit`](#8-deprecated-import-for-openzeppelins-erc20permit-))                                                                                             | ![Informational] |             |
| 9   | [Misleading comments](#9-misleading-comments-))                                                                                                                                                        | ![Informational] |             |
| 10  | [Migrating `0` tokens will emit a `Transfer` event](#10-migrating-0-tokens-will-emit-a-transfer-event-))                                                                                               | ![Informational] |             |
| 11  | [Event parameters are not indexed](#11-event-parameters-are-not-indexed-))                                                                                                                             | ![Informational] |             |
| 12  | [Use `calldata` instead of `memory` for function parameters](#12-use-calldata-instead-of-memory-for-function-parameters-))                                                                             | ![Informational] |             |
| 13  | [Spelling issues](#13-spelling-issues-))                                                                                                                                                               | ![Informational] |             |

# Findings

## Medium Risk Findings (1)

### 1. Locked V1 tokens are not considered and can prevent migration ![Medium] ![resolved]

> The suggested recommendation has been implemented in the pull request [**#14**](https://github.com/jbx-protocol/juice-v3-migration/pull/14).

**Context:** [JBV3Token.sol#L303-L306](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L303-L306)

**Description:** Staked V1 tokens can be locked by their holder, preventing them from being redeemed and transferred. However, the `JBV3Token._migrateV1Tokens` function does not consider locked tokens and will try to migrate the entire staked balance of the `msg.sender`. Suppose the V1 token holder has parts of the staked V1 tokens locked. The [`TicketBooth.transfer`](https://github.com/jbx-protocol/juice-contracts-v1/blob/71fd42afb0ef0d51606019d9a17dcb746505efd5/contracts/TicketBooth.sol#L531-L532) function will then revert due to insufficient, unlocked V1 tokens, preventing the migration of the holders available, unlocked V1 tokens as well as potentially migrating V2 tokens.

Please note that unlocking V1 tokens can not be possible in certain situations where an operator has locked the tokens.

```solidity
function _migrateV1Tokens() internal returns (uint256 v3TokensToMint) {
  // No V1 tokens to migrate if a V1 project ID isn't stored or a Ticket Booth isn't stored.
  if (v1ProjectId == 0 || address(v1TicketBooth) == address(0)) return 0;

  // Keep a local reference to the the project's V1 token instance.
  ITickets _v1Token = v1TicketBooth.ticketsOf(v1ProjectId);

  // Get a reference to the migrating account's unclaimed balance.
  uint256 _tokensToMintFromUnclaimedBalance = v1TicketBooth.stakedBalanceOf(
    msg.sender,
    v1ProjectId
  );

  // Get a reference to the migrating account's ERC20 balance.
  uint256 _tokensToMintFromERC20s = _v1Token == ITickets(address(0))
    ? 0
    : _v1Token.balanceOf(msg.sender);

  // Calculate the amount of V3 tokens to mint from the total tokens being migrated.
  unchecked {
    v3TokensToMint = _tokensToMintFromERC20s + _tokensToMintFromUnclaimedBalance;
  }

  // Return if there's nothing to mint.
  if (v3TokensToMint == 0) return 0;

  // Transfer V1 ERC20 tokens to this contract from the msg sender if needed.
  if (_tokensToMintFromERC20s != 0)
    IERC20(_v1Token).transferFrom(msg.sender, address(this), _tokensToMintFromERC20s);

  // Transfer V1 unclaimed tokens to this contract from the msg sender if needed.
  if (_tokensToMintFromUnclaimedBalance != 0)
    v1TicketBooth.transfer(
      msg.sender,
      v1ProjectId,
      _tokensToMintFromUnclaimedBalance,
      address(this)
    );
}
```

#### Recommendation

Consider the locked V1 tokens when migrating V1 tokens.

## Minor Findings (6)

### 2. The total supply calculation in `JBV3Token.totalSupply` can potentially revert ![minor] ![ack]

**Context:** [contracts/JBV3Token.sol#L89](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L89)

**Description:** The `JBV3Token.totalSupply` function returns the sum of the total supply of the V3 token and the total supply of all unmigrated V1 and V2 tokens. However, the entire function will revert if either of the two external calls to gather the V1 and V2 `totalSupply` reverts. This could be caused by malicious intent or accidentally, for example, if a custom V2 token contract is self-destructed. Well after the migration is completed, old V1 and V2 token contracts impose a risk of reverts.

```solidity
function totalSupply() public view override returns (uint256) {
  uint256 _nonMigratedSupply;

  // If a V1 token is set get the remaining non-migrated supply.
  if(v1ProjectId != 0 && address(v1TicketBooth) != address(0)) {
    _nonMigratedSupply = v1TicketBooth.totalSupplyOf(v1ProjectId)
      - v1TicketBooth.balanceOf(address(this), v1ProjectId);
  }

  if (address(v2TokenStore) != address(0)) {
    _nonMigratedSupply += v2TokenStore.totalSupplyOf(projectId) -
      v2TokenStore.balanceOf(address(this), projectId);
  }

  return
    super.totalSupply() +
    _nonMigratedSupply;
}
```

#### Recommendation

Consider adding functionality to stop the migration and only return the V3 token's total supply from the `JBV3Token.totalSupply` function once the migration is stopped.

### 3. The total supply calculation in `JBV3Token.totalSupply` is vulnerable to overflows ![minor] ![ack]

**Context:** [contracts/JBV3Token.sol#L89](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L89)

**Description:** The `JBV3Token.totalSupply` function returns the sum of the total supply of the V3 token and the total supply of all unmigrated V1 and V2 tokens. However, the combined token supply of V1 and V2 tokens may overflow the `uint256` type, causing the function to revert.

```solidity
function totalSupply() public view override returns (uint256) {
  uint256 _nonMigratedSupply;

  // If a V1 token is set get the remaining non-migrated supply.
  if(v1ProjectId != 0 && address(v1TicketBooth) != address(0)) {
    _nonMigratedSupply = v1TicketBooth.totalSupplyOf(v1ProjectId)
      - v1TicketBooth.balanceOf(address(this), v1ProjectId);
  }

  if (address(v2TokenStore) != address(0)) {
    _nonMigratedSupply += v2TokenStore.totalSupplyOf(projectId) -
      v2TokenStore.balanceOf(address(this), projectId);
  }

  return
    super.totalSupply() +
    _nonMigratedSupply;
}
```

#### Recommendation

Consider performing a check to ensure the combined total token supply does not overflow.

### 4. Migrating V1 and V2 tokens is susceptible to overflows leading to incorrectly minted V3 tokens ![minor] ![ack]

**Context:** [contracts/JBV3Token.sol#L277-L283](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L277-L283)

**Description:** The `JBV3Token.migrate` function calls the internal `_migrateV1Tokens` and `_migrateV2Tokens` functions, which return the number of tokens to migrate. The `migrate` function then adds the number of V1 and V2 tokens to migrate and mints the corresponding amount of V3 tokens. However, due to the use of the `unchecked` Solidity keyword, the arithmetic operation may overflow, causing the function to mint an incorrect amount of V3 tokens while still transferring the correct amount of V1 and V2 tokens to the V3 token contract.

```solidity
function migrate() external {
  uint256 _tokensToMint;

  unchecked {
    // Add the number of V1 tokens to migrate.
    _tokensToMint += _migrateV1Tokens();

    // Add the number of V2 tokens to migrate.
    _tokensToMint += _migrateV2Tokens();
  }

  // Mint tokens as needed.
  _mint(msg.sender, _tokensToMint);
}
```

#### Recommendation

Consider removing the `unchecked` Solidity keyword to ensure the migration function reverts if the combined token supply overflows.

### 5. Deploying and attaching a misconfigured V3 token is irreversible ![minor] ![ack]

**Context:** [contracts/JBV3TokenDeployer.sol#L93](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3TokenDeployer.sol#L93)

**Description:** The `JBV3TokenDeployer.deploy` function deploys a new V3 token contract and attaches it to the V3 project with the provided id `_projectId`. This function allows the deployer to set the optional `_v1TicketBooth` and `_v2TokenStore` addresses as well as the V1 project id `_v1ProjectId`, which are then set as immutable values in the V3 token contract. Once the V3 token contract is attached to the V3 project via the `tokenStore.setFor` function, it is not possible to change the token contract or the respective values. This can cause multiple issues, worst case, the V3 token is rendered unusable.

#### Recommendation

Consider mitigating the possible error surface using booleans as function parameters for the `JBV3TokenDeployer.deploy` function to indicate the interest in migrating V1 and V2 tokens.

### 6. Migrating V2 tokens with a custom token contract is potentially unsafe ![minor] ![resolved]

> A reentrancy guard has been added to the `JBV3Token.migrate` function as per the suggested recommendation in the pull request [**#14**](https://github.com/jbx-protocol/juice-v3-migration/pull/14).

**Context:** [contracts/JBV3Token.sol#L369](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L369)

**Description:** V2 tokens are potentially custom token contracts and thus need to be taken special care of. The `JBV3Token._migrateV2Tokens` function calls the potentially unsafe, external `_v2Token.transferFrom` function to transfer the V2 tokens to the V3 token contract. For example, if the token transfer fails without reverting, instead returning `false`, V3 tokens are minted nevertheless. Additionally, reentrancy is possible in case the V2 token contract employs transfer hooks. Worst case, more V3 tokens are minted than the V2 token owner (i.e., `msg.sender`) is entitled to.

#### Recommendation

Consider using OpenZeppelin’s `SafeERC20` `safeTransferFrom` function to handle non-standard-compliant tokens, check received token balances as well as adding a reentrancy guard to the `JBV3Token.migrate` function as a safety precaution.

### 7. Erroneous migration of either V1 or V2 tokens will prevent migration of the other ![minor] ![ack]

**Context:** [JBV3Token.sol#L274](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L274)

**Description:** The `JBV3Token.migrate` function, if configured accordingly, migrates both V1 and V2 tokens. If either of the two internal functions `_migrateV1Tokens` or `_migrateV2Tokens` reverts, the entire transaction will revert, preventing the migration of the other token. This can be caused by a custom V2 token contract in various ways, for example, if the token contract is self-destructed or if the required `JBOperations.TRANSFER` permission is not granted to the `JBV3Token` contract or revoked.

```solidity
function migrate() external {
  uint256 _tokensToMint;

  unchecked {
    // Add the number of V1 tokens to migrate.
    _tokensToMint += _migrateV1Tokens();

    // Add the number of V2 tokens to migrate.
    _tokensToMint += _migrateV2Tokens();
  }

  // Mint tokens as needed.
  _mint(msg.sender, _tokensToMint);
}
```

#### Recommendation

Consider adding additional separate functions for migrating V1 and V2 tokens separately.

## Informational Findings (6)

### 8. Deprecated import for OpenZeppelin's `ERC20Permit` ![Informational]

**Context:** [contracts/JBV3Token.sol#L9](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L9)

**Description:** The current import of `draft-ERC20Permit.sol` is deprecated as [EIP-2612](https://eips.ethereum.org/EIPS/eip-2612) is final as of 2022-11-01.

#### Recommendation

Consider using the latest version of the OpenZeppelin ERC20Permit contract.

```diff
- import '@openzeppelin/contracts/token/ERC20/extensions/draft-ERC20Permit.sol';
+ import '@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol';
```

### 9. Misleading comments ![Informational]

**Context:** [contracts/JBV3Token.sol](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol)

**Description:** Multiple comments are misleading or outdated and should be updated.

**Found instances:**

- [JBV3Token.sol#L22](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L22) \
  `ERC20Permit: General token standard for fungible membership.`
- [JBV3Token.sol#L179](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L179) \
  `@param _projectId The ID of the project to which the token belongs. This is ignored.` - The `_projectId` function parameter is not ignored. Same in [L200](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L200), [L218](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L218), [L236](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L236) and [L254](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L254)
- [JBV3Token.sol#L188](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L188) \
  `// Can't transfer for a wrong project.` - It should mention _"minting"_ instead of _"transfer"_
- [JBV3Token.sol#L209](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L209) \
  `// Can't transfer for a wrong project.` - It should mention _"burn"_ instead of _"transfer"_
- [JBV3Token.sol#L227](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L227) \
  `// Can't transfer for a wrong project.` - It should mention _"approve"_ instead of _"transfer"_

#### Recommendation

Consider updating the affected comments.

### 10. Migrating `0` tokens will emit a `Transfer` event ![Informational]

**Context:** [contracts/JBV3Token.sol#L286](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L286)

**Description:** The `JBV3Token.migrate` function calls the internal `_mint` function, which emits the `Transfer` event. However, if there are no tokens to mint (i.e. `_tokensToMint = 0`), no actual minting takes place, but the event is still emitted. This can cause confusion and spams off-chain event monitoring tools.

#### Recommendation

Consider adding a check to ensure `_tokensToMint` is larger than `0`.

### 11. Event parameters are not indexed ![Informational]

**Description:** Indexed event parameters are stored in the topics part of the log instead of the data part, which allows for faster indexing/querying because of the use of bloom filters for topics. Up to three parameters in every event can be indexed. While this costs a little extra gas, doing so allows for faster and more efficient/effective event lookups.

**Found instances:**

- [JBV3TokenDeployer.sol#L26](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3TokenDeployer.sol#L26) - `event Deploy(uint256 v3ProjectId, address v3Token, address owner);`

#### Recommendation

Consider adding the `indexed` parameter, especially for address parameters where their faster lookup for security monitoring issues can be a good trade-off for the extra gas consumed.

### 12. Use `calldata` instead of `memory` for function parameters ![Informational]

**Context:** [contracts/JBV3TokenDeployer.sol#L72-L73](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3TokenDeployer.sol#L72-L73)

The `JBV3TokenDeployer.deploy` function declares the parameters `_name` and `_symbol` as `memory` instead of `calldata`. However, using `calldata` for function parameters whose value is read-only is slightly more gas efficient.

#### Recommendation

Consider using `calldata` instead of `memory` to save gas costs.

### 13. Spelling issues ![Informational]

**Found instances:**

- [JBV3Token.sol#L348](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol#L348) - `balane` - Suggestion: `balance`

# Appendix

## Appendix 1

This audit covered the following files:

| File                                                                                                                                                                | SHA-1 Hash                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| [contracts/JBV3Token.sol](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3Token.sol)                 | `a93f6186fa89e9e89cb5ae4a40bc9895a01d33e5855667dd04bd926b42a25062` |
| [contracts/JBV3TokenDeployer.sol](https://github.com/jbx-protocol/juice-v3-migration/blob/c8a068f0f12b1ad41e1b7dc97976a0dd7ae0e464/contracts/JBV3TokenDeployer.sol) | `a9b8ebcfc85b37fc65871038939d5c7aab0ae941d83813ea3519bf6e0ebf24f8` |

[critical]: https://img.shields.io/badge/-Critical-d10b0b "Critical"
[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[minor]: https://img.shields.io/badge/-Minor-yellow "Minor"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
[resolved]: https://img.shields.io/badge/-Resolved-brightgreen "Resolved"
[pending]: https://img.shields.io/badge/-Pending-lightgrey "Pending"
[ack]: https://img.shields.io/badge/-Acknowledged-grey "Acknowledged"

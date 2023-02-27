# Audit Report - Reunit Wallet Token Presale

|             |                                                                           |
| ----------- | ------------------------------------------------------------------------- |
| **Date**    | February 2023                                                             |
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

The review focused on the deployed `reunitPresale` contract [0xdca1b8732cd9847c6f422b7a733a077c6b24b631](https://etherscan.io/address/0xdca1b8732cd9847c6f422b7a733a077c6b24b631#code) on Mainnet. The code for the contract is hosted in the private GitHub repository https://github.com/GoldenNaim/audit_reunit_presale and was frozen to the commit hash [`7bbb2c100946c984fae3679be7228a0225f69929`](https://github.com/GoldenNaim/audit_reunit_presale/tree/7bbb2c100946c984fae3679be7228a0225f69929) for the audit.

The list of files in scope can be found in [Appendix 1](#Appendix-1).

# License

This audit report is licensed under a [Creative Commons - Attribution-NoDerivatives 4.0 International (CC BY-ND 4.0)](https://creativecommons.org/licenses/by-nd/4.0/) license.

# Disclaimer

_This audit report should not be taken as a guarantee that all vulnerabilities have been identified and addressed. While every effort has been made to thoroughly review the smart contracts for potential vulnerabilities, it is not possible to absolutely ensure the complete absence of vulnerabilities. The audit is a time- and resource-limited engagement. As such, it is not possible to guarantee that all vulnerabilities will be identified or that the smart contracts will be completely secure following the audit. The auditor can not be held liable for any damages or losses that may arise from using the contracts in scope._

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

The audited contracts contain **0 critical** issues, **1 high severity** issue, **0 medium** severity issues, **3 minor** issues, **2 informational** issues and **1 gas optimization recommendation**.

| #   | Title                                                                                                                                                                                     | Severity         | Status      |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ----------- |
| 1   | [Using the non-standard ERC-20 token `USDT` will cause transactions to revert unexpectedly](#1-using-the-non-standard-erc-20-token-usdt-will-cause-transactions-to-revert-unexpectedly--) | ![high]          | ![resolved] |
| 2   | [Unrecoverable native tokens due to unnecessary `receive()` function](#2-unrecoverable-native-tokens-due-to-unnecessary-receive-function--)                                               | ![minor]         | ![resolved] |
| 3   | [Potential `REUNI_TOKEN` token reentrancy issues](#3-potential-reuni_token-token-reentrancy-issues--)                                                                                     | ![minor]         | ![resolved] |
| 4   | [Anyone can claim Reuni tokens on behalf of someone else](#4-anyone-can-claim-reuni-tokens-on-behalf-of-someone-else--)                                                                   | ![minor]         | ![resolved] |
| 5   | [Permissionless contract design to reduce centralization risk](#5-permissionless-contract-design-to-reduce-centralization-risk--)                                                         | ![informational] | ![ack]      |
| 6   | [Add functionality to block already whitelisted users from participating in the presale](#6-add-functionality-to-block-already-whitelisted-users-from-participating-in-the-presale--)     | ![informational] | ![resolved] |
| 7   | [Redundant ERC-20 token spending allowance and balance checks](#7-redundant-erc-20-token-spending-allowance-and-balance-checks--)                                                         | ![gas]           | ![resolved] |

## High Risk Findings (1)

### 1. Using the non-standard ERC-20 token `USDT` will cause transactions to revert unexpectedly ![high] ![resolved]

**Context:** [presale.sol#L69](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L106)

#### Impact

Using `USDT` as a means of payment will cause the transaction to revert unexpectedly.

#### Description

Some tokens, such as `USDT` (see [L126](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code)) do not correctly implement the `EIP20` standard and their `transfer` and `transferFrom` functions return `void` instead of a success boolean. Calling these functions and expecting a `bool` return value will cause the transaction to revert due.

The issue in the `reunitPresale` contract is twofold:

1. The `manageToken` interface expects a `bool` return value from the `transferFrom` and `transfer` function and causes a revert while trying to decode the return value
2. The `buyPresale` function asserts the return value of the `transferFrom` function in line 69 to be `true`

As the `reunitPresale` contract specifies a list of certain payment tokens and `USDT` being one of them, it is important to ensure that the contract is compatible with all tokens in the list.

#### Recommendation

Consider using OpenZeppelin’s `SafeERC20` `safeTransfer()`/`safeTransferFrom` functions to handle non-standard-compliant ERC-20 tokens.

#### Resolution

Fixed in [commit - 99ad5d0a984137427bf3dbe7890542f7e9585cc9](https://github.com/GoldenNaim/audit_reunit_presale/commit/99ad5d0a984137427bf3dbe7890542f7e9585cc9).

## Minor Findings (3)

### 2. Unrecoverable native tokens due to unnecessary `receive()` function ![minor] ![resolved]

**Context:** [presale.sol#L14](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L14)

#### Impact

Anyone accidentally sending native tokens (e.g., `ETH`) to the `reunitPresale` contract will lose their funds.

#### Description

The `reunitPresale` contract contains a `receive()` function in line 14, which can be used to receive native tokens. However, the contract does not intend to use native tokens as a means of payment for `Reuni` tokens. Thus, native tokens sent to the contract will be lost and unrecoverable.

#### Recommendation

Consider removing the `receive()` function to prevent accidental loss of native tokens.

#### Resolution

Fixed in [commit - 31a6c4b84df8df2a0769e008dd900a368d361419](https://github.com/GoldenNaim/audit_reunit_presale/commit/31a6c4b84df8df2a0769e008dd900a368d361419).

### 3. Potential `REUNI_TOKEN` token reentrancy issues ![minor] ![resolved]

**Context:** [presale.sol#L69](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L106)

#### Impact

If the `REUNI_TOKEN` token contract has callback capabilities, such as `ERC-777` transfer hooks, the `claimReuni` function can be reentered to claim more `REUNI_TOKEN` tokens than the user is entitled to.

#### Description

The `claimReuni` function performs the `REUNI_TOKEN` token transfer prior to changing the state of `AMOUNT_WITHDRAWN[user]` in line 107.

```solidity
092: function claimReuni(address user) public {
093:     require(CLAIM_OPEN  == 1, "The claim period is not open");
094:     require(PARTICIPANT_LIST[user] > 0, "You have nothing to claim");
095:     uint256 AUTHORIZED_AMOUNT;
096:
097:     if(CLAIM_MULTIPLIER == 100) {
098:         AUTHORIZED_AMOUNT   =   PARTICIPANT_LIST[user]*REUNI_DECIMALS;
099:     } else {
100:         AUTHORIZED_AMOUNT   =   PARTICIPANT_LIST[user]*REUNI_DECIMALS/100*CLAIM_MULTIPLIER;
101:     }
102:
103:     require(AMOUNT_WITHDRAWN[user] < AUTHORIZED_AMOUNT, "You have reached the maximum claimable amount");
104:     uint256 AMOUNT_TO_WITHDRAW  =   AUTHORIZED_AMOUNT-AMOUNT_WITHDRAWN[user];
105:
106:     manageToken(REUNI_TOKEN).transfer(user,AMOUNT_TO_WITHDRAW);
107:     AMOUNT_WITHDRAWN[user]  +=   AMOUNT_TO_WITHDRAW; // @audit-info State change after the token transfer
108:     emit Claim(user,AMOUNT_TO_WITHDRAW);
109: }
```

Similarly, the `buyPresale` function changes the state of `PARTICIPANT_LIST[msg.sender]` and `DEPOSITS` after the token transfer. While there's no inherent risk due to using a selected list of payment tokens, it's still good practice to apply the checks-effects-interactions pattern.

```solidity
60: function buyPresale(bytes memory signature, uint256 amount, address token) public {
...     // [...]
68:
69:     if( manageToken(token).transferFrom(msg.sender,address(this),amount*ALLOWED_TOKENS[token]) ) {
70:         PARTICIPANT_LIST[msg.sender] += amount; // @audit-info State changes after the token transfer
71:         DEPOSITS += amount;
72:         emit Deposit(msg.sender, amount);
73:     } else {
74:         revert();
75:     }
76: }
```

#### Recommendation

Consider applying the checks-effects-interactions pattern by moving the token `transfer` calls in lines 69 and 106 to the end of the function after state changes have been made.

#### Resolution

Fixed in [commit - 99ad5d0a984137427bf3dbe7890542f7e9585cc9](https://github.com/GoldenNaim/audit_reunit_presale/commit/99ad5d0a984137427bf3dbe7890542f7e9585cc9).

### 4. Anyone can claim Reuni tokens on behalf of someone else ![minor] ![resolved]

**Context:** [presale.sol#L92](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L92)

#### Impact

Anyone can claim `Reuni` tokens on behalf of someone else and cause issues with smart contracts that rely on claiming the rewards themselves.

#### Description

The `claimReuni` function allows claiming the `Reuni` tokens for a given `user` address.

While the tokens are sent to the correct address, `user`, this can lead to issues with smart contracts that might rely on claiming the rewards themselves.

```solidity
092: function claimReuni(address user) public {
093:     require(CLAIM_OPEN  == 1, "The claim period is not open");
094:     require(PARTICIPANT_LIST[user] > 0, "You have nothing to claim");
095:     uint256 AUTHORIZED_AMOUNT;
096:
097:     if(CLAIM_MULTIPLIER == 100) {
098:         AUTHORIZED_AMOUNT   =   PARTICIPANT_LIST[user]*REUNI_DECIMALS;
099:     } else {
100:         AUTHORIZED_AMOUNT   =   PARTICIPANT_LIST[user]*REUNI_DECIMALS/100*CLAIM_MULTIPLIER
101:     }
102:
103:     require(AMOUNT_WITHDRAWN[user] < AUTHORIZED_AMOUNT, "You have reached the maximum claimable amount");
104:     uint256 AMOUNT_TO_WITHDRAW  =   AUTHORIZED_AMOUNT-AMOUNT_WITHDRAWN[user];
105:
106:     manageToken(REUNI_TOKEN).transfer(user,AMOUNT_TO_WITHDRAW);
107:     AMOUNT_WITHDRAWN[user]  +=   AMOUNT_TO_WITHDRAW;
108:     emit Claim(user,AMOUNT_TO_WITHDRAW);
109: }
```

#### Recommendation

Consider using `msg.sender` instead of the `address user` parameter to prevent anyone from claiming on behalf of someone else.

#### Resolution

Fixed in [commit - 96535138d10b91a8cdae453c31b08548cca4629e](https://github.com/GoldenNaim/audit_reunit_presale/commit/96535138d10b91a8cdae453c31b08548cca4629e).

## Informational Findings (2)

### 5. Permissionless contract design to reduce centralization risk ![informational] ![ack]

**Context:** [presale.sol](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L12)

#### Impact

The current contract design grants the contract owner full control over the presale and claim period and the ability to change critical parameters at any time.

#### Description

Allow-listed users deposit funds in the `reunitPresale` as part of the presale period to be able to claim the corresponding amount of `Reunit` tokens during the claim period.

To ensure high trustworthiness of the contract, critical parameters to configure the presale and claim periods should be immutable and initialized at the deployment. Additionally, the presale and claim periods should be pre-defined upfront to establish trust and a clear timeline for the sale.

#### Recommendation

Consider adopting a more permissionless contract design for the `reunitPresale` contract.

### 6. Add functionality to block already whitelisted users from participating in the presale ![informational] ![resolved]

**Context:** [presale.sol#L45](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L45)

#### Impact

A misbehaving allow-listed actor can not be blocked from the presale.

#### Description

Only allow-listed users can participate in the presale by providing a signed message by the `REUNIT_DAO` address containing the user's address. The provided signature is then verified by the `isWhitelisted()` function.

However, once a user has retrieved the required signature, it is not possible to revoke their presale access. This could lead to a situation where a previously good actor turns into a misbehaving actor without the ability to block them from the presale.

```solidity
45: function isWhitelisted(bytes memory signature, address user) public view returns (bool) {
46:     bytes32 messageHash     =   keccak256(abi.encodePacked(user));
47:     return REUNIT_DAO       ==  messageHash.toEthSignedMessageHash().recover(signature);
48: }
```

#### Recommendation

Consider adding a block-list functionality to prevent bad actors from participating in the presale.

#### Resolution

Fixed in [commit - 283c6ac2d2eb3461b74c1920d64576c47aad8089](https://github.com/GoldenNaim/audit_reunit_presale/commit/283c6ac2d2eb3461b74c1920d64576c47aad8089).

## Gas Optimizations (1)

### 7. Redundant ERC-20 token spending allowance and balance checks ![gas] ![resolved]

**Context:** [presale.sol#L66-L67](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol#L66-L67)

#### Impact

The ERC-20 token spending allowance and balance checks are redundant and can be safely removed to save gas as the transfer function already performs them.

#### Description

The `buyPresale()` function checks the ERC-20 token spending allowance and balance of the caller prior to transferring the tokens to the contract. However, the `transferFrom()` function already performs these checks and will revert if the allowance or balance is insufficient. Therefore, the redundant checks can be safely removed to save gas.

```solidity
60: function buyPresale(bytes memory signature, uint256 amount, address token) public {
...     // [...]
66:     require(manageToken(token).allowance(msg.sender,address(this)) >= amount*ALLOWED_TOKENS[token], "Please authorize this token to be deposited in this contract");
67:     require(manageToken(token).balanceOf(msg.sender) >= amount*ALLOWED_TOKENS[token], "It looks like you don't have the required balance");
...     // [...]
76: }
```

#### Recommendation

Consider removing the redundant ERC-20 token spending allowance and balance checks to save gas.

#### Resolution

Fixed in [commit - 3512ada1b536d9eed1e5e74e3ebfe61ecaef8fac](https://github.com/GoldenNaim/audit_reunit_presale/commit/3512ada1b536d9eed1e5e74e3ebfe61ecaef8fac).

# Appendix

## Appendix 1

This audit covered the following files:

| File                                                                                                                        | SHA-1 Hash                                                         |
| --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| [presale.sol](https://github.com/GoldenNaim/audit_reunit_presale/blob/7bbb2c100946c984fae3679be7228a0225f69929/presale.sol) | `c8c416a47f69256a294032863ef3e6fb80e7ae783adfd9aa34015f52d74dfadf` |

[critical]: https://img.shields.io/badge/-Critical-d10b0b "Critical"
[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[minor]: https://img.shields.io/badge/-Minor-yellow "Minor"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
[gas]: https://img.shields.io/badge/-Gas-9cf "Gas"
[resolved]: https://img.shields.io/badge/-Resolved-brightgreen "Resolved"
[pending]: https://img.shields.io/badge/-Pending-lightgrey "Pending"
[ack]: https://img.shields.io/badge/-Acknowledged-grey "Acknowledged"

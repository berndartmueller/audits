# Audit Report - Bazaar

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

The review focused on the commit hash [`0c4c7691931c2dc6a41ea7da6b3b8edf852437bc`](https://github.com/bazaar-buidlers/contracts/tree/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc) of the public https://github.com/bazaar-buidlers/contracts GitHub repository.

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

The audited contracts contain **0 critical** issues, **0 high severity** issues, **5 medium** severity issues, **2 minor** issues, **2 informational** issues and **2 gas optimization recommendations**.

| #   | Title                                                                                                                                                                   | Severity         | Status      |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ----------- |
| 1   | [The owner of the `Bazaar` contract can withdraw escrowed funds](#1-the-owner-of-the-bazaar-contract-can-withdraw-escrowed-funds--)                                     | ![medium]        | ![resolved] |
| 2   | [Tokens not compliant with ERC-20 will cause transactions to revert unexpectedly](#2-tokens-not-compliant-with-erc-20-will-cause-transactions-to-revert-unexpectedly--) | ![medium]        | ![resolved] |
| 3   | [Fee-less token mints caused by low-price tokens can possibly revert](#3-fee-less-token-mints-caused-by-low-price-tokens-can-possibly-revert--)                         | ![medium]        | ![resolved] |
| 4   | [The uniqueness configuration of a listing can be arbitrarily changed](#4-the-uniqueness-configuration-of-a-listing-can-be-arbitrarily-changed--)                       | ![medium]        | ![resolved] |
| 5   | [Anyone can mint an arbitrary amount of free listing tokens](#5-anyone-can-mint-an-arbitrary-amount-of-free-listing-tokens--)                                           | ![medium]        | ![resolved] |
| 6   | [Inconsistent escrow balance when supplying rebase or fee-on-transfer tokens](#6-inconsistent-escrow-balance-when-supplying-rebase-or-fee-on-transfer-tokens--)         | ![minor]         | ![resolved] |
| 7   | [Overpaying with native tokens for minting tokens](#7-overpaying-with-native-tokens-for-minting-tokens--)                                                               | ![minor]         | ![resolved] |
| 8   | [Use of floating pragma](#8-use-of-floating-pragma--)                                                                                                                   | ![informational] | ![resolved] |
| 9   | [Miscellaneous](#9-miscellaneous--)                                                                                                                                     | ![informational] | ![resolved] |
| 10  | [Use custom errors instead of error strings](#10-use-custom-errors-instead-of-error-strings-)                                                                           | ![gas]           |             |
| 11  | [Pack structs by putting variables that can fit together next to each other](#11-pack-structs-by-putting-variables-that-can-fit-together-next-to-each-other--)          | ![gas]           | ![resolved] |

# Findings

## Medium Risk Findings (5)

### 1. The owner of the `Bazaar` contract can withdraw escrowed funds ![medium] ![resolved]

**Context:** [Escrow.sol#L44](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol#L44)

#### Impact

The current owner of the `Bazaar` contract can add functionality to withdraw escrowed funds from the `Escrow` contract by upgrading the `Bazaar` contract.

#### Description

The `Escrow.withdraw` function is access protected and only callable by the contract owner. In this specific case, the `Bazaar` contract gets the ownership transferred as part of the deployment process.

The `Escrow.withdraw` function enables the caller to withdraw escrowed funds from the provided `from` address. This imposes a significant centralization risk on the escrowed funds. In the event that the owner of the `Bazaar` contract turns malicious, the contract can be upgraded to add additional functionality allowing the owner to withdraw escrowed funds from any address.

```solidity
44: function withdraw(address from, address payable to, address erc20) external onlyOwner {
45:     uint256 amount = _deposits[from][erc20];
46:     require(amount > 0, "nothing to withdraw");
47:
48:     _deposits[from][erc20] = 0;
49:     emit Withdrawn(from, erc20, amount);
50:
51:     // zero address is native tokens
52:     if (erc20 == address(0)) {
53:         to.sendValue(amount);
54:     } else {
55:         require(IERC20(erc20).transfer(to, amount), "transfer failed");
56:     }
57: }
```

#### Recommendation

Consider using `_msgSender()` instead of the `address from` parameter in the `Escrow.withdraw` function and remove the `onlyOwner` modifier from the `Escrow.depositsOf` view function as well.

#### Resolution

Fixed in [Audit: Escrow Withdraw #1](https://github.com/bazaar-buidlers/contracts/pull/1).

### 2. Tokens not compliant with ERC-20 will cause transactions to revert unexpectedly ![medium] ![resolved]

**Context:** [Escrow.sol#L35](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol#L35), [Escrow.sol#L55](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol#L55)

#### Impact

The protocol does not work with certain ERC-20 tokens.

#### Description

Some tokens (like `USDT`, see [L126](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code)) do not correctly implement the `EIP20` standard and their `transfer` and `transferFrom` functions return `void` instead of a success boolean. Calling these functions and asserting the return value will always revert.

As the `Bazaar` protocol allows using arbitrary ERC-20 tokens as a means of payment, using non-standard compliant tokens will render the protocol unusable as the return value of `transfer`/`transferFrom` is checked.

#### Recommendation

Consider using OpenZeppelin’s `SafeERC20` `safeTransfer()`/`safeTransferFrom` functions that handle the return value check as well as non-standard-compliant tokens.

#### Resolution

Fixed in [Audit: Safe ERC20 #2](https://github.com/bazaar-buidlers/contracts/pull/2).

### 3. Fee-less token mints caused by low-price tokens can possibly revert ![medium] ![resolved]

**Context:** [Bazaar.sol#L117](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol#L26)

#### Impact

If the `fee` for a given mint is zero, minting will revert.

#### Description

Due to the assertion in `Escrow.deposit` in line 26, deposits with a zero amount will revert.

Minting tokens are subject to a protocol fee, `feeNumerator`. If the price of a token is low (e.g., by using a low-decimal ERC-20 token - see [Weird ERC20 Tokens](https://github.com/d-xo/weird-erc20#low-decimals)) and the `feeNumerator` is set to a small value, the `fee` calculation could result in `0`.

If the `fee` is equal to `0`, the `escrow.deposit` of the fee for the protocol owner will revert due to zero-value deposits being disabled.

[Bazaar.sol#L112-L122](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Bazaar.sol#L112-L122)

```solidity
094: function mint(address to, uint256 id, uint256 amount, address erc20, bytes32[] calldata proof) external payable {
...      // [...]
111:
112:     // fee goes to owner and remainder goes to vendor
113:     uint256 fee = (price * feeNumerator) / feeDenominator;
114:     if (erc20 == address(0)) {
115:         // native token deposit
116:         escrow.deposit{ value: price - fee }(sender, listing.vendor, erc20, price - fee);
117:         escrow.deposit{ value: fee }(sender, owner, erc20, fee);
118:     } else {
119:         // erc20 token deposit
120:         escrow.deposit(sender, listing.vendor, erc20, price - fee);
121:         escrow.deposit(sender, owner, erc20, fee);
122:     }
123: }
```

[Escrow.sol#L26](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol#L26)

```solidity
25: function deposit(address from, address to, address erc20, uint256 amount) external payable onlyOwner {
26:     require(amount > 0, "nothing to deposit");
27:
28:     _deposits[to][erc20] += amount;
29:     emit Deposited(to, erc20, amount);
30:
31:     // zero address is native tokens
32:     if (erc20 == address(0)) {
33:         require(msg.value == amount, "value must equal amount");
34:     } else {
35:         require(IERC20(erc20).transferFrom(from, address(this), amount), "transfer failed");
36:     }
37: }
```

#### Recommendation

Consider removing the `require(amount > 0, "nothing to deposit");` check in line 26 in the `Escrow.deposit` function.

#### Resolution

Fixed in [Audit: Fee-less Mint #3](https://github.com/bazaar-buidlers/contracts/pull/3).

### 4. The uniqueness configuration of a listing can be arbitrarily changed ![medium] ![resolved]

**Context:** [Bazaar.sol#L155](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Bazaar.sol#L155)

### Impact

A vendor could trick buyers into thinking the listing is unique when it is not.

#### Description

Suppose a **non-unique** listing is added via the `Bazaar.list` function and later on, the configuration of the listing is changed to **unique** by the vendor. In that case, it's not guaranteed that the minted tokens so far are unique per address.

```solidity
149: function configure(uint256 id, uint256 config, uint256 limit, uint256 allow, uint96 royalty) external onlyVendor(id) {
150:     Listings.Listing storage listing = _listings[id];
151:
152:     require(royalty <= feeDenominator, "royalty will exceed sale price");
153:     require(limit == 0 || limit >= listing.supply, "limit lower than supply");
154:
155:     listing.config = config; // @audit-info The listing configuration can be changed from non-unique to unique
156:     listing.limit = limit;
157:     listing.allow = allow;
158:     listing.royalty = royalty;
159:
160:     emit Configure(config, limit, allow, royalty, id);
161: }
```

#### Recommendation

Consider adding a check to the `Bazaar.configure` function to prevent changing the configuration of a listing from non-unique to unique.

```diff
diff --git a/contracts/Bazaar.sol b/contracts/Bazaar.sol
index 8be2f4f..cc051a3 100644
--- a/contracts/Bazaar.sol
+++ b/contracts/Bazaar.sol
@@ -152,11 +152,15 @@ contract Bazaar is Initializable, OwnableUpgradeable, ERC1155Upgradeable, IERC29
require(royalty <= feeDenominator, "royalty will exceed sale price");
require(limit == 0 || limit >= listing.supply, "limit lower than supply");

-        listing.config = config;
+        bool _isUnique = listing.isUnique();
+
+        listing.config = config;
listing.limit = limit;
listing.allow = allow;
listing.royalty = royalty;

+        require(_isUnique == listing.isUnique() || listing.supply == 0, "cannot change the unique configuration");
+
emit Configure(config, limit, allow, royalty, id);
}
```

#### Resolution

Fixed in [Audit: Lock Config #4](https://github.com/bazaar-buidlers/contracts/pull/4).

### 5. Anyone can mint an arbitrary amount of free listing tokens ![medium] ![resolved]

**Context:** [Bazaar.sol#L105](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Bazaar.sol#L105)

#### Impact

A bad actor is able to mint `type(uint256).max` free tokens to prevent other mints by other users.

#### Description

Anyone can mint an arbitrary amount of free listing tokens, given the listing is configured to be non-unique. The maximum possible amount of tokens to mint is limited by the optional `Listing.limit` limit.

This allows a bad actor to mint up to a maximum of `type(uint256).max` free tokens, which prevents other mints by other users due to `listing.supply += amount;` in line 256 reverting with an overflow error.

```solidity
094: function mint(address to, uint256 id, uint256 amount, address erc20, bytes32[] calldata proof) external payable {
095:     Listings.Listing storage listing = _listings[id];
096:     require(!listing.isPaused(), "minting is paused");
097:
098:     address owner = owner();
099:     address sender = _msgSender();
100:
101:     if (listing.allow != 0) {
102:         require(listing.isAllowed(sender, proof), "not allowed");
103:     }
104:     if (listing.isFree()) {
105:         return _mint(to, id, amount, ""); // @audit-info`amount` can be arbitrarily large
106:     }
107:
...:     [...]
123: }
```

#### Recommendation

Consider removing the `amount` parameter of the `Bazaar.mint` function to limit minting more than a single token at a time.

#### Resolution

Fixed in [Audit: Mint Amount #5](https://github.com/bazaar-buidlers/contracts/pull/5).

## Minor Risk Findings (2)

### 6. Inconsistent escrow balance when supplying rebase or fee-on-transfer tokens ![minor] ![resolved]

**Context:** [Escrow.sol#L28](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol#L28)

#### Impact

Escrowed fund balance bookkeeping is inaccurate and may account for more funds than actually available.

#### Description

ERC-20 tokens may make specific customizations to their ERC-20 contracts.
One type of these tokens is FoT (fee-on-transfer) tokens that charge a specific fee for every `transfer()` or `transferFrom()` token transfer.

In the current protocol implementation, the `Escrow` contract assumes that the received amount in the `deposit` function is the same as the transfer amount and uses it for internal balance bookkeeping.

However, if the token is a fee-on transfer token, the received amount can differ and will be less than the initial transfer amount.

#### Recommendation

Determine the received amount by calculating the difference between the balance before and after the transfer.

#### Resolution

Fixed in [Audit: QA fee-on-transfer #6](https://github.com/bazaar-buidlers/contracts/pull/6).

### 7. Overpaying with native tokens for minting tokens ![minor] ![resolved]

**Context:** [Bazaar.sol#L116](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Bazaar.sol#L116)

#### Impact

Users can overpay for a token mint without getting the overpaid amount of native tokens refunded.

#### Description

A user minting a listed token for a specified price can accidentally send more native tokens than needed without the possibility of getting the overpaid amount refunded.

#### Recommendation

Consider adding a check to prevent sending more native tokens than required for the mint:

```solidity
require(msg.value == price, "incorrect amount of native tokens sent");
```

Additionally, consider reverting a free token mint if `msg.value` is non-zero in line 105.

#### Resolution

Fixed in [Audit: QA overpayment #7](https://github.com/bazaar-buidlers/contracts/pull/7).

## Informational Findings (2)

### 8. Use of floating pragma ![informational] ![resolved]

It is best to deploy contracts with the same compiler version and flags that they have been tested with. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

https://swcregistry.io/docs/SWC-103

Used compiler version: `^0.8.9`

#### Recommendation

We recommend locking the pragma version in the audited contracts and also consider known bugs (https://github.com/ethereum/solidity/releases) for the compiler version that is chosen.

#### Resolution

Fixed in [Audit: QA Solidity Version #8](https://github.com/bazaar-buidlers/contracts/pull/8).

### 9. Miscellaneous ![informational] ![resolved]

#### [MISC-01] Constant variable names should be UPPER_CASE

- `Bazaar.sol#L28` - `feeDenominator` - Suggestion: `FEE_DENOMINATOR`

Fixed in [Audit: QA Constant Name #9](https://github.com/bazaar-buidlers/contracts/pull/9).

#### [MISC-02] Shadowing of state variables

Consider renaming the found instances:

- `Bazaar.sol#L98` - `address owner = owner();` - Suggestion: `address _owner = owner();`

Fixed in [Audit: QA Shadow Variable #10](https://github.com/bazaar-buidlers/contracts/pull/10).

## Gas Optimizations (2)

### 10. Use custom errors instead of error strings ![gas]

Instead of using error strings, you should use custom errors to reduce deployment and runtime costs. This would save both deployment and runtime costs.

_Instances (17)_

[Bazaar.sol](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Bazaar.sol)

```
47:         require(_feeNumerator <= feeDenominator, "invalid protocol fee");
64:         require(royalty <= feeDenominator, "royalty will exceed sale price");
96:         require(!listing.isPaused(), "minting is paused");
102:        require(listing.isAllowed(sender, proof), "not allowed");
109:        require(price > 0, "invalid currency or amount");
131:        require(erc20s.length == prices.length, "mismatched erc20 and price");
152:        require(royalty <= feeDenominator, "royalty will exceed sale price");
153:        require(limit == 0 || limit >= listing.supply, "limit lower than supply");
223:        require(_listings[id].vendor == _msgSender(), "sender is not vendor");
260:        require(balanceOf(to, id) + amount == 1, "token is unique");
264:        require(from == address(0), "token is soulbound");
268:        require(listing.supply <= listing.limit, "token limit reached");
```

[Escrow.sol](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol)

```
26:         require(amount > 0, "nothing to deposit");
33:         require(msg.value == amount, "value must equal amount");
35:         require(IERC20(erc20).transferFrom(from, address(this), amount), "transfer failed");
46:         require(amount > 0, "nothing to withdraw");
55:         require(IERC20(erc20).transfer(to, amount), "transfer failed");
```

### 11. Pack structs by putting variables that can fit together next to each other ![gas] ![resolved]

As the EVM works with 32 bytes, variables less than 32 bytes should be packed inside a struct so that they can be stored in the same slot, this saves gas when writing to storage.

In [`Listings.sol#L18`](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Listings.sol#L18), by moving the `vendor` variable next to the `royalty` variable, 1 storage slot can be saved.

```diff
diff --git a/contracts/Listings.sol b/contracts/Listings.sol
index 611f0d3..88ac6da 100644
--- a/contracts/Listings.sol
+++ b/contracts/Listings.sol
@@ -14,8 +14,6 @@ library Listings {
uint256 constant CONFIG_UNIQUE = 1 << 3;

struct Listing {
-        // vendor address
-        address vendor;
// configuration mask
uint256 config;
// total number of mints
@@ -24,6 +22,8 @@ library Listings {
uint256 limit;
// allow merkle tree root
uint256 allow;
+        // vendor address
+        address vendor;
// royalty fee basis points
uint96 royalty;
// metadata uri
```

#### Resolution

Fixed in [Audit: QA Pack Struct #11](https://github.com/bazaar-buidlers/contracts/pull/11).

# Appendix

## Appendix 1

This audit covered the following files:

| File                                                                                                                                        | SHA-1 Hash                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| [contracts/Bazaar.sol](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Bazaar.sol)     | `b30c15f81c3d2bd7054fded7a645d2be8b161ef7bd606f52d597db1e3dd0930e` |
| [contracts/Escrow.sol](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Escrow.sol)     | `df92f6cee4fb59b0d719da07738545804ffd45404716fa0edc49f50c33facc28` |
| [contracts/Listings.sol](https://github.com/bazaar-buidlers/contracts/blob/0c4c7691931c2dc6a41ea7da6b3b8edf852437bc/contracts/Listings.sol) | `273e32fb4e2faf018116da967dfe902325d7b13018a6e5100c977fc8c345b650` |

[critical]: https://img.shields.io/badge/-Critical-d10b0b "Critical"
[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[minor]: https://img.shields.io/badge/-Minor-yellow "Minor"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
[gas]: https://img.shields.io/badge/-Gas-9cf "Gas"
[resolved]: https://img.shields.io/badge/-Resolved-brightgreen "Resolved"
[pending]: https://img.shields.io/badge/-Pending-lightgrey "Pending"
[ack]: https://img.shields.io/badge/-Acknowledged-grey "Acknowledged"

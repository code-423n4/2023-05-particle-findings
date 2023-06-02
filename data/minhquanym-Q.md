# Summary

| Id | Title |
| --- | --- |
| L-1 | Function repayWithNft() should not be called when the auction is concluded |
| L-2 | Lender can call stopLoanAuction() for a concluded auction |
| L-3 | Attacker can create spam lien without transferring any ERC721 token |
| L-4 | Use of floating pragma |
| NC-1 | Lack of Natspec comments |
| NC-2 | Avoidance of magic values |

# L-1. Function `repayWithNft()` should not be called when the auction is concluded

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L466

## Detail

A concluded auction allows the lender to liquidate the loan and take the ETH back. Thus, when the auction is concluded, the borrower should not be allowed to call `repayWithNft()`.

## Recommendation

Consider disabling `repayWithNft()` if the auction is concluded for that loan.

# L-2. Lender can call `stopLoanAuction()` for a concluded auction

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L663

## Detail

The lender can still call `stopLoanAuction()` for a concluded auction, while it should only be called for a live auction.

```
// @done-audit can stop a concluded auction
if (lien.auctionStartTime == 0) {
    revert Errors.AuctionNotStarted();
}

```

## Recommendation

Consider disabling `stopLoanAuction()` for concluded auctions.

# L-3. Anyone can create spam lien without transferring any ERC721 token

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L90

## Detail

The function `onERC721Received()` accepts any sender, even EOA, to call it. Malicious users might spam call this function to create spam lien without transferring any ERC721 token. However, it will only affect the UI/UX of the protocol and not the smart contract if users do not interact with these malicious liens.

## Recommendation

Consider disabling EOA to call `onERC721Received()`. It will not completely fix the issue since the contract can still be spammed, but it at least limits the possibility that users might mistakenly call this function.

# L-4. Use of Floating Pragma

## Occurrences

The floating pragma is used in the following locations:

- https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/types/Structs.sol#L2
- https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L2
- https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L2

## Detail

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

```

## Recommendation

Consider using a fixed solidity version.

# NC-1. Lacking Natspec comments

The following functions lack Natspec comments for their parameters:

- https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L58
- https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L790-L818

For example:

```solidity
// @audit Missing bips in Natspec comment
/**
 * @dev Converts an integer bips value to a signed wad value.
 */
function bipsToWads(uint256 bips) public pure returns (uint256 wad) {
    return (bips * 1e18) / _BASIS_POINTS;
}

```

## Recommendation

Consider completing the missing Natspec comments.

# NC-2. Avoid magic values

## Detail

The use of magic values should be avoided as they are easy to make mistakes with.

```solidity
function calculateCurrentInterest(
    uint256 principal,
    uint256 rateBips,
    uint256 loanStartTime
) external view returns (uint256 interest) {
    uint256 loanTimeWad = (block.timestamp - loanStartTime) * 1e18; // @audit avoid magic value (1e18)
    interest = principal.wadMul(bipsToWads(rateBips).wadMul(loanTimeWad.wadDiv(_YEAR_WAD)));
    return interest;
}

```

## Recommendation

Create constants for these values and use the constants instead of magic values.

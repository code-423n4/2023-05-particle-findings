
## Simplify math
In [bipsToWads()`](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L60-L62)
```diff
- return (bips * 1e18) / _BASIS_POINTS;
+ return bips * 1e14;
```

## Use WETH9 to save deployment gas
Consider using the already deployed and accepted WETH9 at 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 to save the gas from deploying your own.

## `safeTransferFrom()` can be `transferFrom`
In `supplyNft()` the NFT is [transferred to self](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L57) which we know can receive an ERC721, so there is no need to use safe transfer, which wastes gas by also calling `onERC721Received()`.
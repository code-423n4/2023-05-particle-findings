L-01:_execSellNftToMarket() IERC721.ownerOf() method will revert after NFT has burn(), resulting in sell failure

`_execSellNftToMarket()` uses `IERC721(collection).ownerOf(tokenId) == address(this)` at the end of the method to determine if it is `InvalidNFTSell`
But in some extreme cases, for example, people who buy NFTs with the intention of burning them, but use ownerOf(), resulting in a revert
```solidity
    function ownerOf(uint256 id) public view virtual returns (address owner) {
        require((owner = _ownerOf[id]) ! = address(0), "NOT_MINTED").
    }
```

`_execSellNftToMarket ()` purpose is to sell NFT, even if this sell NFT was burned, should also be successful, should not be NFT burn restrictions, resulting in the inability to sell

It is recommended to use try/catch, if it has been burn, considered a valid `sell`


L-02:_withdrawAccountInterest() may not be able to use `transfer()` to receive funds if lender is a contract
It is recommended to add the entry `receiver`


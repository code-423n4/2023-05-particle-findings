## QA REPORT

| |Issue|
|-|:-|
| [01] | LIEN'S LENDER CAN LOSE NFT AND ASSOCIATED NFT PRICE AND POTENTIAL INTEREST WHEN `ParticleExchange.sellNftToMarket` FUNCTION IS MALICIOUSLY CALLED WITH `pushBased` BEING TRUE AND `marketplace` BEING A MARKETPLACE THAT DOES NOT SUPPORT "PUSH BASED" METHOD BUT IMPLEMENTS `onERC721Received` FUNCTION FOR OTHER PURPOSES |
| [02] | CALLING `transfer` FUNCTION TO SEND ETH CAN REVERT |
| [03] | LENDER, WHO USES "PUSH BASED" METHOD, CAN LOSE NFT TO `ParticleExchange` CONTRACT IF `data.length` IS NOT 64 WHEN CALLING `ParticleExchange.onERC721Received` FUNCTION |
| [04] | `ParticleExchange.refinanceLoan` FUNCTION DOES NOT PREVENT BORROWER FROM REFINANCING TO A LESS COMPETITIVE OFFERING |
| [05] | `_treasuryRate` CAN BE SET TO `MathUtils._BASIS_POINTS`, WHICH FORCES LENDERS TO EARN NO INTERESTS |
| [06] | `ParticleExchange.receive` FUNCTION ALLOWS USERS TO ACCIDENTALLY SEND ETH TO `ParticleExchange` CONTRACT |
| [07] | MISSING `address(0)` CHECK FOR `wethAddress` IN `ParticleExchange` CONTRACT'S CONSTRUCTOR |
| [08] | COMMENT FOR `interest` IN `MathUtils.calculateTreasuryProportion` FUNCTION IS INCORRECT |
| [09] | REDUNDANT NAMED RETURNS CAN BE REMOVED |
| [10] | REPEATED CODE CAN BE REFACTORED INTO A MODIFIER TO BE USED IN CORRESPONDING FUNCTIONS |
| [11] | CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS |
| [12] | WORD TYPING TYPOS |
| [13] | FLOATING PRAGMAS |
| [14] | INCOMPLETE NATSPEC COMMENTS |
| [15] | MISSING NATSPEC COMMENTS |

## [01] LIEN'S LENDER CAN LOSE NFT AND ASSOCIATED NFT PRICE AND POTENTIAL INTEREST WHEN `ParticleExchange.sellNftToMarket` FUNCTION IS MALICIOUSLY CALLED WITH `pushBased` BEING TRUE AND `marketplace` BEING A MARKETPLACE THAT DOES NOT SUPPORT "PUSH BASED" METHOD BUT IMPLEMENTS `onERC721Received` FUNCTION FOR OTHER PURPOSES
When calling the following `ParticleExchange.sellNftToMarket` function, a malicious actor can input `amount` as a value that equals `lien.price` without sending any `msg.value`. The malicious actor can also input `pushBased` to be true and `marketplace` to be a marketplace that does not support the "push based" method.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L253-L289
```solidity
    function sellNftToMarket(
        Lien calldata lien,
        uint256 lienId,
        uint256 amount,
        bool pushBased,
        address marketplace,
        bytes calldata tradeData
    ) external payable override validateLien(lien, lienId) {
        ...

        /// @dev cannot instantly become insolvent (will revert if credit < 0)
        uint256 credit = amount + msg.value - lien.price;

        // update lien
        liens[lienId] = keccak256(
            abi.encode(
                Lien({
                    lender: lien.lender,
                    borrower: msg.sender,
                    collection: lien.collection,
                    tokenId: lien.tokenId,
                    price: lien.price,
                    rate: lien.rate,
                    loanStartTime: block.timestamp,
                    credit: credit,
                    auctionStartTime: 0
                })
            )
        );

        // route trade execution to marketplace
        _execSellNftToMarket(lien.collection, lien.tokenId, amount, pushBased, marketplace, tradeData);

        ...
    }
```

Then, when the following `ParticleExchange._execSellNftToMarket` function is called with `pushBased` being true and `marketplace` being a marketplace that does not support the "push based" method, if such `marketplace` implements the `onERC721Received` function for other purposes, the lien's NFT would be sent to such `marketplace` but no payment would be sent back to the `ParticleExchange` contract. As a result, the lien's lender loses such NFT and cannot receive the specified NFT price or any potential interest in return. As a mitigation, please consider adding another state variable for the trusted admin to record the marketplaces that support the "push based" method and updating the `ParticleExchange._execSellNftToMarket` function to revert if `pushBased` is true but `marketplace` is not one of these marketplaces that support the "push based" method.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L291-L335
```solidity
    function _execSellNftToMarket(
        address collection,
        uint256 tokenId,
        uint256 amount,
        bool pushBased,
        address marketplace,
        bytes calldata tradeData
    ) internal {
        if (!registeredMarketplaces[marketplace]) {
            revert Errors.UnregisteredMarketplace();
        }

        ...

        if (pushBased) {
            /// @dev directly send NFT to a marketplace router (e.g., Reservoir's), based on
            /// the data, the router will match order and transfer back the correct amount of fund
            IERC721(collection).safeTransferFrom(address(this), marketplace, tokenId, tradeData);
        } else {
            ...
        }

        ...
    }
```

## [02] CALLING `transfer` FUNCTION TO SEND ETH CAN REVERT
Calling the following `ParticleExchange.withdrawEthWithInterest`, `ParticleExchange._withdrawAccountInterest`, `ParticleExchange.buyNftFromMarket`, `ParticleExchange.repayWithNft`, `ParticleExchange.auctionBuyNft`, and `ParticleExchange.withdrawTreasury` functions further call the `transfer` function to send ETH to the respective recipient. If such recipient is a contract, which has gas-intensive logics in its `receive` function, the `transfer` function call can revert because the fixed 2300 gas stipend that is forwarded is not enough for running such `receive` function. To ensure that the relevant ETH amounts can be sent successfully, please consider updating these functions to use the low-level `call` function for transferring ETH to the respective recipient and check the returned success status for determining whether these function calls should revert or not.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L192-L223
```solidity
    function withdrawEthWithInterest(Lien calldata lien, uint256 lienId) external override validateLien(lien, lienId) {
        ...

        // transfer ETH with interest back to lender
        payable(lien.lender).transfer(lien.price + payableInterest);

        // transfer PnL to borrower
        if (lien.credit > payableInterest) {
            payable(lien.borrower).transfer(lien.credit - payableInterest);
        }

        ...
    }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L231-L246
```solidity
    function _withdrawAccountInterest(address payable lender) internal {
        ...

        lender.transfer(interest);

        ...
    }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L338-L393
```solidity
    function buyNftFromMarket(
        Lien calldata lien,
        uint256 lienId,
        uint256 tokenId,
        uint256 amount,
        uint256 useToken,
        address marketplace,
        bytes calldata tradeData
    ) external payable override validateLien(lien, lienId) {
        ...

        // payback PnL to borrower
        if (payback > 0) {
            payable(lien.borrower).transfer(payback);
        }

        ...
    }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L466-L515
```solidity
    function repayWithNft(
        Lien calldata lien,
        uint256 lienId,
        uint256 tokenId
    ) external override validateLien(lien, lienId) {
        ...
        if (payback > 0) {
            payable(lien.borrower).transfer(payback);
        }

        ...
    }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L688-L748
```solidity
    function auctionBuyNft(
        Lien calldata lien,
        uint256 lienId,
        uint256 tokenId,
        uint256 amount
    ) external override validateLien(lien, lienId) auctionLive(lien) {
        ...

        // pay auction price to new NFT supplier
        if (amount > 0) {
            payable(msg.sender).transfer(amount);
        }

        ...
        if (payback > 0) {
            payable(lien.borrower).transfer(payback);
        }

        ...
    }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L808-L818
```solidity
    function withdrawTreasury(address receiver) external onlyOwner {
        if (_treasury > 0) {
            ...
            payable(receiver).transfer(withdrawAmount);
            ...
        }
    }
```

## [03] LENDER, WHO USES "PUSH BASED" METHOD, CAN LOSE NFT TO `ParticleExchange` CONTRACT IF `data.length` IS NOT 64 WHEN CALLING `ParticleExchange.onERC721Received` FUNCTION
When the lender uses the "push based" method to supply an NFT, it can accidentally input `data` with a length that is not 64 when calling the NFT's `safeTransferFrom` function. When this happens, calling the following `ParticleExchange.onERC721Received` function would not supply the NFT and also does not revert. As a result, the lender loses the NFT to the `ParticleExchange` contract and cannot get back the specified NFT price or earn any interest because no lien is created for such NFT. To avoid this from happening, please consider updating the `ParticleExchange.onERC721Received` function to revert if `data.length` is not 64 and `operator` is not a registered marketplace.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L74-L93
```solidity
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        if (data.length == 64) {
            if (registeredMarketplaces[operator]) {
                /// @dev transfer coming from registeredMarketplaces will go through buyNftFromMarket, where the NFT
                /// is matched with an existing lien (realize PnL) already. If procceds here, this NFT will be tied
                /// with two liens, which creates divergence.
                revert Errors.Unauthorized();
            }
            /// @dev MAX_PRICE and MAX_RATE should each be way below bytes32
            (uint256 price, uint256 rate) = abi.decode(data, (uint256, uint256));
            /// @dev the msg sender is the NFT collection (called by safeTransferFrom's _checkOnERC721Received check)
            _supplyNft(from, msg.sender, tokenId, price, rate);
        }
        return this.onERC721Received.selector;
    }
```

## [04] `ParticleExchange.refinanceLoan` FUNCTION DOES NOT PREVENT BORROWER FROM REFINANCING TO A LESS COMPETITIVE OFFERING
When calling the following `ParticleExchange.refinanceLoan` function, executing `uint256 newCredit = (oldLien.credit + oldLien.price - payableInterest) - newLien.price` does not guarantee that `newLien.price` is less than `oldLien.price`. This function also does not check if `newLien.rate` is less than `oldLien.rate` or not. Thus, it is possible that the borrower refinances to a less competitive offering. To prevent this, please consider updating the `ParticleExchange.refinanceLoan` function to check if the `newLien.price`-`newLien.rate` combination would generate less interest than that for the `oldLien.price`-`oldLien.rate` combination and revert if not.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L544-L613
```solidity
    function refinanceLoan(
        Lien calldata oldLien,
        uint256 oldLienId,
        Lien calldata newLien,
        uint256 newLienId
    ) external override validateLien(oldLien, oldLienId) validateLien(newLien, newLienId) {
        ...
        uint256 newCredit = (oldLien.credit + oldLien.price - payableInterest) - newLien.price;

        // update old lien
        liens[oldLienId] = keccak256(
            abi.encode(
                Lien({
                    lender: oldLien.lender,
                    borrower: address(0),
                    collection: oldLien.collection,
                    tokenId: newLien.tokenId,
                    price: oldLien.price,
                    rate: oldLien.rate,
                    loanStartTime: 0,
                    credit: 0,
                    auctionStartTime: 0
                })
            )
        );

        // update new lien
        liens[newLienId] = keccak256(
            abi.encode(
                Lien({
                    lender: newLien.lender,
                    borrower: oldLien.borrower,
                    collection: newLien.collection,
                    tokenId: newLien.tokenId,
                    price: newLien.price,
                    rate: newLien.rate,
                    loanStartTime: block.timestamp,
                    credit: newCredit,
                    auctionStartTime: 0
                })
            )
        );

        ...
    }
```

## [05] `_treasuryRate` CAN BE SET TO `MathUtils._BASIS_POINTS`, WHICH FORCES LENDERS TO EARN NO INTERESTS
When calling the following `ParticleExchange.setTreasuryRate` function, it is possible that `_treasuryRate` is set to `MathUtils._BASIS_POINTS`. Although such change might not be malicious, this would force lenders to earn no interests, which can be unexpected when the lenders are not aware of such change. To allow the lenders to earn at least some reasonable interests, please consider updating the `ParticleExchange.setTreasuryRate` function to revert if `rate` is above a reasonable upper threshold value, which should be less than `MathUtils._BASIS_POINTS`.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L800-L806
```solidity
    function setTreasuryRate(uint256 rate) external onlyOwner {
        if (rate > MathUtils._BASIS_POINTS) {
            revert Errors.InvalidParameters();
        }
        _treasuryRate = rate;
        emit UpdateTreasuryRate(rate);
    }
```

## [06] `ParticleExchange.receive` FUNCTION ALLOWS USERS TO ACCIDENTALLY SEND ETH TO `ParticleExchange` CONTRACT
The following `ParticleExchange.receive` function allows users to accidentally send ETH to the `ParticleExchange` contract. Such sent ETH amounts are lost in the perspective of these users. To avoid this, please consider updating the `ParticleExchange.receive` function to revert if `msg.value` is not from the registered marketplaces.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L826
```solidity
    receive() external payable {}
```

## [07] MISSING `address(0)` CHECK FOR `wethAddress` IN `ParticleExchange` CONTRACT'S CONSTRUCTOR
To prevent unexpected behaviors, `address` inputs for constructors should be checked against `address(0)`. The `address(0)` check is missing for `wethAddress` in the following `ParticleExchange` contract's constructor. Please consider checking it.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L32-L35
```solidity
    constructor(address wethAddress) {
        weth = WETH(payable(wethAddress));
        _disableInitializers();
    }
```

## [08] COMMENT FOR `interest` IN `MathUtils.calculateTreasuryProportion` FUNCTION IS INCORRECT
The comment for `interest` in the following `MathUtils.calculateTreasuryProportion` function mentions about the max auction buy price instead of the interest, which is incorrect. Please update such comment accordingly.

https://github.com/code-423n4/2023-05-particle/blob/9774cfd1684ae70981a54af42f6bff1716b3f203/contracts/libraries/math/MathUtils.sol#L44-L55
```solidity
    /**
     * @dev Calculates the proportion that goes into treasury
     * @param interest The max auction buy price in WEI
     * @param rateBips The treasury proportion in bips
     * @return treasuryAmount Amount goes into treasury in WEI
     */
    function calculateTreasuryProportion(
        uint256 interest,
        uint256 rateBips
    ) external pure returns (uint256 treasuryAmount) {
        return interest.wadMul(bipsToWads(rateBips));
    }
```

## [09] REDUNDANT NAMED RETURNS CAN BE REMOVED
When a function has a named return and a return statement, this named return becomes redundant. To improve maintainability, please consider removing the named returns for the following functions that have the corresponding return statements.

https://github.com/code-423n4/2023-05-particle/blob/9774cfd1684ae70981a54af42f6bff1716b3f203/contracts/libraries/math/MathUtils.sol#L19-L27
```solidity
contracts\libraries\math\MathUtils.sol
  23: ) external view returns (uint256 interest) {
  53: ) external pure returns (uint256 treasuryAmount) {
  60: function bipsToWads(uint256 bips) public pure returns (uint256 wad) {

contracts\protocol\ParticleExchange.sol
  52: ) external override returns (uint256 lienId) {
  101: ) internal returns (uint256 lienId) {
```

## [10] REPEATED CODE CAN BE REFACTORED INTO A MODIFIER TO BE USED IN CORRESPONDING FUNCTIONS
The following code are repeated in multiple functions. For better code organization and maintainability, please consider refactoring each of them into a modifier to be used in the corresponding functions.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L136-L138
```solidity
        if (msg.sender != lien.lender) {
            revert Errors.Unauthorized();
        }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L347-L349
```solidity
        if (msg.sender != lien.borrower) {
            revert Errors.Unauthorized();
        }
```

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L351-L353
```solidity
        if (lien.loanStartTime == 0) {
            revert Errors.InactiveLoan();
        }
```

## [11] CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS
To improve readability and maintainability, a constant can be used instead of the magic number. Please consider replacing the magic numbers, such as `1e18`, used in the following code with constants.

```solidity
contracts\libraries\math\MathUtils.sol
  24: uint256 loanTimeWad = (block.timestamp - loanStartTime) * 1e18; 
  61: return (bips * 1e18) / _BASIS_POINTS;   

contracts\protocol\ParticleExchange.sol
  80: if (data.length == 64) {
```

## [12] WORD TYPING TYPOS
`procceds` can be updated to `proceeds` in the following code comment.
https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L83
```solidity
                /// is matched with an existing lien (realize PnL) already. If procceds here, this NFT will be tied
```

`accure` can be updated to `accrue` in the following code comment.
https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L733
```solidity
        // accure interest to lender
```

## [13] FLOATING PRAGMAS
It is a best practice to lock pragmas instead of using floating pragmas to ensure that contracts are tested and deployed with the intended compiler version. Accidentally deploying contracts with different compiler versions can lead to unexpected risks and undiscovered bugs. Please consider locking pragmas for the following files.

```solidity
contracts\interfaces\IParticleExchange.sol
  2: pragma solidity ^0.8.17;    

contracts\libraries\math\MathUtils.sol
  2: pragma solidity ^0.8.17;    

contracts\libraries\types\Errors.sol
  2: pragma solidity ^0.8.17;    

contracts\libraries\types\Structs.sol
  2: pragma solidity ^0.8.17;    

contracts\protocol\ParticleExchange.sol
  2: pragma solidity ^0.8.17;    
```

## [14] INCOMPLETE NATSPEC COMMENTS
NatSpec comments provide rich code documentation. The following functions miss the `@param` and `@return` comments. Please consider completing the NatSpec comments for these functions.

```solidity
contracts\libraries\math\MathUtils.sol
  60: function bipsToWads(uint256 bips) public pure returns (uint256 wad) {

contracts\protocol\ParticleExchange.sol
  231: function _withdrawAccountInterest(address payable lender) internal {
```

## [15] MISSING NATSPEC COMMENTS
NatSpec comments provide rich code documentation. The following functions miss NatSpec comments. Please consider adding NatSpec comments for these functions.

```solidity
contracts\protocol\ParticleExchange.sol
  37: function initialize() external initializer {    
  95: function _supplyNft(    
  291: function _execSellNftToMarket(  
  395: function _execBuyNftFromMarket( 
  750: function _auctionConcluded(uint256 startTime) internal view returns (bool) { 
  754: function _calculateCurrentPayableInterest(Lien calldata lien) internal view returns (uint256) { 
  782: function _validateLien(Lien calldata lien, uint256 lienId) internal view returns (bool) {   
  790: function registerMarketplace(address marketplace) external onlyOwner {   
  795: function unregisterMarketplace(address marketplace) external onlyOwner {   
  800: function setTreasuryRate(uint256 rate) external onlyOwner {   
  808: function withdrawTreasury(address receiver) external onlyOwner {   	
```
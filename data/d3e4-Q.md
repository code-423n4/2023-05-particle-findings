
## Lender can be prevented from stopping auction
Since the call to `stopLoanAuction()` requires the lien to be validated, any change in the lien will revert the call. A frontrunner might change the stored lien digest just before the lender's call, which causes the lien validation to revert, by calling `addCredit()` (with any non-zero amount).
This might be mitigated by requiring that `addCredit()` makes the borrower solvent as per the current auction price.

## MAX_PRICE seems low
MAX_PRICE is set to only 1000 ETH. This seems to be too restrictive considering what exorbitant sums some NFTs are sold for.

## Multiplying a fraction
There are rounding errors of the form `a*(b/c) <= a*b/c` [here](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L25)
```solidity
interest = principal.wadMul(bipsToWads(rateBips).wadMul(loanTimeWad.wadDiv(_YEAR_WAD)));
```
and [here](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L41)
```solidity
return price.wadMul(auctionElapsed.wadDiv(auctionDuration));
```
Consider dividing the entire product last.

## Using proprietary WETH
Using a proprietary WETH implementation might instil doubt in some users. Consider using the already deployed and accepted WETH9 at 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2.

## Check that `receive` only accepts from approved market
The only time `ParticleExchange` receives funds through `receive()` is from a market place when selling an NFT via `sellNftToMarket()`, which [only allows registered marketplaces](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L299-L301). Consider similarly protecting `receive()` lest funds be elsewhence sent and lost.

## `buyNftFromMarket()` should not be `payable`
The trader is not supposed to send any funds in the call to [`buyNftFromMarket()`](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L338-L393); the funds are taken from the remaining credit and price which are [required to cover the buy amount](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L391). Any funds sent in this call will be lost.
Consider changing `buyNftFromMarket()` to `nonpayable` (in IParticleExchange.sol as well).

## The auction price increases quadratically
It is stated in the gitbook as well as in the comments that the auction price increases linearly. This is not quite correct. The auction price increases linearly with time AND max price, but since the max price also increases linearly with time as interest accrues the auction price actually increases quadratically. This might be worth a note.

## Typos
[interst -> interest](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L17)
[accured -> accrued](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L13)
[linearly increase in time -> increases linearly with time](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L30)
[accured -> accrued](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/interfaces/IParticleExchange.sol#L83)
[accured -> accrued](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/interfaces/IParticleExchange.sol#L90)
[accured -> accrued](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L182)
[accure -> accrue](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L384)
[accure -> accrue](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L501)
[accure -> accrue](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L609)
[accure -> accrue](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L733)
[If procceds -> If it proceeds](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L83)
# Summary

| Id | Title |
| -- | ----- |
| G-1 | Function `withdrawEthWithInterest()` calls `transfer()` for lender twice |
| G-2 | `internal` functions only called once can be inlined to save gas |
| G-3 | Optimize names to save gas |
| G-4 | Use a more recent version of solidity |

# G-1. Function `withdrawEthWithInterest()` calls `transfer()` for lender twice

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L192

## Detail
Function `withdrawEthWithInterest()` does ETH transfers to lender twice in one transaction. It could be optimized to transfer only once.
```solidity
payable(lien.lender).transfer(lien.price + payableInterest);

// transfer PnL to borrower
if (lien.credit > payableInterest) {
    payable(lien.borrower).transfer(lien.credit - payableInterest);
}

emit WithdrawETH(lienId);

// withdraw interest from this account too
_withdrawAccountInterest(payable(msg.sender));
```

## Recommendation
ETH is transferred to lender in first line and last line in, which is twice. Consider grouping them into only 1 transfer.

# G-2. `internal` functions only called once can be inlined to save gas
https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L298

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L402

## Detail
Not inlining costs 20 to 40 gas because of two extra JUMP instructions and additional stack operations needed for function calls.

# G-3. Optimize names to save gas

Contracts most called functions could simply save gas by function ordering via Method ID. Calling a function at runtime will be cheaper if the function is positioned earlier in the order (has a relatively lower Method ID) because 22 gas are added to the cost of a function for every position that came before it. The caller can save on gas if you prioritize most called functions. 

## Recommendation
Find a lower method ID name for the most called functions for example Call() vs. Call1() is cheaper by 22 gas For example, the function IDs in the Core.sol contract will be the most used; A lower method ID may be given.

## Proof of Consept
https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92

# G-4. Use a more recent version of solidity

I recommend using the latest strictness version 0.8.19. Reference: https://blog.soliditylang.org/2023/02/22/solidity-0.8.19-release-announcement/

## L-1 Users may receive less Tokens due to slippage 

When depositing to the variable pool users may receive less tokens than expected due to slippage factors, like available liquidity interest rate model, change in token price, utilization rate. As shown below when deposits happen 

```solidity
 uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));

        state.data.underlyingBorrowToken.forceApprove(address(state.data.variablePool), amount);
        state.data.variablePool.supply(address(state.data.underlyingBorrowToken), amount, address(this), 0);
        //@audit L1- Slippgae When Depositing Tokens, the expected tokens may be less than expected
        uint256 scaledAmount = aToken.scaledBalanceOf(address(this)) - scaledBalanceBefore;

        state.data.borrowAToken.mintScaled(to, scaledAmount);
```
As shown in that snippet, no slippage protection is implemented leading to the user receiving less tokens than expected
## Excessive Gas Consumption in `validateUserIsNotBelowOpeningLimitBorrowCR` Function

### Lines of Code
[RiskLibrary.sol#L140-L150](https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/RiskLibrary.sol#L140-L150)

### Description
The `validateUserIsNotBelowOpeningLimitBorrowCR` function calls the `collateralRatio(state, account)` function twice, which leads to unnecessary gas consumption. Each call to `collateralRatio` reads from the contract's state, which is a gas-intensive operation. By storing the result of `collateralRatio(state, account)` in a temporary variable, you can avoid redundant state reads and reduce gas costs.

### Proof of Concept
The function calls `collateralRatio(state, account)` twice:
```solidity
if (collateralRatio(state, account) < openingLimitBorrowCR) {
    revert Errors.CR_BELOW_OPENING_LIMIT_BORROW_CR(
        account, collateralRatio(state, account), openingLimitBorrowCR
    );
}
```

### Recommended Mitigation Steps
To optimize the gas consumption, store the result of `collateralRatio(state, account)` in a temporary variable and use this variable in the condition check and revert statement. Here is the optimized implementation:
```solidity
function validateUserIsNotBelowOpeningLimitBorrowCR(State storage state, address account) external view {
    uint256 openingLimitBorrowCR = Math.max(
        state.riskConfig.crOpening,
        state.data.users[account].openingLimitBorrowCR // 0 by default, or user-defined if SetUserConfiguration has been used
    );
    uint256 userCollateralRatio = collateralRatio(state, account);
    if (userCollateralRatio < openingLimitBorrowCR) {
        revert Errors.CR_BELOW_OPENING_LIMIT_BORROW_CR(
            account, userCollateralRatio, openingLimitBorrowCR
        );
    }
}
```

### Benefits of the Fix
- **Reduced Gas Costs**: Eliminates redundant state reads, reducing the overall gas consumption.
- **Improved Performance**: Reduces the number of state accesses, enhancing the function's efficiency.
- **Enhanced Code Readability**: Storing the result in a variable makes the code cleaner and easier to maintain.

## Unused `State storage` Parameter in `validateMinimumCollateralProfit` Function

### Lines of Code
[Liquidate.sol#L59-L69](https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L59-L69)

### Description
The function `validateMinimumCollateralProfit` in the `Liquidate.sol` contains an unused parameter `State storage`. This parameter is included in the function signature but is not utilized within the function body. Including this unused parameter results in unnecessary gas consumption and reduces code clarity.

### Proof of Concept
`State storage` is declared but not used within the function body.
```solidity
function validateMinimumCollateralProfit(
    State storage,
    LiquidateParams calldata params,
    uint256 liquidatorProfitCollateralToken
) external pure {
    if (liquidatorProfitCollateralToken < params.minimumCollateralProfit) {
        revert Errors.LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(
            liquidatorProfitCollateralToken, params.minimumCollateralProfit
        );
    }
}
```

### Recommended Mitigation Steps
Remove the unused `State storage` parameter from the function signature to optimize gas usage and improve code readability. The revised function definition should look like this:
```solidity
function validateMinimumCollateralProfit(
    LiquidateParams calldata params,
    uint256 liquidatorProfitCollateralToken
) external pure {
    if (liquidatorProfitCollateralToken < params.minimumCollateralProfit) {
        revert Errors.LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(
            liquidatorProfitCollateralToken, params.minimumCollateralProfit
        );
    }
}
```

### Benefits of the Fix
- **Gas Optimization**: Eliminating the unnecessary `State storage` parameter reduces gas costs associated with passing storage variables.
- **Code Readability**: Removing unused parameters simplifies the function signature and makes the code easier to understand and maintain.
- **Best Practices**: Adhering to best practices by avoiding unused variables leads to cleaner and more efficient code, which is easier to audit and less prone to errors.

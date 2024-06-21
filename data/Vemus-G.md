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

## Excessive Gas Consumption in `validateSelfLiquidate` Function

### Lines of Code
[SelfLiquidate.sol#L34-L54](https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/SelfLiquidate.sol#L34-L54)

### Description
The `validateSelfLiquidate` function in the contract makes redundant calls to `state.collateralRatio(debtPosition.borrower)`, which results in unnecessary gas consumption. The same function is called three times within the same execution context, which is inefficient and increases the overall cost of gas for this operation.

### Proof of Concept
In the provided `validateSelfLiquidate` function, the `state.collateralRatio(debtPosition.borrower)` function is invoked three times. This is redundant and can be optimized by storing the result of the first call in a local variable and reusing it:
```
        if (!state.isCreditPositionSelfLiquidatable(params.creditPositionId)) {
            revert Errors.LOAN_NOT_SELF_LIQUIDATABLE(
                params.creditPositionId,
                state.collateralRatio(debtPosition.borrower),
                state.getLoanStatus(params.creditPositionId)
            );
        }
        if (state.collateralRatio(debtPosition.borrower) >= PERCENT) {
            revert Errors.LIQUIDATION_NOT_AT_LOSS(params.creditPositionId, state.collateralRatio(debtPosition.borrower));
        }
```

### Recommended Mitigation Steps
Refactor the `validateSelfLiquidate` function to call `state.collateralRatio(debtPosition.borrower)` only once and store the result in a local variable. Use this variable for subsequent checks within the function. This reduces the number of external function calls, thereby optimizing gas usage.
```
function validateSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external view {
    CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);
    DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);

    // Store the collateral ratio in a local variable
    uint256 collateralRatio = state.collateralRatio(debtPosition.borrower);

    // validate creditPositionId
    if (!state.isCreditPositionSelfLiquidatable(params.creditPositionId)) {
        revert Errors.LOAN_NOT_SELF_LIQUIDATABLE(
            params.creditPositionId,
            collateralRatio,
            state.getLoanStatus(params.creditPositionId)
        );
    }
    if (collateralRatio >= PERCENT) {
        revert Errors.LIQUIDATION_NOT_AT_LOSS(params.creditPositionId, collateralRatio);
    }

    // validate msg.sender
    if (msg.sender != creditPosition.lender) {
        revert Errors.LIQUIDATOR_IS_NOT_LENDER(msg.sender, creditPosition.lender);
    }
}
```

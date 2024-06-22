Enhance `executeSelfLiquidate` Function to Handle Zero Collateral Transfer

**Issue**

The function does not efficiently handle the scenario where no collateral (assignedCollateral == 0) is assigned to the credit position. This leads to unnecessary gas costs for executing a transfer operation with zero tokens.

```diff
function executeSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external {
    emit Events.SelfLiquidate(params.creditPositionId);

    CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);
    DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);

    uint256 assignedCollateral = state.getCreditPositionProRataAssignedCollateral(creditPosition);

    // Debt and credit reduction
    state.reduceDebtAndCredit(creditPosition.debtPositionId, params.creditPositionId, creditPosition.credit);

    // Transfer collateral only if assignedCollateral is greater than 0
    if (assignedCollateral > 0) {
        state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);
    } else {
        revert("No collateral to transfer");
    }
}
```

By introducing the conditional check, we avoid unnecessary token transfer operations (transferFrom) when no collateral is assigned (assignedCollateral == 0). This results in significant gas savings, especially during periods of high network activity.
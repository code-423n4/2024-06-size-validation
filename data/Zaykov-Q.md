In the `isCreditPositionSelfLiquidatable` the variables for `creditPosition`,  `debtPosition` are not consistent as everywhere else in the contract. 
```
function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)
        public
        view
        returns (bool)
    {
        CreditPosition storage creditPosition = state.data.creditPositions[creditPositionId];
        DebtPosition storage debtPosition = state.data.debtPositions[creditPosition.debtPositionId];
        LoanStatus status = state.getLoanStatus(creditPositionId);
        // Only CreditPositions can be self liquidated
        return state.isCreditPositionId(creditPositionId)
            && (isUserUnderwater(state, debtPosition.borrower) && status != LoanStatus.REPAID);
    }
```

They should be changed as everywhere in the code for  consistency and maintainability.
``` diff
+  CreditPosition storage creditPosition = state.getCreditPosition(creditPositionId);
+  DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(creditPosition.debtPositionId);
```

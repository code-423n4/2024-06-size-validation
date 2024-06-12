# Title
Lack of check on `debtPositionId` in `isCreditPositionSelfLiquidatable` function

# Code
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/RiskLibrary.sol#L66-L82

# Description
In `RiskLibrary`, the function `isCreditPositionSelfLiquidatable` checks if a credit position is self-liquidatable by verifying if the user is underwater and the loan is not repaid. However, it fails to consider the case where the credit position is not associated with a valid debt position.

# Recommendation
To fix this vulnerability, the `isCreditPositionSelfLiquidatable` function should include an additional check to verify if the credit position is associated with a valid debt position:
```solidity
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
            && state.isDebtPositionId(creditPosition.debtPositionId)
            && (isUserUnderwater(state, debtPosition.borrower) && status != LoanStatus.REPAID);
    }
```
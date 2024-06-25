| ID | Title |
| --- | --- |
| NC-01 | Code Optimization - 1 |
| NC-02 | Code Optimization - 2 |
| NC-03 | Usage of Magic Strings for `UpdateConfigParams` Keys |
| NC-04 | Typo in Comments |

## NC-01: **Code Optimization - 1**

The code can be optimized to avoid executing unnecessary function calls and calculations.

### Lines of affected code:
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/RiskLibrary.sol#L53-L64

### Existing code:
```solidity
function collateralRatio(State storage state, address account) public view returns (uint256) {
    uint256 collateral = state.data.collateralToken.balanceOf(account);
    uint256 debt = state.data.debtToken.balanceOf(account);
    uint256 debtWad = Math.amountToWad(debt, state.data.underlyingBorrowToken.decimals());
    uint256 price = state.oracle.priceFeed.getPrice();

    if (debt != 0) {
        return Math.mulDivDown(collateral, price, debtWad);
    } else {
        return type(uint256).max;
    }
}
```

### Proposed optimized code:
```solidity
function collateralRatio(State storage state, address account) public view returns (uint256) {
    uint256 debt = state.data.debtToken.balanceOf(account);
    if (debt == 0) {
        return type(uint256).max;
    }
    
    uint256 collateral = state.data.collateralToken.balanceOf(account);
    uint256 debtWad = Math.amountToWad(debt, state.data.underlyingBorrowToken.decimals());
    uint256 price = state.oracle.priceFeed.getPrice();
    
    return Math.mulDivDown(collateral, price, debtWad);
}
```

## NC-02: **Code Optimization - 2**

The code can be optimized to avoid executing unnecessary function calls and calculations.

### Lines of affected code:
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Compensate.sol#L42-L101

### Proposed optimization:
```diff
function validateCompensate(State storage state, CompensateParams calldata params) external view {
    CreditPosition storage creditPositionWithDebtToRepay =
        state.getCreditPosition(params.creditPositionWithDebtToRepayId);
    DebtPosition storage debtPositionToRepay =
        state.getDebtPositionByCreditPositionId(params.creditPositionWithDebtToRepayId);

+   // validate msg.sender
+   if (msg.sender != debtPositionToRepay.borrower) {
+       revert Errors.COMPENSATOR_IS_NOT_BORROWER(msg.sender, debtPositionToRepay.borrower);
+   }

    uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);

+   // validate amount
+   if (amountToCompensate == 0) {
+       revert Errors.NULL_AMOUNT();
+   }

    // validate creditPositionWithDebtToRepayId
    if (state.getLoanStatus(params.creditPositionWithDebtToRepayId) != LoanStatus.ACTIVE) {
        revert Errors.LOAN_NOT_ACTIVE(params.creditPositionWithDebtToRepayId);
    }

    // validate creditPositionToCompensateId
    if (params.creditPositionToCompensateId == RESERVED_ID) {
        uint256 tenor = debtPositionToRepay.dueDate - block.timestamp;

        // validate tenor
        if (tenor < state.riskConfig.minTenor || tenor > state.riskConfig.maxTenor) {
            revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);
        }
    } else {
        CreditPosition storage creditPositionToCompensate =
            state.getCreditPosition(params.creditPositionToCompensateId);
        DebtPosition storage debtPositionToCompensate =
            state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId);
        if (!state.isCreditPositionTransferrable(params.creditPositionToCompensateId)) {
            revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(
                params.creditPositionToCompensateId,
                state.getLoanStatus(params.creditPositionToCompensateId),
                state.collateralRatio(debtPositionToCompensate.borrower)
            );
        }
        if (
            debtPositionToRepay.dueDate
                < state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId).dueDate
        ) {
            revert Errors.DUE_DATE_NOT_COMPATIBLE(
                params.creditPositionWithDebtToRepayId, params.creditPositionToCompensateId
            );
        }
        if (creditPositionToCompensate.lender != debtPositionToRepay.borrower) {
            revert Errors.INVALID_LENDER(creditPositionToCompensate.lender);
        }
        if (params.creditPositionToCompensateId == params.creditPositionWithDebtToRepayId) {
            revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionToCompensateId);
        }
        amountToCompensate = Math.min(amountToCompensate, creditPositionToCompensate.credit);
    }

-   // validate msg.sender
-   if (msg.sender != debtPositionToRepay.borrower) {
-       revert Errors.COMPENSATOR_IS_NOT_BORROWER(msg.sender, debtPositionToRepay.borrower);
-   }

-   // validate amount
-   if (amountToCompensate == 0) {
-       revert Errors.NULL_AMOUNT();
-   }
}
```

## NC-03: Usage of Magic Strings for `UpdateConfigParams` Keys

The function `executeUpdateConfig` makes use of magic strings.

### Lines of affected code:
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/UpdateConfig.sol#L86-L148

### Affected code (see the equality checking of `params.key` with the respective magic strings):
```solidity
function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {
    if (Strings.equal(params.key, "crOpening")) {
        state.riskConfig.crOpening = params.value;
    } else if (Strings.equal(params.key, "crLiquidation")) {
        if (params.value >= state.riskConfig.crLiquidation) {
            revert Errors.INVALID_COLLATERAL_RATIO(params.value);
        }
        state.riskConfig.crLiquidation = params.value;
    } else if (Strings.equal(params.key, "minimumCreditBorrowAToken")) {
        state.riskConfig.minimumCreditBorrowAToken = params.value;
    } else if (Strings.equal(params.key, "borrowATokenCap")) {
        state.riskConfig.borrowATokenCap = params.value;
    } else if (Strings.equal(params.key, "minTenor")) {
        if (
            state.feeConfig.swapFeeAPR != 0
                && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
        ) {
            revert Errors.VALUE_GREATER_THAN_MAX(
                params.value, Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
            );
        }
        state.riskConfig.minTenor = params.value;
    } else if (Strings.equal(params.key, "maxTenor")) {
        if (
            state.feeConfig.swapFeeAPR != 0
                && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
        ) {
            revert Errors.VALUE_GREATER_THAN_MAX(
                params.value, Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
            );
        }
        state.riskConfig.maxTenor = params.value;
    } else if (Strings.equal(params.key, "swapFeeAPR")) {
        if (params.value >= Math.mulDivDown(PERCENT, YEAR, state.riskConfig.maxTenor)) {
            revert Errors.VALUE_GREATER_THAN_MAX(
                params.value, Math.mulDivDown(PERCENT, YEAR, state.riskConfig.maxTenor)
            );
        }
        state.feeConfig.swapFeeAPR = params.value;
    } else if (Strings.equal(params.key, "fragmentationFee")) {
        state.feeConfig.fragmentationFee = params.value;
    } else if (Strings.equal(params.key, "liquidationRewardPercent")) {
        state.feeConfig.liquidationRewardPercent = params.value;
    } else if (Strings.equal(params.key, "overdueCollateralProtocolPercent")) {
        state.feeConfig.overdueCollateralProtocolPercent = params.value;
    } else if (Strings.equal(params.key, "collateralProtocolPercent")) {
        state.feeConfig.collateralProtocolPercent = params.value;
    } else if (Strings.equal(params.key, "feeRecipient")) {
        state.feeConfig.feeRecipient = address(uint160(params.value));
    } else if (Strings.equal(params.key, "priceFeed")) {
        state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));
    } else if (Strings.equal(params.key, "variablePoolBorrowRateStaleRateInterval")) {
        state.oracle.variablePoolBorrowRateStaleRateInterval = uint64(params.value);
    } else {
        revert Errors.INVALID_KEY(params.key);
    }

    Initialize.validateInitializeFeeConfigParams(feeConfigParams(state));
    Initialize.validateInitializeRiskConfigParams(riskConfigParams(state));
    Initialize.validateInitializeOracleParams(oracleParams(state));

    emit Events.UpdateConfig(params.key, params.value);
}
```

## NC-04: Typo in Comments

### Lines of affected code:
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/AccountingLibrary.sol#L133

### Proposed spelling correction:
```diff
- ///    If the loan is in REPAID status, this is expected, as lenders grdually claim their credit.
+ ///    If the loan is in REPAID status, this is expected, as lenders gradually claim their credit.
```
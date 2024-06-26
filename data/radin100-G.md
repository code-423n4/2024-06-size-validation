### [GAS-1]
The check at https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/UpdateConfig.sol#L98-L108, could be simplified as shown below due to it's enough to make sure that minTenor < maxTenor 
```solidity
function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {

        .....

        } else if (Strings.equal(params.key, "minTenor")) {
           if (
---                state.feeConfig.swapFeeAPR != 0
---                 && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
+++                params.value >= state.riskConfig.maxTenor
            ) {
                revert Errors.VALUE_GREATER_THAN_MAX(
                    params.value, Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
                );
            }
            state.riskConfig.minTenor = params.value;
        } 

        .....

        } else {
            revert Errors.INVALID_KEY(params.key);
        }
```
### [GAS-2]
No CreditPosition initialisation in memory needed if `params.creditPositionId == RESERVED_ID` in `BuyCreditMarket::executeBuyCreditMarket`
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/BuyCreditMarket.sol#L130C9-L130C46
```diff
function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
        external
        returns (uint256 cashAmountIn)
    {
        .....

---     CreditPosition memory creditPosition;
        uint256 tenor;
        address borrower;
        if (params.creditPositionId == RESERVED_ID) {
            borrower = params.borrower;
            tenor = params.tenor;
        } else {
+++     CreditPosition memory creditPosition;
            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);
            creditPosition = state.getCreditPosition(params.creditPositionId);

            borrower = creditPosition.lender;
            tenor = debtPosition.dueDate - block.timestamp;
        }
        
        .....

    }
```
### Repetition of check in validateBuyCreditLimit:
In validateBuyCreditLimit() function  "maxDueDate == 0" is checked and gives Error on revert but this condition will never meet because same check is carried in "loanOffer.isNull()" , In case of maxDueDate is 0 it will not succeed in 1st if statement 

```
  function validateBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external view {
        LoanOffer memory loanOffer =
            LoanOffer({maxDueDate: params.maxDueDate, curveRelativeTime: params.curveRelativeTime});

        // a null offer mean clearing their limit order
        if (!loanOffer.isNull()) {
            // validate msg.sender
            // N/A

            // validate maxDueDate
            /// @ audit this check is done in isNull() 
            if (params.maxDueDate == 0) {
                revert Errors.NULL_MAX_DUE_DATE();
            }
            if (params.maxDueDate < block.timestamp + state.riskConfig.minTenor) {
                revert Errors.PAST_MAX_DUE_DATE(params.maxDueDate);
            }

            // validate curveRelativeTime
            YieldCurveLibrary.validateYieldCurve(
                params.curveRelativeTime, state.riskConfig.minTenor, state.riskConfig.maxTenor
            );
        }
    }
```


```
 function isNull(LoanOffer memory self) internal pure returns (bool) {
        return self.maxDueDate == 0 && self.curveRelativeTime.isNull();
    }
```
The next code lack validation, which can lead to users think they get a wrong repay validation

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src\libraries\actions\Repay.sol#L30-L44

### Sugestions
Change the code as follows

```javascript
    /// @notice Validates the input parameters for repaying a debt position
    /// @param state The state
    /// @param params The input parameters for repaying a debt position
    function validateRepay(State storage state, RepayParams calldata params) external view returns(bool) {
        // validate debtPositionId
        if (state.getLoanStatus(params.debtPositionId) == LoanStatus.REPAID) {
            revert Errors.LOAN_ALREADY_REPAID(params.debtPositionId);
        }

        // validate msg.sender
        // N/A
       return true;
    }
```
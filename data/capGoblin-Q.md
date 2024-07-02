# QA Report


## [L-01] Potential DoS Vulnerability in `Size::setUserConfiguration`
The `executeSetUserConfiguration` function used allows users to set their configuration, including updating multiple credit positions. This function iterates over the `params.creditPositionIds` array to update each credit position. 
If this array is very large, the function could consume a significant amount of gas, potentially leading to the transaction running out of gas and failing. This situation could be exploited to cause a DoS by repeatedly submitting transactions with large arrays, thereby disrupting the normal operation of the contract.
To mitigate this risk, consider adding a limit on the length of params.creditPositionIds or processing large lists in smaller, manageable chunks.

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SetUserConfiguration.sol#L63-L83
```solidity
    function executeSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params) external {
        User storage user = state.data.users[msg.sender];

        user.openingLimitBorrowCR = params.openingLimitBorrowCR;
        user.allCreditPositionsForSaleDisabled = params.allCreditPositionsForSaleDisabled;

@>      for (uint256 i = 0; i < params.creditPositionIds.length; i++) {
            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionIds[i]);
            creditPosition.forSale = params.creditPositionIdsForSale;
            emit Events.UpdateCreditPosition(
                params.creditPositionIds[i], creditPosition.lender, creditPosition.credit, creditPosition.forSale
            );
        }

        emit Events.SetUserConfiguration(
            params.openingLimitBorrowCR,
            params.allCreditPositionsForSaleDisabled,
            params.creditPositionIdsForSale,
            params.creditPositionIds
        );
    }
```

## [L-02] Avoid Repetition in comments in `RiskLibrary`
The documentation comment (@dev) for the isDebtPositionLiquidatable function contains a redundancy in describing the conditions under which a debt position is considered liquidatable. It states that a debt position is liquidatable if the user is underwater and the loan is not REPAID (i.e., ACTIVE or OVERDUE), and then redundantly specifies that a loan being OVERDUE is a condition for liquidation.
`/// @dev A debt position is liquidatable if the user is underwater and the loan is not REPAID (ie, ACTIVE or OVERDUE) or
///        if the loan is OVERDUE.` can be written as `/// @dev A debt position is liquidatable if the user is underwater and the loan is not REPAID (ACTIVE or OVERDUE).`

## [L-03] Use of Non-Descriptive Parameter Names in `initialize` Function

Renaming the parameters `f`, `r`, `o`, and `d` to more descriptive names such as `feeConfig`, `riskConfig`, `oracleParams`, and `dataParams` will enhance the readability and maintainability of the code.

## [L-04] Improve Readability of `isCreditPositionSelfLiquidatable` Function
The `isCreditPositionSelfLiquidatable` function in the contract checks whether a credit position is self-liquidatable by performing several conditional checks. The current implementation can be made more readable by breaking down the complex return statement into intermediate boolean variables with descriptive names

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/RiskLibrary.sol#L80-L81
```solidity 
        // Only CreditPositions can be self liquidated
        return
            state.isCreditPositionId(creditPositionId) &&
            (isUserUnderwater(state, debtPosition.borrower) &&
                status != LoanStatus.REPAID);
```
to
```solidity
    bool isUnderwater = isUserUnderwater(state, debtPosition.borrower);
    bool isCreditPosition = state.isCreditPositionId(creditPositionId);
    bool isNotRepaid = status != LoanStatus.REPAID;
    // Only CreditPositions can be self liquidated
    return isCreditPosition && isUnderwater && isNotRepaid;
```
Similarly to `isDebtPositionLiquidatable`, `isCreditPositionTransferrable` functions in `RiskLibrary` as well


## [L-06] Missing Inline Comments
In the `AccountingLibrary.sol`, the functions `createDebtAndCreditPositions` and `createCreditPosition` lack inline comments explaining their logic and operations. These functions play a crucial role in the protocol's functionality, handling the creation and management of debt and credit positions

## [L-05] Use of Public/External Functions Instead of Internal in Libraries in many places
Many functions in Libraries are using `public` or `external` visibility for functions that are meant to be `internal` which can expose the internal workings of the contract to external calls, increasing the attack surface and potential security risks. several functions are declared as public or external instead of internal.
E.g, `validateUserIsNotBelowOpeningLimitBorrowCR` function in `RiskLibrary` is declared as `public`. But, this function is only used within the `Size.sol` contract by importing the library and using the `using` directive, for which internal visibility is sufficient and more appropriate. If there is an intention behind declaring functions as `public` or `external`, this should be explicitly documented to explain their purpose

## [L-07] Order of functions

The solidity documentation recommends the following order for functions:
- constructor
- receive function (if exists)
- fallback function (if exists)
- external
- public
- internal
- private

Ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. But there are contracts in the project that do not comply with this. Eg. `executeInitialize` function in `Initialize.sol` which is an external function defined at the end.

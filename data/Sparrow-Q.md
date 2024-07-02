## [QA-01] Array Indexing Issue in `YieldCurveLibrary`
## Description
The issue identified in the `validateYieldCurve` function within the `YieldCurveLibrary` pertains to the loop condition used for iterating over the tenors array in reverse order. The initial value of the loop counter `i` is set to `self.tenors.length`, which exceeds the valid index range for the array, leading to a potential out-of-bounds access. This is problematic because array indices in Solidity start at 0, and the highest valid index for an array of length `n` is `n - 1`. Accessing an element beyond the array's bounds can cause undefined behavior or a runtime error.
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/YieldCurveLibrary.sol#L63-L64

## Recommended Mitigation
To address this issue, the loop condition should be revised to ensure that the iteration starts from the last valid index of the array and proceeds towards 0. This can be achieved by initializing `i` to `self.tenors.length - 1` and decrementing `i` until it reaches 0. This adjustment guarantees that all elements are accessed without exceeding the array's bounds, thereby preventing potential out-of-bounds access errors.
Here is the corrected version of the loop condition:
```
for (uint256 i = self.tenors.length - 1; i >= 0; i--) {
    // Loop body
}
```

## [QA-02] Incorrect Comparison Operator in `validateMinimumCredit` Function
## Description
The `validateMinimumCredit` function in the `RiskLibrary` checks if the remaining credit is below the minimum required credit amount (`state.riskConfig.minimumCreditBorrowAToken`). However, the current implementation uses the comparison operator `>` to check if the credit is greater than 0 and less than the minimum required credit. This allows the function to pass when the credit equals the minimum required credit, which is incorrect.
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/RiskLibrary.sol#L21-L26
## Recommendation
Update the `validateMinimumCredit` function to use the `>=` comparison operator instead of `>` to ensure that the check fails when the credit equals the minimum required credit value.

## [QA-03] State Management Issue in Multicall Function Leading to Inconsistent State
## Description
The `multicall` function in the `Multicall.sol` has a state management issue related to the `state.data.isMulticall` flag. Specifically, if any call within the batch fails, the `state.data.isMulticall` flag remains set to true, causing the contract to be in an inconsistent state. This can lead to incorrect behavior or failures in future operations that depend on this flag being false.

## Recommended Mitigation
To ensure that the `state.data.isMulticall` flag is reset to false even if an error occurs, implement a `try...catch` block within the `multicall` function. This block will manage exceptions and ensure the flag is always reset, maintaining a consistent contract state.

## [QA-04] Usability becomes very limited when access to Chainlink oracle data feed is blocked
## Description
The `PriceFeed` contract relies on Chainlink oracles to fetch price data for critical protocol operations such as borrowing, withdrawing, and liquidating assets. As https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ mentions, it is possible that Chainlink’s “multisigs can immediately block access to price feeds at will”. If Chainlink's multisig entities block access to price feeds, functions like `getPrice()` in the contract can revert, leading to denial of service and impairing the protocol's functionality. This issue could severely impact user transactions and operational continuity.
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/oracle/PriceFeed.sol#L63-L78

## Recommended Mitigation
Wrap calls to oracles in `try/catch` blocks to gracefully handle errors when they occur. This allows the contract to manage failures without reverting all related transactions.

## [QA-05] Missing Zero Address Check
First intance:
## Description
In the `LoanLibrary.sol` contract, there is a potential issue related to the absence of checks for zero addresses (address(0)) when setting the `borrower` or `lender` in the `DebtPosition` and `CreditPosition` structs
This omission poses a risk because it allows for the possibility of these fields being set to the zero address, which could lead to unexpected behavior or vulnerabilities in the contract logic.
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/LoanLibrary.sol#L14-L43

## Recommended Mitigation
To prevent issues arising from the use of zero addresses, it's crucial to add checks to ensure that borrower and lender addresses are not zero. This can be done using the require statement in functions that set these addresses.

Second instance:
## Description
In the `executeBuyCreditMarket` function, there are no checks to ensure that `msg.sender` and `params.borrower` are not zero addresses. Allowing zero addresses can lead to several issues, such as loss of funds, incorrect contract logic, and potential security vulnerabilities.
If a zero address is used in a function that transfers tokens or funds, those assets may be irretrievably lost because the zero address is a non-existent address in Ethereum.
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/BuyCreditMarket.sol#L121-L198
## Recommended Mitigation
To prevent these issues, you should add checks to ensure that `msg.sender` and `params.borrower` are not zero addresses. This can be done using the require statement to enforce these checks.

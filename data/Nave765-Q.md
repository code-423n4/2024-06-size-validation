The `BuyCreditLimit::validateBuyCreditLimit` method incorrectly uses `minTenor` instead of `maxTenor`, potentially allowing `LoanOffer` creation that exceeds the maximum due date.

## Impact

The method `BuyCreditLimit::validateBuyCreditLimit` does not correctly validate the `dueDate` because it uses `minTenor` instead of `maxTenor`. As a result, it allows the creation of a `LoanOffer` even if `block.timestamp + tenor` exceeds the maximum due date.

## Proof of Concept

In the `BuyCreditLimit` library, the `validateBuyCreditLimit` function is intended to validate the input parameters for buying credit as a limit order. However, one of the validations is not functioning as intended. The function uses `state.riskConfig.minTenor` instead of `state.riskConfig.maxTenor`, which allows the possibility for a `LoanOffer` to exceed the maximum due date.

```javascript
    if (params.maxDueDate < block.timestamp + state.riskConfig.minTenor) {
            revert Errors.PAST_MAX_DUE_DATE(params.maxDueDate);
    }
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

```diff
-   if (params.maxDueDate < block.timestamp + state.riskConfig.minTenor) {
+   if (params.maxDueDate < block.timestamp + state.riskConfig.maxTenor) {
            revert Errors.PAST_MAX_DUE_DATE(params.maxDueDate);
    }
```
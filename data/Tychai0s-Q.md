## Title

Potential Inconsistency in Tenor Configuration

## Impact

In [`UpdateConfig.sol:98`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/UpdateConfig.sol#L98), there's a risk of setting an inconsistent configuration where the minimum tenor could potentially exceed the maximum tenor. This oversight in validation could lead to logical conflicts within the system, potentially causing unexpected behavior or errors in tenor-dependent operations.

## Proof of Concept

The vulnerability occurs in the configuration update process:

```solidity
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
```

The code checks if the new `minTenor` value is less than a calculated maximum based on `swapFeeAPR`, but it doesn't verify if it's less than or equal to the current `maxTenor`. Similarly, when updating `maxTenor`, there's no check to ensure it remains greater than or equal to `minTenor`.

## Tools Used

Manual Review

## Recommended Mitigation Steps

To address this issue, implement additional checks when updating either `minTenor` or `maxTenor`:

1. When updating `minTenor`:
   ```solidity
   if (params.value > state.riskConfig.maxTenor) {
       revert Errors.MIN_TENOR_EXCEEDS_MAX_TENOR(params.value, state.riskConfig.maxTenor);
   }
   ```

2. When updating `maxTenor`:
   ```solidity
   if (params.value < state.riskConfig.minTenor) {
       revert Errors.MAX_TENOR_BELOW_MIN_TENOR(params.value, state.riskConfig.minTenor);
   }
   ```

These checks will ensure that `minTenor` always remains less than or equal to `maxTenor`, maintaining logical consistency in the tenor configuration.

## Issue Type

Invalid Validation
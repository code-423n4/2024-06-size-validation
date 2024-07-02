# L-01 Miscalculation of CreditAmountIn in sell credit market order with exact cashAmountout and fragmentation fee

## Impact

There is a slight error when calculating `creditAmountIn` in the function `getCreditAmountIn` in `AccountLibrary.sol`. In the case of credit fractionalization the credit is calculated as follows:

```
            creditAmountIn = Math.mulDivUp(
                cashAmountOut + state.feeConfig.fragmentationFee, PERCENT + ratePerTenor, PERCENT - swapFeePercent
            );
```
We can observe that the swap fee is applied to the swapped amount plus the fragmentation fee (instead of only the swapped amount), meaning that the borrower credit is overestimated. In practice, the impact is however only a few cents since `fragmentationFee` is a low constant value.

## Proof of Concept

Let's say Alice wants to borrow 100 USDC from Bob. Alice chooses cashAmountOut = 100. To more easily showcase the issue we use a high `swapFee` of 10%.

```
cashAmountOut = 100
Rate = 10%
Swap fee = 10%
Fragmentation fee = 5 USDC
```

Bob pays 100 USDC to Alice and 15 USDC in fees = 115. According to the formula in the code, the credit is calculated as:

```
CreditAmountIn = (cashAmountOut + state.feeConfig.fragmentationFee) * ((PERCENT + ratePerTenor) / (PERCENT - swapFeePercent))

= (100 + 5) * ((100 + 10) / (100 - 10)) = 128.33333333333334
```

In this case, Bob gets a rate of 10.38% and Alice pays Bob an extra 0.61 USD, which equals 10% of the fragmentation fee plus interest.

The correct calculation would be:

```
CreditAmountIn = (cashAmountOut) * ((PERCENT + ratePerTenor) / (PERCENT - swapFeePercent)) + state.feeConfig.fragmentationFee * ((PERCENT + ratePerTenor) / PERCENT)

100 * ((100 + 10) / (100 - 10)) + 5 * ((100 + 10) / (100)) = 127.72222222222223
```

Not we arrive at a rate of exactly 10% for Bob.

## Tools Used

Manual review

## Recommended Mitigation Steps

Update the formula such that the fragmentation fee is added correctly.

## Assessed type

Math

# L-02 LiquidateWithReplacement benefits the protocol at the expense of lenders

`LiquidateByReplacement` pays a `liquidatorProfitBorrowToken` fee to the protocol. This fee is essentially carried by lenders in the form of opportunity cost.

In a normal liquidation of an underwater borrower, lenders can claim the full `futureAmount` at the time of liquidation, and earn interest by re-lending those funds immediately, rather than having to wait for the due date. A regular liquidation also reduces risk for the lenders compared to `LiquidateByReplacement`.

This design creates a "lenders vs. the protocol" situation, where lenders are incentivized to always liquidate underwater positions the conventional way rather than allowing `executeLiquidateWithReplacement` to be triggered.

This is a fairness/design issue rather than a 'vulnerability' but probably still useful to consider so I'm submitting this as a QA report.

## Proof of Concept

This issue doesn't require a proof-of-concept. 

## Tools Used

Manual review

## Recommended Mitigation Steps

Remove the `executeLiquidateWithReplacement` functionality.

# L-03 Liquidation collateral ratio can never be increased due to incorrect input validation

In `executeUpdateConfig`, there is a check that prevents `crLiquidation` from exceeding its current value.

```
        } else if (Strings.equal(params.key, "crLiquidation")) {
            if (params.value >= state.riskConfig.crLiquidation) {                
              revert Errors.INVALID_COLLATERAL_RATIO(params.value);
            }
```

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/UpdateConfig.sol#L87-L93

This appears to be a mistake, assuming that the intent here was to prevent `crLiquidation` from exceeding `crOpening`.

## Proof of Concept

I did not create a proof-of-concept since the issue is trivial.

## Tools Used

Manual review

## Recommended Mitigation Steps

 The correct comparison would be:

```
        } else if (Strings.equal(params.key, "crLiquidation")) {
            if (params.value >= state.riskConfig.crOpening) {
```

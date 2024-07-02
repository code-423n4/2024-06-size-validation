## [L-01] min and maxAnswer never checked for oracle price feed
Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This would allow user to continue borrowing with the asset but at the wrong price. This is exactly what happened to Venus on BSC when LUNA imploded. However, the protocol misses to implement such a check.

[Link to code:](https://github.com/code-423n4/2024-06-size/blob/main/src/oracle/PriceFeed.sol#L84-L93)
```
function _getPrice(AggregatorV3Interface aggregator, uint256 stalePriceInterval) internal view returns (uint256) {
    // slither-disable-next-line unused-return
    (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();

    if (price <= 0) revert Errors.INVALID_PRICE(address(aggregator), price);
    if (block.timestamp - updatedAt > stalePriceInterval) {
        revert Errors.STALE_PRICE(address(aggregator), updatedAt);
    }

    return SafeCast.toUint256(price);
}
```

**Fix:**
Add logic along the lines of:
```
require(answer >= minPrice && answer <= maxPrice, "invalid price");
```
min and max prices can be gathered using one of [these ways](https://medium.com/cyfrin/chainlink-oracle-defi-attacks-93b6cb6541bf#99af:~:text=Developers%20%26%20Auditors%20can%20find%20Chainlink%E2%80%99s%20oracle%20feed).


## [L-02] crLiquidation can only be updated to lower values.
[When `crLiquidation` is updated](https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/UpdateConfig.sol#L89-L93), the old value (and any other value greather than the old value) can be set again. This causes that `crLiquidation` can only be updated to lower and lower values.

```
function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {
        ...
        //@audit-issue => The new value for `crLiquidation` can never be greather than the existing value.
        //@audit-issue => After the `crLiquidation` has been updated, the old value (and any other value > than that) would not be possible to be set again.
            if (params.value >= state.riskConfig.crLiquidation) {
                revert Errors.INVALID_COLLATERAL_RATIO(params.value);
            }
            state.riskConfig.crLiquidation = params.value;
        ...
```

**Fix:**
Instead of comparing against the current value, compare against an upper limit that can't never be surpassed, i.e. `>= crOpening`
- The first upper limit for the `crLiquidation` is the `crOpening` because crOpening prevents new loans from being created close to liquidation.


## [L-03] Incorrect operator to bound the Grade Period of the Sequencer
The [current operator `<=` will rever when the duration time is equals to the GRACE_TIME](https://github.com/code-423n4/2024-06-size/blob/main/src/oracle/PriceFeed.sol#L73-L76), which, it should not, once the grace time is reached, the execution should be allowed to continue because the Sequencer has been up for the minimum amount of time defined by the GRACE_PERIOD

**Fix:**
- Use the `<` operator, instead of `<=`
```
function getPrice() external view returns (uint256) {
    if (address(sequencerUptimeFeed) != address(0)) {
        ...

-       if (block.timestamp - startedAt <= GRACE_PERIOD_TIME) {
+       if (block.timestamp - startedAt < GRACE_PERIOD_TIME) {
        }
    }

    ...
}
```

## [L-04] Not validating `liquidatorProfitBorrowToken` when doing a Liquidation with Replacement
As per the[ inline documentation about the `liquidatorProfitBorrowToken` parameter](https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/LiquidateWithReplacement.sol#L119), it represents the profit in borrow tokens expected by the liquidator. Not validating this value prevents the liquidators from ensuring they are receiving the expected amount of profit.

**Fix:**
- This is either an old comment for that parameter that may not longer be applicable, but, if this parameter is indeed intended for the liquidator to ensure a specific profit, then, add a validation to allow the liquidator to specify the desired amount of profit in borrow tokens.

## [L-05] Not allowing repayments while the protocol is paused
[Loans could go overdue while the protocol is paused](https://github.com/code-423n4/2024-06-size/blob/main/src/Size.sol#L198), causing a race between liquidators and repayers to liquidate/repay the debt.
- If liquidator's tx is executed first, the borrower will be affected because it would need to pay the liquidator's reward from its collateral.

**Fix:**
- Allow repayments while the protocol is paused.
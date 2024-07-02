# L-01 Incorrect assumption of the stale price interval in L2 networks.

https://data.chain.link/feeds/arbitrum/mainnet/eth-usd

# L-02 Do not allow to call multicall during a multicall


While this inconsistency cannot be exploited in the current codebase a future upgrade mechanism which relies on `state.isMulticall` can become exploitable.

# L-03 Several Events are emitted wrongly:

1. In `executeCompensate` the following event is emitted:

```solidity
emit Events.Compensate(params.creditPositionWithDebtToRepayId, params.creditPositionToCompensateId, params.amount);
```
However, `params.amount` is incorrect as this variable could change during the code execution due to the execution state of the credit positions involved in the operation. Take for example the following code from the same function:

```solidity
uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);
```

2. At the beginning of the function `executeSellCreditMarket` the event `SellCreditMarket` is emitted:

```solidity
emit Events.SellCreditMarket(params.lender, params.creditPositionId, params.tenor, params.amount, params.tenor, params.exactAmountIn);
```

However, two issues exists in this event:

- The 3rd field which is passed as `params.tenor` is not necessarily the `tenor` at which the order is executed this comes from the fact that in case an existing `creditPosition` is being sold, then `tenor` is recomputed as `debtPosition.dueDate - block.timestamp`.

- The 5th field which is passed as `params.tenor` should be `dueDate` as per the definition in the `Events.sol` file. 

3. At the beginning of the `executeBuyCreditMarket` function the code emits the following event:

```solidity
emit Events.BuyCreditMarket(params.borrower, params.creditPositionId, params.tenor, params.amount, params.exactAmountIn);
```

However, there is no guarantee that `params.borrower` is the underlying borrower for `params.creditPositionId`. This comes from the fact that when `creditPositionId != RESERVED_ID` then the borrower is derived from the current state instead of using `params.borrower`.

# L-04 ratePerTenor is a better slippage mechanism rather than just the apr

Take the cause `dueToDate - block.timestamp`

# L-05 Cash receiver is not paying swap fee in `LiquidateWithReplacement`

The new borrower in `LiquidateWithReplacement` is being transferred the `issuanceValue` computed as follows:

```solidity
issuanceValue = Math.mulDivDown(debtPositionCopy.futureValue, PERCENT, PERCENT + ratePerTenor);
```

It is being computed using the `futureValue` which represents the debt that the new borrower is accruing. However, this new borrower is not paying for the swap fee even though the borrower is receiving cash.


# L-06 A malicious user can fragment a credit position without the paying fragmentation fee via compensate

With the `compensate` mechanism a user can go from the debt position having only one credit position to the debt position having several credit positions as the minimum credit configuration allows. This is achieved by using the `RESERVED_ID` as the `creditPositionToCompensateId` parameter.

Therefore, the Size's bots will need to call `claim` for each of the created credit positions without receiving a fragmentation fee. 

# L-07 Is it possible to create a credit position with `tenor < minTenor`

# L-08 Some `buyCreditMarket` and `sellCreditMarket` calls can fail due to an incorrect comparison.

```solidity
 if (params.amount < state.riskConfig.minimumCreditBorrowAToken) {
         revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);
}
```
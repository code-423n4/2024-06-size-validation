# L-01 Incorrect assumption of the stale price interval in L2 networks.

https://data.chain.link/feeds/arbitrum/mainnet/eth-usd

# L-02 Do not allow to call multicall during a multicall

# L-03 A malicious user can make `executeBuyCreditMarket` to emit an incorrect event

At the beginning of the `executeBuyCreditMarket` function the code emits the following event:

```solidity
emit Events.BuyCreditMarket(params.borrower, params.creditPositionId, params.tenor, params.amount, params.exactAmountIn);
```

# L-04 ratePerTenor is a better slippage mechanism rather than just the apr

Take the cause `dueToDate - block.timestamp`


# L-05 The `executeSellCreditMarket` function emits an incorrect event

At the beginning of the function `executeSellCreditMarket` the event `SellCreditMarket` is emitted:

```solidity
emit Events.SellCreditMarket(params.lender, params.creditPositionId, params.tenor, params.amount, params.tenor, params.exactAmountIn);
```

However, two issues exists in this event:

- The 3rd field which is passed as `params.tenor` is not necessarily the `tenor` at which the order is executed this comes from the fact that in case an existing `creditPosition` is being sold, then `tenor` is recomputed as `debtPosition.dueDate - block.timestamp`.

- The 5th field which is passed as `params.tenor` should be `dueDate` as per the definition in the `Events.sol` file. 
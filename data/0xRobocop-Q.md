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
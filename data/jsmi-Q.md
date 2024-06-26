The `SellCreditMarket` event is delared as follows
```solidity
    event SellCreditMarket(
        address indexed lender,
        uint256 indexed creditPositionId,
        uint256 indexed tenor,
        uint256 amount,
        uint256 dueDate,
        bool exactAmountIn
    );
```
But is called like this.
```solidity
        emit Events.SellCreditMarket(
            params.lender, params.creditPositionId, params.tenor, params.amount, params.tenor, params.exactAmountIn
        );
```
As shown above, `params.tenor` is used for `dueDate`.
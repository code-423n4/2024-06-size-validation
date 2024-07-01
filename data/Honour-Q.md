## [QA-1] Events can emit wrong data, eg in buycreditMarket

## Impact

incorrect event data can severly impact protocol performance on the frontend

## Proof of Concept

From the protocol design, one can see that the protocol data used by the frontend is heavily reliant on the data emitted from events,however in many cases it's possible for events to emit wrong data about an operation, one example of this is in `buyCreditmarket` where the `params.borrower` and `params.tenor` parameter are ignored when buying credit from an existing credit position.A malicious user could purposefully fill both parameters with wrong values ,emitting wrong data to the frontend whilst their transaction executes as expected as such values are ignored.
![image](https://gist.github.com/assets/133224432/899699f3-c27a-4600-85db-d90e89d97720)

note: asides `buyCreditMarket` there are many such instances of this.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Events should only be emitted after all necessarry parameters have been assigned

```diff

function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
        external
        returns (uint256 cashAmountIn)
    {
-       emit Events.BuyCreditMarket(
-           params.borrower, params.creditPositionId, params.tenor, params.amount, params.exactAmountIn
-       );

        // slither-disable-next-line uninitialized-local
        CreditPosition memory creditPosition;
        uint256 tenor;
        address borrower;
        if (params.creditPositionId == RESERVED_ID) {
            borrower = params.borrower;
            tenor = params.tenor;
        } else {
            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);
            creditPosition = state.getCreditPosition(params.creditPositionId);

            borrower = creditPosition.lender;
            tenor = debtPosition.dueDate - block.timestamp;
        }
+       emit Events.BuyCreditMarket(
+           borrower, params.creditPositionId, tenor, params.amount, params.exactAmountIn
+       );

        //...code omitted for brevity...
    }
```

## [QA-2] Users cannot deposit both ETH and Usdc in a single multicall

## Impact

Complex batch operations that require depositing both ETH and USDC are impossible.

## Proof of Concept

In `validateDeposit` the msg.value is [asserted first](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Deposit.sol#L40-L43) causing an early revert if there's a msg.value but the `params.token` is not weth. Since a msg.value persists through the multicall , any multicall operation that contains a msg.value will revert if it attempts to deposit USDC. an example operation might be, a liquidator wanting to take out a loan using the collateral from the liquidation

1. deposit usdc for liquidation
2. liquidate loan
3. deposit ETH to top off collateralToken recieved from liquidation
4. take out a loan using the collateral

This operation reverts at 1 due to the presence of msg.value, the obvious workaround would be to wrap the ETH or deposit the USDC before the multicall, however both cases require 2 transactions.

## Tools Used

Manual Review

## Recommended Mitigation Steps

```diff

    function validateDeposit(State storage state, DepositParams calldata params) external view {
        // validate msg.sender
        // N/A

        // validate msg.value
-       if (msg.value != 0 && (msg.value != params.amount || params.token != address(state.data.weth))) {
-           revert Errors.INVALID_MSG_VALUE(msg.value);
-       }

        //if token is not eth we shouldn't care about msg.value
+       if (params.token == state.data.weth && msg.value != 0 && msg.value != params.amount) {
+           revert Errors.INVALID_MSG_VALUE(msg.value);
+       }

        // validate token
        if (
            params.token != address(state.data.underlyingCollateralToken)
                && params.token != address(state.data.underlyingBorrowToken)
        ) {
            revert Errors.INVALID_TOKEN(params.token);
        }

        // validate amount
        if (params.amount == 0) {
            revert Errors.NULL_AMOUNT();
        }

        // validate to
        if (params.to == address(0)) {
            revert Errors.NULL_ADDRESS();
        }
    }

    function executeDeposit(State storage state, DepositParams calldata params) public {
-       address from = msg.sender;
-       uint256 amount = params.amount;
-       if (msg.value > 0) {
-           // do not trust msg.value (see `Multicall.sol`)
-           amount = address(this).balance;
-           // slither-disable-next-line arbitrary-send-eth
-           state.data.weth.deposit{value: amount}();
-           state.data.weth.forceApprove(address(this), amount);
-           from = address(this);
-       }

        if (params.token == address(state.data.underlyingBorrowToken)) {
-           state.depositUnderlyingBorrowTokenToVariablePool(from, params.to, amount);
+           state.depositUnderlyingBorrowTokenToVariablePool(msg.sender, params.to, params.amount);
            // borrow aToken cap is not validated in multicall,
            //   since users must be able to deposit more tokens to repay debt
            if (!state.data.isMulticall) {
                state.validateBorrowATokenCap();
            }
        } else {
+           address from = msg.sender;
+           uint256 amount = params.amount;
+           if (msg.value > 0) {
+               // do not trust msg.value (see `Multicall.sol`)
+               amount = address(this).balance;
+               // slither-disable-next-line arbitrary-send-eth
+               state.data.weth.deposit{value: amount}();
+               state.data.weth.forceApprove(address(this), amount);
+               from = address(this);
+           }
            state.depositUnderlyingCollateralToken(from, params.to, amount);
        }

        emit Events.Deposit(params.token, params.to, amount);
    }
```

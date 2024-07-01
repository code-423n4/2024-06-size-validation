ERC20 functions may not behave as expected. For example: return values are not always meaningful. It is recommended to use OpenZeppelin's SafeERC20 library.

- Found in src/libraries/actions/BuyCreditMarket.sol [Line: 195](src/libraries/actions/BuyCreditMarket.sol#L195)

  ```solidity
          state.data.borrowAToken.transferFrom(msg.sender, borrower, cashAmountIn - fees);
  ```

- Found in src/libraries/actions/BuyCreditMarket.sol [Line: 196](src/libraries/actions/BuyCreditMarket.sol#L196)

  ```solidity
          state.data.borrowAToken.transferFrom(msg.sender, state.feeConfig.feeRecipient, fees);
  ```

- Found in src/libraries/actions/Claim.sol [Line: 56](src/libraries/actions/Claim.sol#L56)

  ```solidity
          state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);
  ```

- Found in src/libraries/actions/Compensate.sol [Line: 152](src/libraries/actions/Compensate.sol#L152)

  ```solidity
              state.data.collateralToken.transferFrom(
  ```

- Found in src/libraries/actions/Liquidate.sol [Line: 118](src/libraries/actions/Liquidate.sol#L118)

  ```solidity
          state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);
  ```

- Found in src/libraries/actions/Liquidate.sol [Line: 119](src/libraries/actions/Liquidate.sol#L119)

  ```solidity
          state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
  ```

- Found in src/libraries/actions/Liquidate.sol [Line: 120](src/libraries/actions/Liquidate.sol#L120)

  ```solidity
          state.data.collateralToken.transferFrom(
  ```

- Found in src/libraries/actions/LiquidateWithReplacement.sol [Line: 161](src/libraries/actions/LiquidateWithReplacement.sol#L161)

  ```solidity
          state.data.borrowAToken.transferFrom(address(this), params.borrower, issuanceValue);
  ```

- Found in src/libraries/actions/LiquidateWithReplacement.sol [Line: 162](src/libraries/actions/LiquidateWithReplacement.sol#L162)

  ```solidity
          state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);
  ```

- Found in src/libraries/actions/Repay.sol [Line: 49](src/libraries/actions/Repay.sol#L49)

  ```solidity
          state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);
  ```

- Found in src/libraries/actions/SelfLiquidate.sol [Line: 70](src/libraries/actions/SelfLiquidate.sol#L70)

  ```solidity
          state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);
  ```

- Found in src/libraries/actions/SellCreditMarket.sol [Line: 201](src/libraries/actions/SellCreditMarket.sol#L201)

  ```solidity
          state.data.borrowAToken.transferFrom(params.lender, msg.sender, cashAmountOut);
  ```

- Found in src/libraries/actions/SellCreditMarket.sol [Line: 202](src/libraries/actions/SellCreditMarket.sol#L202)

  ```solidity
          state.data.borrowAToken.transferFrom(params.lender, state.feeConfig.feeRecipient, fees);
  ```

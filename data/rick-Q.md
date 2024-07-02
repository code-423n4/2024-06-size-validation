# Audit Report: Liquidator cannot achieve proper liquidator reward as protocol promised

## Lines of code

https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditMarket.sol#L91-L93
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/SellCreditMarket.sol#L93-L95

## Root cause and summary

In `validateBuyCreditMarket` and `validateSellCreditMarket`, It is designed to validated at least the minimum credit amount specified by the protocol. If a order is made for less than the protocol's minimum credit amount, the error `Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT` will be reverted.

However, user can execute the order in cash instead of credit setting `exactAmountIn` in the params. In `BuyCreditMakret`, the user sets `exactAmountIn` to true and `SellCreditMarket` to false to execute the trade for the desired cash. In this case, it will revert the error even though the credit that will actually be executed is not less than `state.riskConfig.minimumCreditBorrowAToken`. That means `params.amount` can be cash, depending on the user's intent, and comparing it directly to minimum credit is a problem.

## Proof of Concept

At the test below, minCashAmountIn is the amount whose future value is 5000000 depending on the limit order and tenor.
Since the credit is not less than the minimu credit of 500000 set by the protocol, it should run without issue, but if you run the test code below, you'll see that it throws an error.

```solidity
function test_BuyCreditMarket_NotMinimumCredit_validation() public {
        _deposit(candy, usdc, 100e6);
        _deposit(james, weth, 100e18);
        _sellCreditLimit(james, 0.03e18, 365 days);

        uint256 deadline = block.timestamp;
        uint256 tenor = 365 days;
        bool exactAmountIn = true;

        vm.startPrank(candy);
        uint256 minCashAmountIn = 4854369; // the future value for the limit order and the tenor is 5000000, the minimum credit.

        size.buyCreditMarket(
            BuyCreditMarketParams({
                borrower: james,
                creditPositionId: RESERVED_ID,
                amount: minCashAmountIn,
                tenor: tenor,
                deadline: deadline,
                minAPR: 0,
                exactAmountIn: exactAmountIn
            })
        );
        vm.stopPrank();
    }
```

```
function test_SellCreditMarket_NotMinimumCredit_validation() public {
      _deposit(candy, weth, 100e18);
      _deposit(james, usdc, 100e6);
      _buyCreditLimit(
          james, block.timestamp + 365 days, [int256(0.03e18), int256(0.03e18)], [uint256(10 days), uint256(365 days)]
      );

      uint256 deadline = block.timestamp;
      uint256 tenor = 365 days;
      bool exactAmountIn = false;

      vm.startPrank(candy);
      uint256 ratePerTenor = size.getLoanOfferAPR(james, tenor);
      uint256 minCashAmountIn = 4830097; // the future value for the limit order and the tenor is 5000000, the minimum credit.

      size.sellCreditMarket(
          SellCreditMarketParams({
              lender: james,
              creditPositionId: RESERVED_ID,
              amount: minCashAmountIn,
              tenor: 365 days,
              deadline: deadline,
              maxAPR: type(uint256).max,
              exactAmountIn: exactAmountIn
          })
      );
      vm.stopPrank();
  }
```

## Recommended Mitigation Steps

As with the execute part of the trade, the value of `params.exactAmountIn` should be used to determine the actual credit that will be executed and then compared to the minimum credit set by the protocol to ensure that the original intent is met. However, it is also clever to modify the intent of the protocol with error statements, as this process once again uses `ratePerTenor` and `AccountingLibrary`'s `getCreditAmountOut` and `getCreditAmountIn`.

```
# BuyCreditMakret.sol
if (params.amount < state.riskConfig.minimumCreditBorrowAToken) {
            if (!params.exactAmountIn) {
                revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);
            }
            else {
                CreditPosition memory creditPosition;
                uint256 ratePerTenor = state.data.users[borrower].borrowOffer.getRatePerTenor(
                    VariablePoolBorrowRateParams({
                        variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
                        variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
                        variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
                    }),
                    tenor
                );
                (uint256 creditAmountOut, uint256 fees) = state.getCreditAmountOut({
                    cashAmountIn: params.amount,
                    maxCashAmountIn: params.creditPositionId == RESERVED_ID
                        ? params.amount
                        : Math.mulDivUp(creditPosition.credit, PERCENT, PERCENT + ratePerTenor),
                    maxCredit: params.creditPositionId == RESERVED_ID
                        ? Math.mulDivDown(params.amount, PERCENT + ratePerTenor, PERCENT)
                        : creditPosition.credit,
                    ratePerTenor: ratePerTenor,
                    tenor: tenor
                });
                if (creditAmountOut < state.riskConfig.minimumCreditBorrowAToken) {
                    revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(creditAmountOut, state.riskConfig.minimumCreditBorrowAToken);
                }
            }
        }
```


```
# SellCreditMakret.sol
if (params.amount < state.riskConfig.minimumCreditBorrowAToken) {
            if (params.exactAmountIn) {
                revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);
            }
            else {
                CreditPosition memory creditPosition;
                uint256 ratePerTenor = state.data.users[params.lender].loanOffer.getRatePerTenor(
                    VariablePoolBorrowRateParams({
                        variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
                        variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
                        variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
                    }),
                    tenor
                );
                (uint256 creditAmountIn, uint256 fees) = state.getCreditAmountIn({
                    cashAmountOut: params.amount,
                    maxCashAmountOut: params.creditPositionId == RESERVED_ID
                        ? params.amount
                        : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),
                    maxCredit: params.creditPositionId == RESERVED_ID
                        ? Math.mulDivUp(params.amount, PERCENT + ratePerTenor, PERCENT - state.getSwapFeePercent(tenor))
                        : creditPosition.credit,
                    ratePerTenor: ratePerTenor,
                    tenor: tenor
                });
                if (creditAmountIn < state.riskConfig.minimumCreditBorrowAToken) {
                    revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(creditAmountIn, state.riskConfig.minimumCreditBorrowAToken);
                }
            }
        }
```

you can check the detail in https://github.com/Decipher100Eyes/code4rena-size/tree/CreditMarket/LowerThanMinimumCredit
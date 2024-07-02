### [Low-0] As Referral Program Of Aave Currently Inactive, So In `variablePool.supply(_, _, _, 0). Here 0 Passed But This Refferal Program Could Activated In Future, In that Case SIZE Will Not Able To Take Advantage Of This.

Aave Developer documentation have following 
`Referral supply is currently inactive, you can pass 0 as referralCode. This program may be activated in the future through an Aave governance proposal`

So This can active in future, in that case SIZE will not able to Use this feature due to hardcoded value.

```solidity 
  state.data.underlyingBorrowToken.forceApprove(address(state.data.variablePool), amount);
        state.data.variablePool.supply(address(state.data.underlyingBorrowToken), amount, address(this), 0);
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/DepositTokenLibrary.sol#L60

#### Mitigation
May be Use some state variable for `referral code` and change it when ever its get changed(activated) in aave. 




### [Low-0] AnyOne Can Create A New Loan Offer / New Borrow Offer, Irrespective they Have Deposited Collateral Or Not.
There is no validation that User creating loan offer has collateral. Although they will not receive loan as in `buyCreditMarket()` function `validateVariablePoolHasEnoughLiquidity()` will failed.

But Point is Many Users could create those type of Offers and populate whole system.

```solidity
    function validateSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external view {
        BorrowOffer memory borrowOffer = BorrowOffer({curveRelativeTime: params.curveRelativeTime});

        // a null offer mean clearing their limit order
        if (!borrowOffer.isNull()) {
            // validate msg.sender
            // N/A

            // validate curveRelativeTime
            YieldCurveLibrary.validateYieldCurve(
                params.curveRelativeTime, state.riskConfig.minTenor, state.riskConfig.maxTenor
            );
        }
    }
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/SellCreditLimit.sol#L25-L38

#### Mitigation
Simple mitigation is to check for collateral before User create loan offer 



### [Low-0] Recidual Amount Of Token Remain In Conract Due To `Rounding`



### [Low-0] By Default A Newly Created CreditPosition Is Set To `For Sale = True` Which Could Problematic

If user's didn't configured eairlier, then may be his Buyed credit could be taken by other lenders without his intention, as by default CreditPosition's forSale parameter set as TRUE.

As for below code segment from `BuyCreditMarket.sol`'s `validateBuyCreditMarket()` if no configuration happen for User then `user.allCreditPositionsForSaleDisabled = false` so in that case this check get passed.

```solidity
            if (user.allCreditPositionsForSaleDisabled || !creditPosition.forSale) {
                revert Errors.CREDIT_NOT_FOR_SALE(params.creditPositionId);
            }
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditMarket.sol#L75-L77

#### Miigation
Its make compulsory for User to set configuration, or newly created credit position should created with `forSale = false` have feature to change its later. 

### [Low-0] Chainlink Price Feed Have Improper Validation



### [Low-0] PriceFeed Will Use The Wrong Price If The Chainlink Registry Returns Price Outside min/max Range 

Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This would allow user to continue borrowing with the asset but at the wrong price. This is exactly what happened to [Venus on BSC when LUNA imploded](https://rekt.news/venus-blizz-rekt/).

aggregator#latestRoundData pulls the associated aggregator and requests round data from it. ChainlinkAggregators have minPrice and maxPrice circuit breakers built into them. This means that if the price of the asset drops below the minPrice, the protocol will continue to value the token at minPrice instead of it's actual value. 

Example: TokenA has a minPrice of $1. The price of TokenA drops to $0.10. The aggregator still returns $1 

```diff
    (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();
    
+   if (price >= maxPrice or price <= minPrice) revert();
```
https://github.com/code-423n4/2024-06-size/blob/main/src/oracle/PriceFeed.sol#L86

#### Mitigation
Mentioned in above code segment


### [Low-0] Function May Be Trying To transfer 0 amount



### [Low-0] RatePerTenor



### [Low-0] Many NatSpec Comments Are Wrong



### [Low-0] Repay Should Allowed In Paused Event Period or User Should Allowed Some Grace Period After Unpause To Supply Collateral And Escape From UnWanted Liquidation. 
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




### [Low-0] `params.minAPR` checks in `validateBuyCreditMarket()` is not true with actual senario.

```solidity
        // validate minAPR
        uint256 apr = borrowOffer.getAPRByTenor(
            VariablePoolBorrowRateParams({
                variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
                variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
                variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
            }),
            tenor
        );
        if (apr < params.minAPR) {
            revert Errors.APR_LOWER_THAN_MIN_APR(apr, params.minAPR);
        }
```

For let say (taking an example from official document)
A Borrow has Borrow Offer of 5% APR for 1YEAR
when lender wants to lend he calls `buyCreditmarket()` with params.minApr = 5%

This `params.minApr` get checked with `borrowOffer.getAPRByTenor()` which return 5% in above simple example.

Then in function pointer moves to execute part

In `executeBuyCreditMarket()`





### [Low-0] Recidual Amount Of Token Remain In Conract Due To `Rounding`
SIZE implementing different calculation approch than AaveV3.

I tested above things in HardHat By forking mainnet

So summery at testing
Deposit to aave pool 100USDC
Then checked my account's
- scaledBalance (`aToken.scaledBalanceOf(deployer.address)`) : 92956033
- balanceOf (`aToken.balanceOf(deployer.address)`) : 100000000
- liquidityIndexAt (`lendingPool.getReserveNormalizedIncome(USDC_ADDRESS)`) : 1075777402162638815192807739

Then i go for withdraw
After withdraw my account's balances now
- USDC balance : 100000000
- scaledBalance (`aToken.scaledBalanceOf(deployer.address)`) : 0
- balanceOf (`aToken.balanceOf(deployer.address)`) : 0

Same things i done but via size
`NOTE : here i use hardhat for aave, and size's Math.sol on Remix to get exact answer for calculation`

Deposit 100USDC to size (Considering first Deposit to SIZE and followed by first withdraw from SIZE)
- scaledBalanceBefore : 0
- SIZE's scaledBalanceAfter : 92956033
- SIZE's balanceOf : 100000000
- liquidityIndexAt (`lendingPool.getReserveNormalizedIncome(USDC_ADDRESS)`) : 1075777402162638815192807739
- scaledAmount : scaledBalanceAfter - scaledBalanceBefore =  92956033

Now minting of borrowAToken happens for USER
```solidity
state.data.borrowAToken.mintScaled(to, scaledAmount);
```

- USER's borrowATokens.scaledBalanceOf() : 
- USER's borrowATokens.balanceOf() :


### [Low-0] Due to Rounding Down Protocol may loss some reward fee

In `executeLiquidate()` there is below code segment
```solidity
            // cap the collateral remainder to the liquidation collateral ratio
            //   otherwise, the split for non-underwater overdue loans could be too much
            uint256 collateralRemainderCap =
                Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);

            collateralRemainder = Math.min(collateralRemainder, collateralRemainderCap);

            protocolProfitCollateralToken = Math.mulDivDown(collateralRemainder, collateralProtocolPercent, PERCENT);
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L107-L112

where `protocolProfitCollateralToken` calculated on Profitable liquidation where `assignCollateral > debtInCollateral` and `protocolProfitCollateralToken` amount of token taken by protocol fee receipient.

Here we can see Double Rounding Down occurs in calculation.

first Rounding Down happens in calculation of Cap variable `collateralRemainderCap`

then let say `collateralRemainderCap` is minimum b/w `collateralRemainder & collateralRemainderCap` so collateralRemainder = collateralRemainderCap

And then second rounding down happens in `protocolProfitCollateralToken` calculation

#### Mitigation
its a standard that fee taken by protocol should always in favour of Protocol and always Rounding Up.



### [Low-0] `weth.forceApprove()` in `executeDeposit()` is not relevant

```solidity
  function executeDeposit(State storage state, DepositParams calldata params) public {
        address from = msg.sender;
        uint256 amount = params.amount;
        if (msg.value > 0) {
            // do not trust msg.value (see `Multicall.sol`)
            amount = address(this).balance;
            // slither-disable-next-line arbitrary-send-eth
            state.data.weth.deposit{value: amount}();
            state.data.weth.forceApprove(address(this), amount);
            from = address(this);
        }
....
....
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Deposit.sol#L67-L73

If we carefully observe here is that caller to weth i.e `address(this)` forceApproving same `address(this)` with amount. then further from overrided with address this

when actual token trasfer happening 
```solidity
    function depositUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount) external {
        IERC20Metadata underlyingCollateralToken = IERC20Metadata(state.data.underlyingCollateralToken);
        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);
        state.data.collateralToken.mint(to, amount);
    }
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/DepositTokenLibrary.sol#L24-L26

here when underlyingCollateralToken's transfer happening through safeTransferFrom
from = address(this)
to = address(this)

so basically self token transfering happen here. I don't see any use case here.

#### Mitigation
Re-configure function logic.



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

### [Low-0] Chainlink Price Feed Have some missing checks for Validation
```diff
-       (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();
+       (uint80 roundId, int256 price,, uint256 updatedAt, uint80 answeredInRound) = aggregator.latestRoundData();

        if (price <= 0) revert Errors.INVALID_PRICE(address(aggregator), price);
        if (block.timestamp - updatedAt > stalePriceInterval) {
            revert Errors.STALE_PRICE(address(aggregator), updatedAt);
        }
+       require(answeredInRound >= roundId, "price is stale");
+       require(updatedAt > 0, "round is incomplete");
```


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

Below we can see that `protocolProfitCollateralToken` could be 0, if the liquidation is not profitable

so in that case `state.feeConfig.feeRecipient` transfered with 0 amount.
```solidity
        uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);
        uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);
        uint256 protocolProfitCollateralToken = 0;
        // profitable liquidation
        if (assignedCollateral > debtInCollateralToken) {
 ...
...
            uint256 collateralRemainderCap =
                Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);

            collateralRemainder = Math.min(collateralRemainder, collateralRemainderCap);

            protocolProfitCollateralToken = Math.mulDivDown(collateralRemainder, collateralProtocolPercent, PERCENT);
        } else {
            // unprofitable liquidation
            liquidatorProfitCollateralToken = assignedCollateral;
        }

        state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);
        state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
+       if (protocolProfitCollateralToken !=0){
        state.data.collateralToken.transferFrom(
            debtPosition.borrower, state.feeConfig.feeRecipient, protocolProfitCollateralToken
        )};
```
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L92-L122

#### Mitigation
Only transfer token when transfered amount is non-zero as showed above 




### [Low-0] Many NatSpec Comments Are Wrong



### [Low-0] Repay Should Allowed In Paused Event Period or User Should Allowed Some Grace Period After Unpause To Supply Collateral And Escape From UnWanted Liquidation. 

`ISizeAdmin.sol` have some admin functions like 
```solidity
    /// @notice Pauses the protocol
    ///         Only callabe by the DEFAULT_ADMIN_ROLE
    function pause() external;

    /// @notice Unpauses the protocol
    ///         Only callabe by the DEFAULT_ADMIN_ROLE
    function unpause() external;
}
```
https://github.com/code-423n4/2024-06-size/blob/main/src/interfaces/ISizeAdmin.sol#L25-L32

So basically whole system goes into pause/unpause mode when this functions called including some crucial functions like `repay` `deposit` etc

if in this duration, some major market correction happens / market dip appears then Collateral ratios of Users could fall below liquidation and they subject to liquidation, (and User can do nothing about it, nither they increase their CR via depositing more underlayingCollateral token, no settle their loans)

On very next moment profitable liqudit position get liqudated by liquidators or Bots.

#### Mitigation
To Deal with this type of situation Protocol should allow some functions to operate even Pause mode Or should give some grace period to Borrowers to take decision to To rise their CRs above thresold or wish to getliquidated.
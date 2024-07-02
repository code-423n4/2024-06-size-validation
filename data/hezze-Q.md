**Description:** Despite all the checks done to guarantee a buyCreditLimit order is valid, an invalid buyCreditLimit order can be created by specifying a maxDueDate lower than the tenors specified in the yield curve of the order. This is because of the use of linear interpolation to find the apr of the tenor. The linear interpolation requires two knowns to identify an unknown (i.e in this case the APR);

**Impact:** Borrowers will be unable to match the buyCreditLimit order. 

**Proof of Concept:**

```solidity
  function test_BuyCreditLimit_Invalid_buyCreditLimit() public {
        assertTrue(_state().alice.user.loanOffer.isNull());

        uint256 maxDueDate = block.timestamp + 20 days;
        uint256[] memory marketRateMultipliers = new uint256[](2);
        uint256[] memory tenors = new uint256[](2);
        tenors[0] = 30 days;
        tenors[1] = 60 days;
        int256[] memory aprs = new int256[](2);
        aprs[0] = 0.15e18;
        aprs[1] = 0.12e18;

        vm.prank(alice);

        size.buyCreditLimit(
            BuyCreditLimitParams({
                maxDueDate: maxDueDate,
                curveRelativeTime: YieldCurve({tenors: tenors, marketRateMultipliers: marketRateMultipliers, aprs: aprs})
            })
        );
        assertTrue(!_state().alice.user.loanOffer.isNull());
        _deposit(alice, usdc, 300e6);
        _deposit(bob, weth, 1e18);

        vm.expectRevert();
        vm.prank(bob);
        size.sellCreditMarket(
            SellCreditMarketParams({
                lender: alice,
                creditPositionId: RESERVED_ID,
                amount: 100e6,
                tenor: 11 days,
                deadline: block.timestamp,
                maxAPR: type(uint256).max,
                exactAmountIn: false
            })
        );
    }

```

**Recommended Mitigation:** Add checks to ensure the maxDueDate is within the tenors provided in the yield curve.
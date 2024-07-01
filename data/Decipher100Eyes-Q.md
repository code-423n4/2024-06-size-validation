# If lender make loanOffer with maxDudate < yieldCurve.tenors[0] + block.timestamp, every sellCreditLimit order for this loanOffer reverts

## Llines of code

https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditLimit.sol#L29-L51

## Root cause and summary

When borrowers sell credit by market order, they select a loanOffer and submit their desired tenor.
In case of secondary trade, this tenor is calculated with existing debtPosition due date.
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/SellCreditMarket.sol#L84

There are 2 main validations for this tenor.
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/SellCreditMarket.sol#L98-L100
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/YieldCurveLibrary.sol#L121C9-L123C10

In other words the tenor must meet the two conditions below.

1. loanOffer.yiledCurve.tenors[0] <= tenor <= loanOffer.yiledCurve.tenors[length - 1]
2. tenor <= loanOffer.maxDueDate - block.timestamp

However, if the loanOffer's maxDudate they chose is less than yieldCurve.tenors[0] + block.timestamp,
(`loanOffer.maxDuedate < yieldCurve.tenors[0] + block.timestamp`
=> `loanOffer.maxDuedate - block.timestamp < yiledCurve.tenors[0]`)
the minium tenor borrower can choose(yieldCurve.tenors[0]) always exceed maxDueDate - block.timestamp.
So this loan offer can not be matched to any sell credit market order forever.

## Proof of Concept

At the test below, loanOffer made with maxDudate < yieldCurve.tenors[0] + block.timestamp,
and executed sellCreditMarket for this loan offer with minimum tenor at the range of laon offer's curve.

```
function test_SellCreditMarket_revert_with_minimum_tenor() public {
    _setPrice(1e18);
    _deposit(alice, usdc, 1_000e6);
    _deposit(bob, weth, 300e18);

    uint256 maxDueDate = block.timestamp + 30 days;
    uint256[] memory marketRateMultipliers = new uint256[](2);
    uint256[] memory tenors = new uint256[](2);
    tenors[0] = 40 days;
    tenors[1] = 90 days;
    int256[] memory aprs = new int256[](2);
    aprs[0] = 0.12e18;
    aprs[1] = 0.15e18;

    vm.prank(alice);
    size.buyCreditLimit(
        BuyCreditLimitParams({
            maxDueDate: maxDueDate,
            curveRelativeTime: YieldCurve({tenors: tenors, marketRateMultipliers: marketRateMultipliers, aprs: aprs})
        })
    );

    uint256 tenor = 40 days;
    uint256 apr = size.getLoanOfferAPR(alice, tenor);
    size.sellCreditMarket(
        SellCreditMarketParams({
            lender: alice,
            creditPositionId: RESERVED_ID,
            amount: 100e6,
            tenor: tenor,
            deadline: block.timestamp,
            maxAPR: apr,
            exactAmountIn: false
        })
    );
}
```

This test reverts with the error of DUE_DATE_GREATER_THAN_MAX_DUE_DATE.

Also, the fuzzing test with random tenors bounded in loan offers yield curve always reverts.

```
function testFuzz_SellCreditMarket_tenors_between_yield_curve_bound(uint256 tenor) public {
    _setPrice(1e18);
    _deposit(alice, usdc, 1_000e6);
    _deposit(bob, weth, 300e18);

    uint256 maxDueDate = block.timestamp + 30 days;
    uint256[] memory marketRateMultipliers = new uint256[](2);
    uint256[] memory tenors = new uint256[](2);
    tenors[0] = 40 days;
    tenors[1] = 90 days;
    int256[] memory aprs = new int256[](2);
    aprs[0] = 0.12e18;
    aprs[1] = 0.15e18;

    vm.prank(alice);
    size.buyCreditLimit(
        BuyCreditLimitParams({
            maxDueDate: maxDueDate,
            curveRelativeTime: YieldCurve({tenors: tenors, marketRateMultipliers: marketRateMultipliers, aprs: aprs})
        })
    );

    tenor = bound(tenor, tenors[0], tenors[1]);
    uint256 apr = size.getLoanOfferAPR(alice, tenor);

    vm.expectRevert(
        abi.encodeWithSelector(
            Errors.DUE_DATE_GREATER_THAN_MAX_DUE_DATE.selector, block.timestamp + tenor, maxDueDate
        )
    );

    size.sellCreditMarket(
        SellCreditMarketParams({
            lender: alice,
            creditPositionId: RESERVED_ID,
            amount: 100e6,
            tenor: tenor,
            deadline: block.timestamp,
            maxAPR: apr,
            exactAmountIn: false
        })
    );
}
```

## Recommended Mitigation Steps

Validation logic like below should be added in https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditLimit.sol#L29-L51.

```
if (params.maxDueDate < block.timestamp + params.curveRelativeTime.tenors[0]) {
   revert Errors.INVALID_MAX_DUE_DATE_WITH_CURVE_MIN_TENOR(params.maxDueDate, params.curveRelativeTime.tenors[0]);
}
```

And if this validation added, exsiting validaion code below can be excluded.

```
if (params.maxDueDate < block.timestamp + state.riskConfig.minTenor) {
    revert Errors.PAST_MAX_DUE_DATE(params.maxDueDate);
}
```

(https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditLimit.sol#L42-L44)

Because 'params.curveRelativeTime.tenors[0] > state.riskConfig.minTenor' is validated by
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditLimit.sol#L47-L49
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/YieldCurveLibrary.sol#L69-L71,

Therefore, if the condition params.maxDueDate > block.timestamp + state.riskConfig.minTenor is met,
ensure params.maxDueDate < block.timestamp + state.riskConfig.minTenor.

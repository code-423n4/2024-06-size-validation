## Impact
Duplicated Tenor validate test leads to excessive gas consumption

## Root cause and summary

In BuyCreditMarket, SellCreditMarket, LiquidateWithReplacement, tenor validity tests are executed.
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/BuyCreditMarket.sol#L61-L63
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/SellCreditMarket.sol#L68-L70C14
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/LiquidateWithReplacement.sol#L67-L69
However, these tests are unnecessary, a validity of a tenor is tested when apr is calculated with getAPRByTenor method.
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/YieldCurveLibrary.sol#L121-L122
getAPRByTenor uses YieldCurveLibrary's getAPR function which chekcs validity of tenor.
The test from YieldCurveLibrary is stronger because a valid tenor boundary of considering yieldCurve is narrower than the boundary of riskConfig. So we do not need to check if the tenor is out of range by riskConfig.


## Proof of Concept
Executed the tests orginally desgined and they all passed.
It saves some gas in the tests as follow,

whole test of test_SellCreditMarket_validation() from (gas: 4979927) to (gas: 4893233)  
whole test of test_BuyCreditMarket_validation() from (gas: 4672647) to (gas: 4600436)  
whole test of test_SellCreditMarket_validation() from (gas: 5610960) to (gas: 5610153) 

## Recommended Mitigation Steps

```
        if (tenor < state.riskConfig.minTenor || tenor > state.riskConfig.maxTenor) {
            revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);
        }
```

Delete above codes in each file (BuyCreditMarket, SellCreditMarket, LiquidateWithReplacement)
https://github.com/Decipher100Eyes/code4rena-size/commit/f08eda89ac4f06a4da910333c61aea63f137b5b8

## Tools Used

foundry

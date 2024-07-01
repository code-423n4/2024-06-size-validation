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

## Impact

SellCreditMarket wastes gas and emits redundant events.

## Root cause and summary

In SellCreditMarket, currently it creates a dummy position with createDebtAndCreditPositions if creditPositionId is RESERVE_ID and executes createCreditPosition right after that.
However, this process is not necessary if the design of the code follows the way which BuyCreditMarket uses.

## Proof of Concept

        if (params.creditPositionId == RESERVED_ID) {
            state.createDebtAndCreditPositions({
                lender: params.lender,
                borrower: msg.sender,
                futureValue: creditAmountIn,
                dueDate: block.timestamp + tenor
            });
        } else {
            state.createCreditPosition({
                exitCreditPositionId: params.creditPositionId,
                lender: params.lender,
                credit: creditAmountIn
            });
        }

This code passes the all the tests that size protocol wants and saves the gas.
In every test using "SellCreditMarket", the new code spends less gas. The new code also does not create unnecessary events.

https://github.com/Decipher100Eyes/code4rena-size/commit/d4ae67f590adb9c33ebbc07e9f1ab0bb7c2967a0

## Tools Used

foundry

# Audit Report: Suggestions for optimizing gas usage in LiquidateWithReplacement.sol

## Impact and Summary

In Solidity, when a variable objet is assigned through memory copy, there is a difference in gas usage between a structure and a single variable. Therefore, in order to optimize gas usage, it is more efficient to memory copy only necessary variables rather than memory copy the entire structure.
To reduce gas usage in size project, we revised the `executeLiquidate()` function and run a test code file(`LiquidateWithReplacement.t.sol`). As a result, gas usage was reduced in 4 of the 5 unit tests.

## Root cause

In the `executeLiquidateWithReplacement()` function of `LiquidateWithReplacement.sol` file, the `debtPositionCopy` variable, which copies `debtPosition` (struct object) to memory, is needed to restore the `debtPosition.futureValue` (uint256) which is updated by the `executeLiquidate()` function.
At this time, rather than copying the entire `debtPosition` structure object, gas usage can be reduced by copying only the actually needed value, `debtPosition.futureValue`.

## Proof of Concept

You can compare code changes: https://github.com/Decipher100Eyes/code4arena-size/commit/4d084263ddc65c604675623b223dfad814d8de5e

The log below shows the results of comparing gas usage using the `LiquidateWithReplacement.t.sol` file. The “consumed” refers to the gas consumption of our proposed code, and the “expected” refers to the gas consumption of the existing code.

- **"LiquidateWithReplacementTest::test_LiquidateWithReplacement_liquidateWithReplacement_cannot_leave_new_borrower_liquidatable()": consumed "(gas: 2209750)" gas, expected "(gas: 2210048)" gas**
- **"LiquidateWithReplacementTest::test_LiquidateWithReplacement_liquidateWithReplacement_experiment()": consumed "(gas: 2600537)" gas, expected "(gas: 2600835)" gas**
- **"LiquidateWithReplacementTest::test_LiquidateWithReplacement_liquidateWithReplacement_updates_new_borrower_borrowOffer_different_rate()": consumed "(gas: 2990403)" gas, expected "(gas: 2990700)" gas**
- **"LiquidateWithReplacementTest::test_LiquidateWithReplacement_liquidateWithReplacement_updates_new_borrower_borrowOffer_same_rate()": consumed "(gas: 3003855)" gas, expected "(gas: 3004152)"**

## Tools Used

To evaluate the gas usage, we utilize Foundry framework(https://github.com/foundry-rs/foundry).

## Recommended Mitigation Steps

We recommend revising some codes in the file `LiquidateWithReplacement.sol` to use `futureValueCopy` instead of memory copying `debtPositionCopy`.

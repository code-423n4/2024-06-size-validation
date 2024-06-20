# G-01 validateSelfLiquidate checks borrower's collateral ratio twice

It appears that most of the code in `isCreditPositionSelfLiquidatable` is redundant.

In `validateSelfLiquidate`, `isCreditPositionSelfLiquidatable` is called which checks whether the borrower of the associated `debtPosition` is underwater:

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/RiskLibrary.sol#L81

Subsequently, it checks again if the borrower has a collateral ratio below 1:

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SelfLiquidate.sol#L46

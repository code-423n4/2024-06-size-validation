### G-01 Unnecessary function call


### Details 
In 
``Loanlibrar.sol``
```soliduty
        if (debt != 0) {
            return Math.mulDivDown(collateral, debtPosition.futureValue, debt);
        } else {
            return 0;
```
Futurevalue and debt updated in same time and same amount.
So,their ratio is one. 
It could simply return ``collateral`` value.

```solidity
            return Math.mulDivDown(debtPositionCollateral, creditPositionCredit, debtPositionFutureValue);
        } else {
            return 0;
```
Furthure increase in gas cost
## `variablePoolBorrowRateStaleRateInterval` variable overflows due to unsafe downcasting
In /src/libraries/actions/UpdateConfig.sol contract, on line 138 , there is a unsafe downcasting bug, as it takes `params.value` which is uint256 ( declared in lines 21-27 of the same contract ) , and casts it without verification to uint64. 
### Vulnerable code snippet ( lines 137-138 ):

```
        } else if (Strings.equal(params.key, "variablePoolBorrowRateStaleRateInterval")) {
            state.oracle.variablePoolBorrowRateStaleRateInterval = uint64(params.value); }
```

Downcasting from uint256 in Solidity does not revert on overflow even in the newer versions. This can result in undesired exploitation or bugs.

## Impact
the `variablePoolBorrowRateStaleRateInterval` variable is used in other contracts of the projects, for example, in the `SellCreditMarket` contract in the function `executeSellCreditMarket()` , if someone updates the oracle to a value bigger than type(uint64).max, it will overflow, and a wrong value will be passed to the function.

## Recommendation
Use OpenZeppelin's SafeCast library which reverts the transaction when such an operation overflows.
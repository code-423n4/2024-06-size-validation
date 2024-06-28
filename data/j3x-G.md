# Use uint256(1)/uint256(2) instead for true and false boolean states

There is a boolean `isMulticall` variable in the contract `SizeStorage.sol`.

In the contract `Multicall.sol` , ( ./src/libraries/Multicall.sol ) , the variable's value is set to true and then to false, ( line 27 and 44 ) , the gas cost for the access and state change using boolean can surpass 20,000 gas units (Gwarmaccess: 100 gas, and Gsset: 20000 gas)
By modifying the type of `isMulticall` to `uint256` and use uint256(2) to represent `false` and uint256(1) to represent `true`, the gas cost will be significantly less.

## References
https://detectors.auditbase.com/int-instead-of-bool-gas-optimization
https://medium.com/@kalexotsu/solidity-gas-optimization-stop-using-bools-for-true-false-values-e3a3d513f7fa
### [Best Practice]Multicall Library function:
The validation of Borrow and Debt token supplies in Multicall library can be implemented in Size's contract's multicall() function instead of library. This will make Library Multicall a generic functionality that can be used else where.


### [Best Practice]Below functions should not have payable modifier:
a) setUserConfiguration()
b) 

### Prevent re-setting a state variable with the same value
Not only is wasteful in terms of gas, but this is especially problematic when an event is emitted and the old and new values set are the same, as listeners might not expect this kind of scenario.

```solidity
Path: ./src/Size.sol

126:        state.oracle.variablePoolBorrowRate = borrowRate;	// @audit-issue
```
[126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L126-L126), 


#### Recommendation

Implement checks in your Solidity contracts to avoid re-setting state variables to their existing values. Prior to updating a state variable, compare the new value with the current value and proceed with the assignment only if they differ. Additionally, ensure that events related to state variable updates are emitted only when actual changes occur. This approach not only saves gas but also prevents confusion and unnecessary triggers in event listeners. Regularly reviewing and optimizing your contract for such redundancies can significantly enhance efficiency and clarity in contract operations.

### Cache Local Variable Array Length In For Loop
If not cached, the solidity compiler will always read the length of the array during each iteration. If it is a memory array, this is an extra mload operation (3 additional gas for each iteration except for the first).

```solidity
Path: ./src/libraries/Multicall.sol

33:        for (uint256 i = 0; i < data.length; i++) {	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L33-L33), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

48:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue

69:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L48-L48), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L69-L69), 


#### Recommendation

To optimize gas consumption, cache the length of memory arrays before for loop iterations in Solidity. This reduces redundant mload operations, saving 3 gas per iteration after the first and improving overall contract efficiency.

### Same cast is done multiple times
It's cheaper to do it once, and store the result to a variable.

```solidity
Path: ./src/oracle/PriceFeed.sol

88:        if (price <= 0) revert Errors.INVALID_PRICE(address(aggregator), price);	// @audit-issue: Same casting also done in line(s): `[90]`.
```
[88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L88-L88), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

60:        state.data.variablePool.supply(address(state.data.underlyingBorrowToken), amount, address(this), 0);	// @audit-issue: Same casting also done in line(s): `[55]`.

83:        state.data.variablePool.withdraw(address(state.data.underlyingBorrowToken), amount, to);	// @audit-issue: Same casting also done in line(s): `[78]`.
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L60-L60), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L83-L83), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

136:            state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));	// @audit-issue: Same casting also done in line(s): `[134]`.
```
[136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L136-L136), 


#### Recommendation

Identify instances in your Solidity contracts where the same value is cast to another type multiple times. Refactor these operations by performing the cast once and storing the result in a local variable. Use this variable for all subsequent operations requiring the cast value.

### `address(this)` should be cached
Cacheing saves gas when compared to repeating the calculation at each point it is used in the contract.

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

52:        state.data.underlyingBorrowToken.safeTransferFrom(from, address(this), amount);	// @audit-issue: `adress(this)` also used on line(s): [62, 57, 60]

85:        uint256 scaledAmount = scaledBalanceBefore - aToken.scaledBalanceOf(address(this));	// @audit-issue: `adress(this)` also used on line(s): [80]
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L52-L52), [85](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L85-L85), 


```solidity
Path: ./src/libraries/Multicall.sol

34:            results[i] = Address.functionDelegateCall(address(this), data[i]);	// @audit-issue: `adress(this)` also used on line(s): [29, 37]
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L34-L34), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

162:        state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);	// @audit-issue: `adress(this)` also used on line(s): [161]
```
[162](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L162-L162), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

254:            address(this),	// @audit-issue: `adress(this)` also used on line(s): [240, 248]
```
[254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L254-L254), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

72:            state.data.weth.forceApprove(address(this), amount);	// @audit-issue: `adress(this)` also used on line(s): [69, 73]
```
[72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L72-L72), 


#### Recommendation

To enhance gas efficiency, cache the contract's address by storing `address(this)` in a state variable at the point of contract deployment or initialization. Use this cached address throughout the contract instead of repeatedly calling `address(this)`. This practice reduces the gas cost associated with multiple computations of the contract's address, leading to more efficient contract execution, especially in scenarios with frequent usage of the contract's address.

### `msg.value` should be cached
Cacheing saves gas when compared to repeating the calculation at each point it is used in the contract.

```solidity
Path: ./src/libraries/actions/Deposit.sol

42:            revert Errors.INVALID_MSG_VALUE(msg.value);	// @audit-issue: `msg.value` also used on line(s): [41]
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L42-L42), 


#### Recommendation

Consider caching `msg.value` in a local variable at the start of payable functions where it is accessed multiple times. Implement this optimization by assigning `msg.value` to a local variable and using this variable for subsequent operations within the function. For example:
        ```solidity
        function myPayableFunction() public payable {
            uint256 cachedValue = msg.value;
            // Use cachedValue instead of msg.value for further operations
        }


### It is a waste of GAS to emit variable literals
Emitting variable literals (true, false, address(0), 1 etc...) in events is inefficient, as it consumes extra gas without providing added value. These literals are fixed values that can be accessed or hardcoded elsewhere in the smart contract or application, making their inclusion in events redundant and an unnecessary drain on resources during transaction execution.

```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

52:        emit TransferUnscaled(address(0), to, _unscale(scaledAmount));	// @audit-issue

66:        emit TransferUnscaled(from, address(0), _unscale(scaledAmount));	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L52-L52), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L66-L66), 


#### Recommendation

Refrain from including fixed variable literals as parameters in your Solidity contract events. Evaluate your event emissions and remove literals that do not provide dynamic information about contract state or actions. For instance, instead of emitting an event with a `true` literal to indicate success, consider naming the event in a way that inherently indicates success or failure, such as `ActionSuccessful()`:
```solidity
// Before optimization
event ActionTaken(bool success);

// After optimization
event ActionSuccessful();
event ActionFailed();
```


### Avoid repeating computations
In Solidity development, repeating the same computations within a contract can lead to unnecessary gas consumption and reduce the contract's efficiency. This is particularly relevant in functions that are called frequently or involve complex calculations. Repeating computations not only wastes computational resources but also increases the cost of executing transactions. By identifying and eliminating redundant calculations, developers can optimize contract performance, reduce gas costs, and improve overall execution speed.

```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

64:            if (self.tenors[i - 1] >= lastTenor) {	// @audit-issue: Same binary operation statement in line(s) between: ['67:67']

72:        if (self.tenors[self.tenors.length - 1] > maxTenor) {	// @audit-issue: Same binary operation statement in line(s) between: ['73:73']

121:        if (tenor < curveRelativeTime.tenors[0] || tenor > curveRelativeTime.tenors[length - 1]) {	// @audit-issue: Same binary operation statement in line(s) between: ['122:122']

135:                    return y0 + Math.mulDivDown(y1 - y0, tenor - x0, x1 - x0);	// @audit-issue: Same binary operation statement in line(s) between: ['137:137']
```
[64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L64-L64), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L72-L72), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L121-L121), [135](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L135-L135), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

321:            cashAmountIn = Math.mulDivUp(maxCredit, PERCENT, PERCENT + ratePerTenor);	// @audit-issue: Same binary operation statement in line(s) between: ['326:326']
```
[321](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L321-L321), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

119:        if (params.creditPositionToCompensateId == RESERVED_ID) {	// @audit-issue: Same binary operation statement in line(s) between: ['140:140']
```
[119](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L119-L119), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

138:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue: Same binary operation statement in line(s) between: ['164:164', '173:173', '176:176', '184:184', '195:195']

175:                    : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),	// @audit-issue: Same binary operation statement in line(s) between: ['177:177']
```
[138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L138-L138), [175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L175-L175), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

100:                state.feeConfig.swapFeeAPR != 0	// @audit-issue: Same binary operation statement in line(s) between: ['110:111']
101:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)

101:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)	// @audit-issue: Same binary operation statement in line(s) between: ['111:111']
```
[100](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L100-L101), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L101-L101), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

133:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue: Same binary operation statement in line(s) between: ['160:160', '163:163', '173:173', '179:179']

162:                    : Math.mulDivUp(creditPosition.credit, PERCENT, PERCENT + ratePerTenor),	// @audit-issue: Same binary operation statement in line(s) between: ['164:164']
```
[133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L133-L133), [162](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L162-L162), 


#### Recommendation

Review your Solidity contracts to identify any computations that are performed multiple times with the same inputs. Cache the results of these computations in local variables and reuse them within the function or across function calls if the state remains unchanged.

### Unused `error` definition
Note that there may be cases where an error superficially appears to be used, but this is only because there are multiple definitions of the error in different files. In such cases, the error definition should be moved into a separate file. The instances below are the unused definitions.


```solidity
Path: ./src/libraries/Errors.sol

49:    error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);	// @audit-issue

74:    error NULL_STALE_RATE();	// @audit-issue
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L49-L49), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L74-L74), 


#### Recommendation

Unused `error` definitions should be removed from the contract, and if needed, consolidated into a separate file to avoid duplication.

### Using Prefix Operators Costs Less Gas Than Postfix Operators in Loops
Conditions can be optimized issues in Solidity refer to situations where smart contract developers write conditional statements that can be simplified or optimized for better gas efficiency, readability, and maintainability. Optimizing conditions can lead to more cost-effective and secure smart contracts.

```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

63:        for (uint256 i = self.tenors.length; i != 0; i--) {	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L63-L63), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

72:        uint256 debtPositionId = state.data.nextDebtPositionId++;	// @audit-issue

84:        uint256 creditPositionId = state.data.nextCreditPositionId++;	// @audit-issue

121:            uint256 creditPositionId = state.data.nextCreditPositionId++;	// @audit-issue
```
[72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L72-L72), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L84-L84), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L121-L121), 


```solidity
Path: ./src/libraries/Multicall.sol

33:        for (uint256 i = 0; i < data.length; i++) {	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L33-L33), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

48:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue

69:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L48-L48), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L69-L69), 


#### Recommendation

To improve gas efficiency in your Solidity code, prefer using prefix operators (e.g., `++i` or `--i`) instead of postfix operators (e.g., `i++` or `i--`) within loops. Prefix operators typically result in lower gas costs and can contribute to more efficient contract execution.

### Multiple accesses of a mapping/array should use a local variable cache
The instances below point to the second+ access of a value inside a mapping/array, within a function. Caching a mapping's value in a local `storage` or `calldata` variable when the value is accessed [multiple times](https://gist.github.com/IllIllI000/ec23a57daa30a8f8ca8b9681c8ccefb0), saves **~42 gas per access** due to not having to recalculate the key's keccak256 hash (Gkeccak256 - **30 gas**) and that calculation's associated stack operations. Caching an array's struct avoids recalculating the array offsets into memory/calldata

```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

51:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionIds[i]);	// @audit-issue: Same index access of variable `params` is used also on line(s): ['49', '54', '55'].

70:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionIds[i]);	// @audit-issue: Same index access of variable `params` is used also on line(s): ['73'].
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L51-L51), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L70-L70), 


#### Recommendation

When a mapping or array value is accessed multiple times within a function, cache it in a local `storage` or `calldata` variable. This approach minimizes the gas cost by reducing the number of hash computations for mappings and offset calculations for arrays. Carefully review your functions to identify opportunities for such optimizations, especially in functions with frequent or repeated accesses to the same mapping or array element, to enhance gas efficiency in your contracts.

### State Variables That Are Used Multiple Times In a Function Should Be Cached In Stack Variables
When performing multiple operations on a state variable in a function, it is recommended to cache it first. Either multiple reads or multiple writes to a state variable can save gas by caching it on the stack. Caching of a state variable replaces each Gwarmaccess (100 gas) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses. Saves 100 gas per instance.


```solidity
Path: ./src/SizeView.sol

91:            borrowAToken: state.data.borrowAToken,	// @audit-issue: State variable `state` is used also on line(s): ['92', '89', '88', '87', '85', '90', '86'].

136:            state.data.nextCreditPositionId - CREDIT_POSITION_ID_START	// @audit-issue: State variable `state` is used also on line(s): ['135'].

150:                variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval	// @audit-issue: State variable `state` is used also on line(s): ['149', '148'].

165:                variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,	// @audit-issue: State variable `state` is used also on line(s): ['164', '166'].
```
[91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L91-L91), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L136-L136), [150](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L150-L150), [165](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L165-L165), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

45:        state.data.debtToken.burn(debtPosition.borrower, repayAmount);	// @audit-issue: State variable `debtPosition` is used also on line(s): ['49'].

111:                exitCreditPositionId, lender, exitCreditPosition.credit, exitCreditPosition.forSale	// @audit-issue: State variable `exitCreditPosition` is used also on line(s): ['107'].

143:            creditPositionId, creditPosition.lender, creditPosition.credit, creditPosition.forSale	// @audit-issue: State variable `creditPosition` is used also on line(s): ['140'].
```
[45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L45-L45), [111](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L111-L111), [143](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L143-L143), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

46:        if (state.collateralRatio(debtPosition.borrower) >= PERCENT) {	// @audit-issue: State variable `debtPosition` is used also on line(s): ['47', '42'].

52:            revert Errors.LIQUIDATOR_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue: State variable `creditPosition` is used also on line(s): ['51'].
```
[46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L46-L46), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

57:            uint256 tenor = debtPositionToRepay.dueDate - block.timestamp;	// @audit-issue: State variable `debtPositionToRepay` is used also on line(s): ['76'].

83:            if (creditPositionToCompensate.lender != debtPositionToRepay.borrower) {	// @audit-issue: State variable `creditPositionToCompensate` is used also on line(s): ['84'].

83:            if (creditPositionToCompensate.lender != debtPositionToRepay.borrower) {	// @audit-issue: State variable `debtPositionToRepay` is used also on line(s): ['93', '94'].
```
[57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L57-L57), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L83-L83), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L83-L83), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

75:                revert Errors.BORROWER_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue: State variable `creditPosition` is used also on line(s): ['74'].

88:                revert Errors.NOT_ENOUGH_CREDIT(params.amount, creditPosition.credit);	// @audit-issue: State variable `creditPosition` is used also on line(s): ['87'].
```
[75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L75-L75), [88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L88-L88), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

121:            debtPosition.borrower, state.feeConfig.feeRecipient, protocolProfitCollateralToken	// @audit-issue: State variable `debtPosition` is used also on line(s): ['119', '81', '86'].

91:        uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);	// @audit-issue: State variable `debtPosition` is used also on line(s): ['125', '118', '98'].
```
[121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L121-L121), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L91-L91), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

156:            debtPosition.futureValue,	// @audit-issue: State variable `debtPosition` is used also on line(s): ['160'].
```
[156](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L156-L156), 


```solidity
Path: ./src/libraries/actions/Claim.sol

55:        state.reduceCredit(params.creditPositionId, creditPosition.credit);	// @audit-issue: State variable `creditPosition` is used also on line(s): ['53'].
```
[55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L55-L55), 


```solidity
Path: ./src/libraries/actions/Repay.sol

49:        state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);	// @audit-issue: State variable `debtPosition` is used also on line(s): ['51'].
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L49-L49), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

79:            borrower = creditPosition.lender;	// @audit-issue: State variable `creditPosition` is used also on line(s): ['74'].
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L79-L79), 


#### Recommendation

Cache state variables in stack or local memory variables within functions when they are used multiple times. This approach replaces costlier Gwarmaccess operations with cheaper stack reads, saving approximately 100 gas per instance and optimizing overall contract performance.

### Divisions can be unchecked to save gas
The expression type(int).min/(-1) is the only case where division causes an overflow. Therefore, uncheck can be used to save gas in scenarios where it is certain that such an overflow will not occur.

```solidity
Path: ./src/libraries/Math.sol

58:            uint256 mid = (low + high) / 2;	// @audit-issue
```
[58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L58-L58), 


#### Recommendation

Utilize 'unchecked' blocks in Solidity for divisions where overflow is impossible, such as when 'type(int).min/(-1)' is not a concern. This can save gas by bypassing overflow checks in these specific cases.

### Add `unchecked {}` for subtractions where the operands cannot underflow because of a previous `require()` or `if`-statement
Unchecked keyword can be added to such scenerios: 
`require(a <= b); x = b - a` => `require(a <= b); unchecked { x = b - a }`

```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

135:                    return y0 + Math.mulDivDown(y1 - y0, tenor - x0, x1 - x0);	// @audit-issue

137:                    return y0 - Math.mulDivDown(y0 - y1, tenor - x0, x1 - x0);	// @audit-issue
```
[135](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L135-L135), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L137-L137), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

203:            cashAmountOut = maxCashAmountOut - fees;	// @audit-issue

213:            cashAmountOut = maxCashAmountOut - fees;	// @audit-issue

241:            maxCashAmountOutFragmentation = maxCashAmountOut - state.feeConfig.fragmentationFee;	// @audit-issue

294:            uint256 netCashAmountIn = cashAmountIn - state.feeConfig.fragmentationFee;	// @audit-issue
```
[203](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L203-L203), [213](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L213-L213), [241](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L241-L241), [294](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L294-L294), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

97:                assignedCollateral - debtInCollateralToken,	// @audit-issue
```
[97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L97-L97), 


#### Recommendation

In scenarios where subtraction cannot result in underflow due to prior `require()` or `if`-statements, wrap these operations in an `unchecked` block to save gas. This optimization should only be applied when the safety of the operation is assured. Carefully analyze each case to confirm that underflow is impossible before implementing `unchecked` blocks, as incorrect usage can lead to vulnerabilities in the contract.

### Stack variable is only used once
If the variable is only accessed once, it's cheaper to use the assigned value directly that one time, and save the 3 gas the extra stack assignment would spend

```solidity
Path: ./src/Size.sol

125:        uint128 oldBorrowRate = state.oracle.variablePoolBorrowRate;	// @audit-issue: oldBorrowRate used only on line: 128

180:        uint256 amount = state.executeBuyCreditMarket(params);	// @audit-issue: amount used only on line: 184

190:        uint256 amount = state.executeSellCreditMarket(params);	// @audit-issue: amount used only on line: 194
```
[125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L125-L125), [180](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L180-L180), [190](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L190-L190), 


```solidity
Path: ./src/SizeView.sol

174:        DebtPosition memory debtPosition = state.getDebtPosition(debtPositionId);	// @audit-issue: debtPosition used only on line: 175
```
[174](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L174-L174), 


```solidity
Path: ./src/oracle/PriceFeed.sol

66:            (, int256 answer, uint256 startedAt,,) = sequencerUptimeFeed.latestRoundData();	// @audit-issue: answer used only on line: 68

66:            (, int256 answer, uint256 startedAt,,) = sequencerUptimeFeed.latestRoundData();	// @audit-issue: startedAt used only on line: 73
```
[66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L66-L66), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L66-L66), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

54:        uint256 collateral = state.data.collateralToken.balanceOf(account);	// @audit-issue: collateral used only on line: 60

56:        uint256 debtWad = Math.amountToWad(debt, state.data.underlyingBorrowToken.decimals());	// @audit-issue: debtWad used only on line: 60

57:        uint256 price = state.oracle.priceFeed.getPrice();	// @audit-issue: price used only on line: 60

76:        CreditPosition storage creditPosition = state.data.creditPositions[creditPositionId];	// @audit-issue: creditPosition used only on line: 77

77:        DebtPosition storage debtPosition = state.data.debtPositions[creditPosition.debtPositionId];	// @audit-issue: debtPosition used only on line: 81

78:        LoanStatus status = state.getLoanStatus(creditPositionId);	// @audit-issue: status used only on line: 81

105:        DebtPosition storage debtPosition = state.data.debtPositions[debtPositionId];	// @audit-issue: debtPosition used only on line: 111
```
[54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L54-L54), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L56-L56), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L57-L57), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L76-L76), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L77-L77), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L78-L78), [105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L105-L105), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

67:        uint256 apr = getAPRByTenor(self, params, tenor);	// @audit-issue: apr used only on line: 68

95:        uint256 apr = getAPRByTenor(self, params, tenor);	// @audit-issue: apr used only on line: 96
```
[67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L67-L67), [95](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L95-L95), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

114:        CreditPosition memory creditPosition = getCreditPosition(state, creditPositionId);	// @audit-issue: creditPosition used only on line: 115

154:        uint256 collateral = state.data.collateralToken.balanceOf(debtPosition.borrower);	// @audit-issue: collateral used only on line: 157

177:        uint256 debtPositionCollateral = getDebtPositionAssignedCollateral(state, debtPosition);	// @audit-issue: debtPositionCollateral used only on line: 182

178:        uint256 creditPositionCredit = creditPosition.credit;	// @audit-issue: creditPositionCredit used only on line: 182
```
[114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L114-L114), [154](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L154-L154), [177](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L177-L177), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L178-L178), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

31:        uint256 debtTokenAmountWad = Math.amountToWad(debtTokenAmount, state.data.underlyingBorrowToken.decimals());	// @audit-issue: debtTokenAmountWad used only on line: 33
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L31-L31), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

24:        IERC20Metadata underlyingCollateralToken = IERC20Metadata(state.data.underlyingCollateralToken);	// @audit-issue: underlyingCollateralToken used only on line: 25

37:        IERC20Metadata underlyingCollateralToken = IERC20Metadata(state.data.underlyingCollateralToken);	// @audit-issue: underlyingCollateralToken used only on line: 39

57:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue: scaledBalanceBefore used only on line: 62

62:        uint256 scaledAmount = aToken.scaledBalanceOf(address(this)) - scaledBalanceBefore;	// @audit-issue: scaledAmount used only on line: 64

80:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue: scaledBalanceBefore used only on line: 85

85:        uint256 scaledAmount = scaledBalanceBefore - aToken.scaledBalanceOf(address(this));	// @audit-issue: scaledAmount used only on line: 87
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L24-L24), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L37-L37), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L57-L57), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L62-L62), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L80-L80), [85](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L85-L85), 


```solidity
Path: ./src/libraries/Multicall.sol

29:        uint256 borrowATokenSupplyBefore = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue: borrowATokenSupplyBefore used only on line: 41

30:        uint256 debtTokenSupplyBefore = state.data.debtToken.totalSupply();	// @audit-issue: debtTokenSupplyBefore used only on line: 41

37:        uint256 borrowATokenSupplyAfter = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue: borrowATokenSupplyAfter used only on line: 41

38:        uint256 debtTokenSupplyAfter = state.data.debtToken.totalSupply();	// @audit-issue: debtTokenSupplyAfter used only on line: 41
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L30-L30), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L38-L38), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

63:        DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue: debtPosition used only on line: 70

65:        uint256 assignedCollateral = state.getCreditPositionProRataAssignedCollateral(creditPosition);	// @audit-issue: assignedCollateral used only on line: 70
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L63-L63), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L65-L65), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

43:        CreditPosition storage creditPositionWithDebtToRepay =	// @audit-issue: creditPositionWithDebtToRepay used only on line: 48
44:            state.getCreditPosition(params.creditPositionWithDebtToRepayId);

66:            DebtPosition storage debtPositionToCompensate =	// @audit-issue: debtPositionToCompensate used only on line: 72
67:                state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId);

113:        DebtPosition storage debtPositionToRepay =	// @audit-issue: debtPositionToRepay used only on line: 124
114:            state.getDebtPositionByCreditPositionId(params.creditPositionWithDebtToRepayId);

136:        uint256 exiterCreditRemaining = creditPositionToCompensate.credit - amountToCompensate;	// @audit-issue: exiterCreditRemaining used only on line: 146

148:            uint256 fragmentationFeeInCollateral = Math.min(	// @audit-issue: fragmentationFeeInCollateral used only on line: 153
149:                state.debtTokenAmountToCollateralTokenAmount(state.feeConfig.fragmentationFee),
150:                state.data.collateralToken.balanceOf(msg.sender)
151:            );
```
[43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L43-L44), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L66-L67), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L113-L114), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L136-L136), [148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L148-L151), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

141:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue: debtPosition used only on line: 144
```
[141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L141-L141), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

38:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue: debtPosition used only on line: 47

80:        LoanStatus loanStatus = state.getLoanStatus(params.debtPositionId);	// @audit-issue: loanStatus used only on line: 83

81:        uint256 collateralRatio = state.collateralRatio(debtPosition.borrower);	// @audit-issue: collateralRatio used only on line: 83

86:        uint256 collateralProtocolPercent = state.isUserUnderwater(debtPosition.borrower)	// @audit-issue: collateralProtocolPercent used only on line: 112
87:            ? state.feeConfig.collateralProtocolPercent
88:            : state.feeConfig.overdueCollateralProtocolPercent;

96:            uint256 liquidatorReward = Math.min(	// @audit-issue: liquidatorReward used only on line: 100
97:                assignedCollateral - debtInCollateralToken,
98:                Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
99:            );

107:            uint256 collateralRemainderCap =	// @audit-issue: collateralRemainderCap used only on line: 110
108:                Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L38-L38), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L80-L80), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L81-L81), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L86-L88), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L96-L99), [107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L107-L108), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

51:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue: debtPosition used only on line: 66

128:        BorrowOffer storage borrowOffer = state.data.users[params.borrower].borrowOffer;	// @audit-issue: borrowOffer used only on line: 138

129:        uint256 tenor = debtPositionCopy.dueDate - block.timestamp;	// @audit-issue: tenor used only on line: 144

138:        uint256 ratePerTenor = borrowOffer.getRatePerTenor(	// @audit-issue: ratePerTenor used only on line: 146
139:            VariablePoolBorrowRateParams({
140:                variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
141:                variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
142:                variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
143:            }),
144:            tenor
145:        );
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L51-L51), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L128-L128), [129](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L129-L129), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L138-L145), 


```solidity
Path: ./src/libraries/actions/Claim.sol

32:        CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue: creditPosition used only on line: 40

50:        DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue: debtPosition used only on line: 53

52:        uint256 claimAmount = Math.mulDivDown(	// @audit-issue: claimAmount used only on line: 56
53:            creditPosition.credit, state.data.borrowAToken.liquidityIndex(), debtPosition.liquidityIndexAtRepayment
54:        );
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L32-L32), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L50-L50), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L52-L54), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

26:        BorrowOffer memory borrowOffer = BorrowOffer({curveRelativeTime: params.curveRelativeTime});	// @audit-issue: borrowOffer used only on line: 29
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L26-L26), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

30:        LoanOffer memory loanOffer =	// @audit-issue: loanOffer used only on line: 34
31:            LoanOffer({maxDueDate: params.maxDueDate, curveRelativeTime: params.curveRelativeTime});
```
[30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L30-L31), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

74:            User storage user = state.data.users[creditPosition.lender];	// @audit-issue: user used only on line: 75

137:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue: debtPosition used only on line: 141
```
[74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L74-L74), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L137-L137), 


#### Recommendation

Eliminate single-use stack variables in Solidity to optimize gas consumption. Directly use the assigned value in the place of the variable. This approach saves the 3 gas typically used for the extra stack assignment, streamlining the function's execution and enhancing overall gas efficiency.

### Storage Layout Optimization
Storage Layout Optimization in Solidity involves arranging state variables to minimize gas costs. Since storage is expensive, combining variables into as few slots as possible and deleting unneeded variables can significantly reduce the gas needed for contract operations.

```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

17:struct SellCreditMarketParams {	// @audit-issue: ['Current storage layout is like this: ', "However it can be optimized by 1 storage by storing variables like in order: \n\t`['creditPositionId', 'amount', 'tenor', 'deadline', 'maxAPR', 'lender', 'exactAmountIn']`"]
18:    // The lender
19:    address lender;
20:    // The credit position ID to sell
21:    // If RESERVED_ID, a new credit position will be created
22:    uint256 creditPositionId;
23:    // The amount of credit to sell
24:    uint256 amount;
25:    // The tenor of the loan
26:    // If creditPositionId is not RESERVED_ID, this value is ignored and the tenor of the existing loan is used
27:    uint256 tenor;
28:    // The deadline for the transaction
29:    uint256 deadline;
30:    // The maximum APR for the loan
31:    uint256 maxAPR;
32:    // Whether amount means credit or cash
33:    bool exactAmountIn;
34:}
```
[17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L17-L34), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

16:struct BuyCreditMarketParams {	// @audit-issue: ['Current storage layout is like this: ', "However it can be optimized by 1 storage by storing variables like in order: \n\t`['creditPositionId', 'amount', 'tenor', 'deadline', 'minAPR', 'borrower', 'exactAmountIn']`"]
17:    // The borrower
18:    // If creditPositionId is not RESERVED_ID, this value is ignored and the owner of the existing credit is used
19:    address borrower;
20:    // The credit position ID to buy
21:    // If RESERVED_ID, a new credit position will be created
22:    uint256 creditPositionId;
23:    // The amount of credit to buy
24:    uint256 amount;
25:    // The tenor of the loan
26:    // If creditPositionId is not RESERVED_ID, this value is ignored and the tenor of the existing loan is used
27:    uint256 tenor;
28:    // The deadline for the transaction
29:    uint256 deadline;
30:    // The minimum APR for the loan
31:    uint256 minAPR;
32:    // Whether amount means cash or credit
33:    bool exactAmountIn;
34:}
```
[16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L16-L34), 


#### Recommendation

To optimize storage and reduce gas costs, rearrange the storage variables in a way that makes the most of each 32-byte storage slot.

### Don't emit events inside a loop
Emitting an event has an overhead of 375 gas, which will be incurred on every iteration of the loop. It is cheaper to emit only once after the loop has finished.

```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

72:            emit Events.UpdateCreditPosition(	// @audit-issue
```
[72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L72-L72), 


#### Recommendation

To optimize gas usage, avoid emitting events inside loops in Solidity. Instead, emit a single event after the loop completes, thereby saving the 375 gas overhead incurred on each iteration.

### `Internal` functions only called once can be inlined to save gas
If an internal function is only used once, there is no need to modularize it, unless the function calling it would otherwise be too long and complex. Not inlining costs 20 to 40 gas because of two extra JUMP instructions and additional stack operations needed for function calls.

```solidity
Path: ./src/libraries/Math.sol

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {	// @audit-issue
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L51), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

62:    function validateOwner(address owner) internal pure {	// @audit-issue

146:    function validateInitializeDataParams(InitializeDataParams memory d) internal view {	// @audit-issue

193:    function executeInitializeFeeConfig(State storage state, InitializeFeeConfigParams memory f) internal {	// @audit-issue

207:    function executeInitializeRiskConfig(State storage state, InitializeRiskConfigParams memory r) internal {	// @audit-issue

222:    function executeInitializeOracle(State storage state, InitializeOracleParams memory o) internal {	// @audit-issue

230:    function executeInitializeData(State storage state, InitializeDataParams memory d) internal {	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L62-L62), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L146-L146), [193](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L193-L193), [207](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L207-L207), [222](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L222-L222), [230](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L230-L230), 


#### Recommendation

Inline 'internal' functions in Solidity that are called only once to save gas. This avoids the additional gas cost of 20 to 40 units associated with extra JUMP instructions and stack operations required for separate function calls, unless the calling function becomes too complex.

### Optimizing Arithmetic with Shift Operators for Division and Multiplication by Powers of Two
In computational operations, especially within contexts where efficiency matters, certain arithmetic operations can be optimized. One such optimization is leveraging shift operators for division and multiplication by powers of two. This stems from the binary nature of numbers in computing, where shifting bits to the left (using `<<`) effectively multiplies a number by 2 for each position shifted, and shifting bits to the right (using `>>`) divides the number by 2 for each position shifted.

In Solidity, and many other programming languages, using shift operators can be more gas-efficient than traditional arithmetic operations for these specific cases. For instance, instead of performing `value * 4`, one can use `value << 2`, and instead of `value / 4`, one can use `value >> 2`. These optimizations can result in reduced gas costs and faster execution times.


```solidity
Path: ./src/libraries/Math.sol

58:            uint256 mid = (low + high) / 2;	// @audit-issue
```
[58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L58-L58), 


#### Recommendation

When performing division or multiplication by powers of two in Solidity, consider using shift operators (`<<` for multiplication and `>>` for division) for improved gas efficiency. This optimization takes advantage of the binary nature of computing and can lead to reduced gas costs and faster execution times.

### Use `private` Rather than `public` for Constants 
In Solidity, constants represent immutable values that cannot be changed after they are set at compile-time. By default, constants have internal visibility, meaning they can be accessed within the contract they are declared in and in derived contracts. If a constant is explicitly declared as `public`, Solidity automatically generates a getter function for it. While this might seem harmless, it actually incurs a gas overhead, especially when the contract is deployed, as the EVM needs to generate bytecode for that getter. Conversely, declaring constants as `private` ensures that no additional getter is generated, optimizing gas usage.

```solidity
Path: ./src/oracle/PriceFeed.sol

28:    uint256 public constant decimals = 18;	// @audit-issue
```
[28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L28-L28), 


#### Recommendation

To optimize gas usage in your Solidity contracts, declare constants with `private` visibility rather than `public` when possible. Using `private` prevents the automatic generation of a getter function, reducing gas overhead, especially during contract deployment.

### Consider pre-calculating the address of `address(this)` to save gas
Use `foundry`'s [`script.sol`](https://book.getfoundry.sh/reference/forge-std/compute-create-address) or `solady`'s [`LibRlp.sol`](https://github.com/Vectorized/solady/blob/main/src/utils/LibRLP.sol) to save the value in a constant, which will avoid having to spend gas to push the value on the stack every time it's used.

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

25:        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);	// @audit-issue

52:        state.data.underlyingBorrowToken.safeTransferFrom(from, address(this), amount);	// @audit-issue

57:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue

60:        state.data.variablePool.supply(address(state.data.underlyingBorrowToken), amount, address(this), 0);	// @audit-issue

62:        uint256 scaledAmount = aToken.scaledBalanceOf(address(this)) - scaledBalanceBefore;	// @audit-issue

80:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue

85:        uint256 scaledAmount = scaledBalanceBefore - aToken.scaledBalanceOf(address(this));	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L25-L25), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L52-L52), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L57-L57), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L60-L60), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L62-L62), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L80-L80), [85](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L85-L85), 


```solidity
Path: ./src/libraries/Multicall.sol

29:        uint256 borrowATokenSupplyBefore = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue

34:            results[i] = Address.functionDelegateCall(address(this), data[i]);	// @audit-issue

37:        uint256 borrowATokenSupplyAfter = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L29-L29), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L34-L34), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L37-L37), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

118:        state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);	// @audit-issue
```
[118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L118-L118), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

161:        state.data.borrowAToken.transferFrom(address(this), params.borrower, issuanceValue);	// @audit-issue

162:        state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);	// @audit-issue
```
[161](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L161-L161), [162](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L162-L162), 


```solidity
Path: ./src/libraries/actions/Claim.sol

56:        state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L56-L56), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

240:            address(this),	// @audit-issue

248:            address(this),	// @audit-issue

254:            address(this),	// @audit-issue
```
[240](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L240-L240), [248](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L248-L248), [254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L254-L254), 


```solidity
Path: ./src/libraries/actions/Repay.sol

49:        state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);	// @audit-issue
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L49-L49), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

69:            amount = address(this).balance;	// @audit-issue

72:            state.data.weth.forceApprove(address(this), amount);	// @audit-issue

73:            from = address(this);	// @audit-issue
```
[69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L69-L69), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L73-L73), 


#### Recommendation

To enhance gas efficiency, cache the contract's address by storing `address(this)` in a state variable at the point of contract deployment or initialization. Use this cached address throughout the contract instead of repeatedly calling `address(this)`. This practice reduces the gas cost associated with multiple computations of the contract's address, leading to more efficient contract execution, especially in scenarios with frequent usage of the contract's address.

### All variables can be packed into fewer storage slots by truncating timestamp bytes
By using a `uint32` rather than a larger type for variables that track timestamps, one can save gas by using fewer storage slots per struct, at the expense of the protocol breaking after the year 2106 (when `uint32` wraps). If this is an acceptable tradeoff, if variables occupying the same slot are both written the same function or by the constructor, avoids a separate Gsset (**20000 gas**). Reads of the variables can also be cheaper

```solidity
Path: ./src/oracle/PriceFeed.sol

86:        (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();	// @audit-issue
```
[86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L86-L86), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

29:    uint256 deadline;	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L29-L29), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

27:    uint256 deadline;	// @audit-issue
```
[27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L27-L27), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

29:    uint256 deadline;	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L29-L29), 


#### Recommendation

Evaluate the use of `uint32` for timestamp variables in your Solidity contracts, especially within structs that are frequently written to or read from storage. Ensure that the reduced size will not impact the functionality of your contract, particularly considering the 2106 overflow limitation. When implementing this optimization, aim to group such variables together in the same storage slot to maximize gas savings. This strategy is most effective when these variables are updated together, as in a constructor or specific functions, allowing for a single storage operation (`Gsset`) instead of multiple. Always balance the benefits of gas savings against the longevity and future-proofing of your contract.

### Counting down in for statements is more gas efficient
Looping downwards in Solidity is more gas efficient due to how the EVM compares variables. In a 'for' loop that counts down, the end condition is usually to compare with zero, which is cheaper than comparing with another number. As such, restructure your loops to count downwards where possible.

```solidity
Path: ./src/libraries/Multicall.sol

33:        for (uint256 i = 0; i < data.length; i++) {	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L33-L33), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

48:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue

69:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L48-L48), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L69-L69), 


#### Recommendation

Where feasible, refactor `for` loops in your Solidity contracts to count downwards. Adjust the loop initialization, condition, and iteration statements to decrement the loop variable and terminate the loop when it reaches zero. This approach can lead to gas savings, making your contract more efficient in terms of execution costs. Ensure that this refactoring aligns with the logic and requirements of your contract, and thoroughly test to confirm that the revised loop behavior matches the intended functionality.

### Use solady library where possible to save gas
The following OpenZeppelin imports have a Solady equivalent, as such they can be used to save GAS as Solady modules have been specifically designed to be as GAS efficient as possible

```solidity
Path: ./src/SizeViewData.sol

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue
```
[5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L5-L5), 


```solidity
Path: ./src/Size.sol

4:import {AccessControlUpgradeable} from "@openzeppelin/contracts-upgradeable/access/AccessControlUpgradeable.sol";	// @audit-issue

7:import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";	// @audit-issue

8:import {PausableUpgradeable} from "@openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L4-L4), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L8-L8), 


```solidity
Path: ./src/SizeStorage.sol

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue
```
[5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L5-L5), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

6:import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";	// @audit-issue
```
[5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L6-L6), 


```solidity
Path: ./src/libraries/Multicall.sol

4:import {Address} from "@openzeppelin/contracts/utils/Address.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L4-L4), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue
```
[5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L5-L5), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

4:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

5:import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L5-L5), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

4:import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";	// @audit-issue

5:import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L5-L5), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

6:import {IERC20Metadata} from "@openzeppelin/contracts/interfaces/IERC20Metadata.sol";	// @audit-issue
```
[6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L6-L6), 


#### Recommendation

Evaluate and, where appropriate, integrate Solady modules in your Solidity contracts as alternatives to similar OpenZeppelin imports. Focus on areas where gas efficiency can be significantly improved. Ensure that any replacement with Solady's modules does not compromise the security or functionality of your contracts. Conduct thorough testing and code reviews when making such substitutions to confirm compatibility and maintain the integrity of your application. Stay informed about updates and community feedback on both libraries to make informed decisions about their use in your projects.

### Use bitmap to save gas
Bitmaps in Solidity are essentially a way of representing a set of boolean values within an integer type variable such as `uint256`. Each bit in the integer represents a true or false value (1 or 0), thus allowing efficient storage of multiple boolean values.

Bitmaps can save gas in the Ethereum network because they condense a lot of information into a small amount of storage. In Ethereum, storage is one of the most significant costs in terms of gas usage. By reducing the amount of storage space needed, you can potentially save on gas fees.

Here's a quick comparison:

If you were to represent 256 different boolean values in the traditional way, you would have to declare 256 different `bool` variables. Given that each `bool` occupies a storage slot and each storage slot costs 20,000 gas to initialize, you would end up paying a considerable amount of gas.

On the other hand, if you were to use a bitmap, you could store these 256 boolean values within a single `uint256` variable. In other words, you'd only pay for a single storage slot, resulting in significant gas savings.

However, it's important to note that while bitmaps can provide gas efficiencies, they do add complexity to the code, making it harder to read and maintain. Also, using bitmaps is efficient only when dealing with a large number of boolean variables that are frequently changed or accessed together. 

In contrast, the straightforward counterpart to bitmaps would be using arrays or mappings to store boolean values, with each `bool` value occupying its own storage slot. This approach is simpler and more readable but could potentially be more expensive in terms of gas usage.

```solidity
Path: ./src/libraries/Multicall.sol

27:        state.data.isMulticall = true;	// @audit-issue

44:        state.data.isMulticall = false;	// @audit-issue
```
[27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L27-L27), [44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L44-L44), 


#### Recommendation

Consider using bitmaps in your Solidity contracts when you need to store and manipulate a large set of boolean values. This approach is particularly advantageous in terms of gas efficiency for scenarios involving frequent changes or accesses to these values. However, balance this efficiency with code readability and maintainability. Ensure that the use of bitmaps is well-documented, and consider the complexity it introduces into the code. Employ bitmaps judiciously, especially when their use results in significant gas savings, and the logic they represent is a core aspect of the contract's functionality.

### Using `bool`s for storage incurs overhead
[Booleans are more expensive than uint256](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27) or any type that takes up a full word because each write operation emits an extra SLOAD to first read the slot's contents, replace the bits taken up by the boolean, and then write back. This is the compiler's defense against contract upgrades and pointer aliasing, and it cannot be disabled.
Use `uint256(0)` and `uint256(1)` for true/false to avoid a Gwarmaccess (**[100 gas](https://gist.github.com/IllIllI000/1b70014db712f8572a72378321250058)**) for the extra SLOAD.


```solidity
Path: ./src/SizeStorage.sol

24:    bool allCreditPositionsForSaleDisabled;	// @audit-issue

94:    bool isMulticall;	// @audit-issue
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L24-L24), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L94-L94), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

36:    bool forSale;	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L36-L36), 


```solidity
Path: ./src/libraries/Events.sol

31:        bool exactAmountIn	// @audit-issue

43:        bool exactAmountIn	// @audit-issue

65:        bool indexed allCreditPositionsForSaleDisabled,	// @audit-issue

66:        bool indexed creditPositionIdsForSale,	// @audit-issue

92:    event UpdateCreditPosition(uint256 indexed creditPositionId, address indexed lender, uint256 credit, bool forSale);	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L31-L31), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L43-L43), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L65-L65), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L66-L66), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L92-L92), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

15:    bool allCreditPositionsForSaleDisabled;	// @audit-issue

17:    bool creditPositionIdsForSale;	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L15-L15), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L17-L17), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

33:    bool exactAmountIn;	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L33-L33), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

33:    bool exactAmountIn;	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L33-L33), 


#### Recommendation

To minimize gas overhead in your Solidity contracts, consider using `uint256(1)` and `uint256(2)` to represent `true` and `false`, respectively, instead of `bool` types for storage. This approach avoids additional `SLOAD` and `SSTORE` operations, resulting in more gas-efficient code.

### Usage of `uints`/`ints` smaller than 32 bytes (256 bits) incurs overhead
When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size.
https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html Each operation involving a `uint8` costs an extra [22-28](https://gist.github.com/IllIllI000/9388d20c70f9a4632eb3ca7836f54977) gas (depending on whether the other operand is also a variable of type `uint8`) as compared to ones involving `uint256`, due to the compiler having to clear the higher bits of the memory word before operating on the `uint8`, as well as the associated stack operations of doing so. Use a larger size then downcast where needed


```solidity
Path: ./src/Size.sol

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue

125:        uint128 oldBorrowRate = state.oracle.variablePoolBorrowRate;	// @audit-issue
```
[120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L120), [125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L125-L125), 


```solidity
Path: ./src/libraries/Math.sol

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

15:    uint8 internal immutable _decimals;	// @audit-issue

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L15-L15), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

31:        uint8 decimals_	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L31-L31), 


```solidity
Path: ./src/SizeStorage.sol

61:    uint128 variablePoolBorrowRate;	// @audit-issue

63:    uint64 variablePoolBorrowRateUpdatedAt;	// @audit-issue

65:    uint64 variablePoolBorrowRateStaleRateInterval;	// @audit-issue
```
[61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L61-L61), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L63-L63), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L65-L65), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

19:    uint128 variablePoolBorrowRate;	// @audit-issue

21:    uint64 variablePoolBorrowRateUpdatedAt;	// @audit-issue

23:    uint64 variablePoolBorrowRateStaleRateInterval;	// @audit-issue
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L19-L19), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L21-L21), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L23-L23), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

44:    uint64 variablePoolBorrowRateStaleRateInterval;	// @audit-issue
```
[44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L44-L44), 


#### Recommendation

Minimize gas overhead by using 'uint256' or 'int256' instead of smaller integer types in Solidity contracts. The EVM operates more efficiently with 32-byte sizes. Downcast to smaller types only when necessary, as operations with smaller types like 'uint8' incur extra gas due to additional EVM operations for size adjustment.

### Avoid Unnecessary Public Variables
Public state variables in Solidity automatically generate getter functions, increasing contract size and potentially leading to higher deployment and interaction costs. To optimize gas usage and contract efficiency, minimize the use of public variables unless external access is necessary. Instead, use internal or private visibility combined with explicit getter functions when required. This practice not only reduces contract size but also provides better control over data access and manipulation, enhancing security and readability. Prioritize lean, efficient contracts to ensure cost-effectiveness and better performance on the blockchain.

```solidity
Path: ./src/oracle/PriceFeed.sol

28:    uint256 public constant decimals = 18;	// @audit-issue

29:    AggregatorV3Interface public immutable base;	// @audit-issue

30:    AggregatorV3Interface public immutable quote;	// @audit-issue

32:    uint256 public immutable baseStalePriceInterval;	// @audit-issue

33:    uint256 public immutable quoteStalePriceInterval;	// @audit-issue
```
[28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L28-L28), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L30-L30), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L33-L33), 


#### Recommendation

Avoid creating explicit getter functions for 'public' state variables in Solidity. The compiler automatically generates getters for such variables, making additional functions redundant. This practice helps reduce contract size, lowers deployment costs, and simplifies maintenance and understanding of the contract.

### Use `do while` loops intead of for loops
A `do while` loop will cost less gas since the condition is not being checked for the first iteration.
```solidity
uint256 i = 1;
do {                   
    param2 += i;
    i++;
}
while (i < 50);
``` 
is better than
```solidity
for(uint256 i = 1; i< 50; i++){
    param1 += i;
}
```


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

63:        for (uint256 i = self.tenors.length; i != 0; i--) {	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L63-L63), 


```solidity
Path: ./src/libraries/Multicall.sol

33:        for (uint256 i = 0; i < data.length; i++) {	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L33-L33), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

48:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue

69:        for (uint256 i = 0; i < params.creditPositionIds.length; i++) {	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L48-L48), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L69-L69), 


#### Recommendation

Where appropriate, consider using a `do while` loop instead of a `for` loop in your Solidity contracts. This is especially beneficial when the first iteration of the loop does not require a condition check. Refactor your loop logic to fit the `do while` structure for more gas-efficient execution. However, ensure that the loop's logic and termination conditions are correctly implemented to avoid infinite loops or other logical errors. Always balance gas efficiency with code readability and the specific requirements of your contract's logic.

### Using XOR (^) and AND (&) bitwise equivalents for gas optimizations
Given 4 variables a, b, c and d represented as such:
```
0 0 0 0 0 1 1 0 <- a
0 1 1 0 0 1 1 0 <- b
0 0 0 0 0 0 0 0 <- c
1 1 1 1 1 1 1 1 <- d
```
To have a == b means that every 0 and 1 match on both variables. Meaning that a XOR (operator ^) would evaluate to 0 ((a ^ b) == 0), as it excludes by definition any equalities.Now, if a != b, this means that there’s at least somewhere a 1 and a 0 not matching between a and b, making (a ^ b) != 0.Both formulas are logically equivalent and using the XOR bitwise operator costs actually the same amount of gas.However, it is much cheaper to use the bitwise OR operator (|) than comparing the truthy or falsy values.These are logically equivalent too, as the OR bitwise operator (|) would result in a 1 somewhere if any value is not 0 between the XOR (^) statements, meaning if any XOR (^) statement verifies that its arguments are different.


```solidity
Path: ./src/Size.sol

181:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue

191:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue
```
[181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L181-L181), [191](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L191-L191), 


```solidity
Path: ./src/SizeView.sol

180:        if (tenor == 0) {	// @audit-issue
```
[180](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L180-L180), 


```solidity
Path: ./src/oracle/PriceFeed.sol

43:        if (_base == address(0) || _quote == address(0)) {	// @audit-issue

48:        if (_baseStalePriceInterval == 0 || _quoteStalePriceInterval == 0) {	// @audit-issue

68:            if (answer == 1) {	// @audit-issue
```
[43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L43-L43), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L48-L48), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L68-L68), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

94:        return state.getLoanStatus(creditPositionId) == LoanStatus.ACTIVE	// @audit-issue

113:            || status == LoanStatus.OVERDUE	// @audit-issue
```
[94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L94-L94), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L113-L113), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

33:        return self.maxDueDate == 0 && self.curveRelativeTime.isNull();	// @audit-issue

53:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue

81:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L33-L33), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L53-L53), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L81-L81), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

133:        if (debtPosition.futureValue == 0) {	// @audit-issue
```
[133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L133-L133), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

39:        return self.tenors.length == 0 && self.aprs.length == 0 && self.marketRateMultipliers.length == 0;	// @audit-issue

51:        if (self.tenors.length == 0 || self.aprs.length == 0 || self.marketRateMultipliers.length == 0) {	// @audit-issue

92:        if (marketRateMultiplier == 0) {	// @audit-issue

95:            params.variablePoolBorrowRateStaleRateInterval == 0	// @audit-issue
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L39-L39), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L51-L51), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L92-L92), [95](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L95-L95), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

107:        if (exitCreditPosition.credit == credit) {	// @audit-issue

194:        if (creditAmountIn == maxCredit) {	// @audit-issue

245:        if (cashAmountOut == maxCashAmountOut) {	// @audit-issue

282:        if (cashAmountIn == maxCashAmountIn) {	// @audit-issue

318:        if (creditAmountOut == maxCredit) {	// @audit-issue
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L107-L107), [194](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L194-L194), [245](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L245-L245), [282](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L282-L282), [318](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L318-L318), 


```solidity
Path: ./src/libraries/Math.sol

59:            if (array[mid] == value) {	// @audit-issue
```
[59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L59-L59), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

56:        if (params.creditPositionToCompensateId == RESERVED_ID) {	// @audit-issue

86:            if (params.creditPositionToCompensateId == params.creditPositionWithDebtToRepayId) {	// @audit-issue

98:        if (amountToCompensate == 0) {	// @audit-issue

119:        if (params.creditPositionToCompensateId == RESERVED_ID) {	// @audit-issue

140:            exitCreditPositionId: params.creditPositionToCompensateId == RESERVED_ID	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L56-L56), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L86-L86), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L98-L98), [119](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L119-L119), [140](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L140-L140), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

64:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue

138:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue

164:                maxCredit: params.creditPositionId == RESERVED_ID ? creditAmountIn : creditPosition.credit,	// @audit-issue

173:                maxCashAmountOut: params.creditPositionId == RESERVED_ID	// @audit-issue

176:                maxCredit: params.creditPositionId == RESERVED_ID	// @audit-issue

184:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue

195:            exitCreditPositionId: params.creditPositionId == RESERVED_ID	// @audit-issue
```
[64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L64-L64), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L138-L138), [164](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L164-L164), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L173-L173), [176](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L176-L176), [184](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L184-L184), [195](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L195-L195), 


```solidity
Path: ./src/libraries/actions/Claim.sol

40:        if (creditPosition.credit == 0) {	// @audit-issue
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

63:        if (owner == address(0)) {	// @audit-issue

91:        if (f.feeRecipient == address(0)) {	// @audit-issue

113:        if (r.minimumCreditBorrowAToken == 0) {	// @audit-issue

121:        if (r.minTenor == 0) {	// @audit-issue

134:        if (o.priceFeed == address(0)) {	// @audit-issue

148:        if (d.underlyingCollateralToken == address(0)) {	// @audit-issue

156:        if (d.underlyingBorrowToken == address(0)) {	// @audit-issue

164:        if (d.variablePool == address(0)) {	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L63-L63), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L91-L91), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L113-L113), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L121-L121), [134](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L134-L134), [148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L148-L148), [156](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L156-L156), [164](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L164-L164), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

42:        if (params.amount == 0) {	// @audit-issue

47:        if (params.to == address(0)) {	// @audit-issue

54:        if (params.token == address(state.data.underlyingBorrowToken)) {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L42-L42), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L47-L47), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L54-L54), 


```solidity
Path: ./src/libraries/actions/Repay.sol

35:        if (state.getLoanStatus(params.debtPositionId) == LoanStatus.REPAID) {	// @audit-issue
```
[35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L35-L35), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

39:            if (params.maxDueDate == 0) {	// @audit-issue
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L39-L39), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

56:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue

133:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue

160:                maxCashAmountIn: params.creditPositionId == RESERVED_ID	// @audit-issue

163:                maxCredit: params.creditPositionId == RESERVED_ID	// @audit-issue

173:                maxCredit: params.creditPositionId == RESERVED_ID ? creditAmountOut : creditPosition.credit,	// @audit-issue

179:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L56-L56), [133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L133-L133), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L160-L160), [163](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L163-L163), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L173-L173), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L179-L179), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

54:        if (params.amount == 0) {	// @audit-issue

59:        if (params.to == address(0)) {	// @audit-issue

76:        if (params.token == address(state.data.underlyingBorrowToken)) {	// @audit-issue
```
[54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L54-L54), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L59-L59), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L76-L76), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

22:        if (decimals_ == 0) {	// @audit-issue

47:        return spender == owner() ? type(uint256).max : 0;	// @audit-issue
```
[22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L22-L22), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L47-L47), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

33:        if (address(variablePool_) == address(0) || address(underlyingToken_) == address(0)) {	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L33-L33), 


#### Recommendation

Review your Solidity contracts to identify any computations that are performed multiple times with the same inputs. Cache the results of these computations in local variables and reuse them within the function or across function calls if the state remains unchanged.

### Consider using `bytes32` rather than a `string`
Using the bytes types for fixed-length strings is more efficient than having the EVM have to incur the overhead of string processing. Consider whether the value needs to be a string. A good reason to keep it as a string would be if the variable is defined in an interface that this project does not own.


```solidity
Path: ./src/libraries/Errors.sol

21:    error INVALID_KEY(string key);	// @audit-issue
```
[21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L21-L21), 


```solidity
Path: ./src/libraries/Events.sol

23:    event UpdateConfig(string indexed key, uint256 value);	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L23-L23), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

23:    string key;	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L23-L23), 


#### Recommendation

For fixed-length strings, prefer using 'bytes32' over 'string' to leverage EVM efficiency and reduce overhead. Evaluate the necessity of using a string, especially if the variable isn't part of an externally defined interface.

### The result of a function call should be cached rather than re-calling the function
The function calls in solidity are expensive. If the same result of the same function calls are to be used several times, the result should be cached to reduce the gas consumption of repeated calls.        

```solidity
Path: ./src/oracle/PriceFeed.sol

58:        if (base.decimals() != quote.decimals()) {	// @audit-issue: Function call `decimals` is called multiple times at lines [59].
```
[58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L58-L58), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

147:                account, collateralRatio(state, account), openingLimitBorrowCR	// @audit-issue: Function call `collateralRatio` is called multiple times at lines [145].
```
[147](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L147-L147), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

207:            fees = getSwapFee(state, maxCashAmountOut, tenor) + state.feeConfig.fragmentationFee;	// @audit-issue: Function call `getSwapFee` is called multiple times at lines [197].

200:                revert Errors.NOT_ENOUGH_CASH(maxCashAmountOut, fees);	// @audit-issue: Function call `NOT_ENOUGH_CASH` is called multiple times at lines [210].

249:            fees = Math.mulDivUp(cashAmountOut, swapFeePercent, PERCENT);	// @audit-issue: Function call `mulDivUp` is called multiple times at lines [256].
```
[207](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L207-L207), [200](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L200-L200), [249](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L249-L249), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

55:                state.riskConfig.borrowATokenCap, state.data.borrowAToken.totalSupply()	// @audit-issue: Function call `totalSupply` is called multiple times at lines [53].
```
[55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L55-L55), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

62:        uint256 scaledAmount = aToken.scaledBalanceOf(address(this)) - scaledBalanceBefore;	// @audit-issue: Function call `scaledBalanceOf` is called multiple times at lines [57].

80:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue: Function call `scaledBalanceOf` is called multiple times at lines [85].
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L62-L62), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L80-L80), 


```solidity
Path: ./src/libraries/Multicall.sol

37:        uint256 borrowATokenSupplyAfter = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue: Function call `balanceOf` is called multiple times at lines [29].

38:        uint256 debtTokenSupplyAfter = state.data.debtToken.totalSupply();	// @audit-issue: Function call `totalSupply` is called multiple times at lines [30].
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L38-L38), 


```solidity
Path: ./src/libraries/Math.sol

55:            return (type(uint256).max, type(uint256).max);	// @audit-issue: Function call is called multiple times at line(s) [55].
```
[55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L55-L55), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

47:            revert Errors.LIQUIDATION_NOT_AT_LOSS(params.creditPositionId, state.collateralRatio(debtPosition.borrower));	// @audit-issue: Function call `collateralRatio` is called multiple times at lines [46, 42].
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L47-L47), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

77:                    < state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId).dueDate	// @audit-issue: Function call `getDebtPositionByCreditPositionId` is called multiple times at lines [67].
```
[77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L77-L77), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

177:                    ? Math.mulDivUp(cashAmountOut, PERCENT + ratePerTenor, PERCENT - state.getSwapFeePercent(tenor))	// @audit-issue: Function call `getSwapFeePercent` is called multiple times at lines [175].
```
[177](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L177-L177), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

114:                    params.value, Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)	// @audit-issue: Function call `mulDivDown` is called multiple times at lines [101, 104, 111].

103:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue: Function call `VALUE_GREATER_THAN_MAX` is called multiple times at lines [113].

121:                    params.value, Math.mulDivDown(PERCENT, YEAR, state.riskConfig.maxTenor)	// @audit-issue: Function call `mulDivDown` is called multiple times at lines [119].
```
[114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L114-L114), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L103-L103), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L121-L121), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

114:            revert Errors.NULL_AMOUNT();	// @audit-issue: Function call `NULL_AMOUNT` is called multiple times at lines [122].

149:            revert Errors.NULL_ADDRESS();	// @audit-issue: Function call `NULL_ADDRESS` is called multiple times at lines [165, 157].

152:            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingCollateralToken).decimals());	// @audit-issue: Function call `decimals` is called multiple times at lines [151].

151:        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {	// @audit-issue: Function call `IERC20Metadata` is called multiple times at lines [152].

160:            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingBorrowToken).decimals());	// @audit-issue: Function call `decimals` is called multiple times at lines [159].

159:        if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {	// @audit-issue: Function call `IERC20Metadata` is called multiple times at lines [160].

241:            string.concat("Size ", IERC20Metadata(state.data.underlyingCollateralToken).name()),	// @audit-issue: Function call `IERC20Metadata` is called multiple times at lines [242, 243].

249:            string.concat("Size Scaled ", IERC20Metadata(state.data.underlyingBorrowToken).name()),	// @audit-issue: Function call `name` is called multiple times at lines [255].

250:            string.concat("sza", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),	// @audit-issue: Function call `IERC20Metadata` is called multiple times at lines [257, 255, 249, 251, 256].

250:            string.concat("sza", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),	// @audit-issue: Function call `symbol` is called multiple times at lines [256].

251:            IERC20Metadata(state.data.underlyingBorrowToken).decimals()	// @audit-issue: Function call `decimals` is called multiple times at lines [257].
```
[114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L114-L114), [149](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L149-L149), [152](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L152-L152), [151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L151-L151), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L160-L160), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L159-L159), [241](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L241-L241), [249](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L249-L249), [250](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L250-L250), [250](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L250-L250), [251](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L251-L251), 


#### Recommendation

Cache the result of function calls in Solidity instead of making repeated calls to the same function. This practice significantly reduces gas consumption by minimizing costly function call operations.

### Avoid updating storage when the value hasn't changed
Manipulating storage in solidity is gas-intensive. It can be optimized by avoiding unnecessary storage updates when the new value equals the existing value. If the old value is equal to the new value, not re-storing the value will avoid a Gsreset (2900 gas), potentially at the expense of a Gcoldsload (2100 gas) or a Gwarmaccess (100 gas).

```solidity
Path: ./src/Size.sol

126:        state.oracle.variablePoolBorrowRate = borrowRate;	// @audit-issue
```
[126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L126-L126), 


#### Recommendation

Optimize gas usage by avoiding storage updates in Solidity when the new value is the same as the existing one. This prevents unnecessary gas expenditure from storage resets, balancing the cost against cold or warm storage access as needed.

### Avoid zero transfers calls
In Solidity, unnecessary operations can waste gas. For example, a transfer function without a zero amount check uses gas even if called with a zero amount, since the contract state remains unchanged. Implementing a zero amount check avoids these unnecessary function calls, saving gas and improving efficiency.

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

25:        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);	// @audit-issue

39:        underlyingCollateralToken.safeTransfer(to, amount);	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L25-L25), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L39-L39), 


#### Recommendation

Include a condition in your transfer functions to check for and prevent zero-value transfers. Implement a `require(amount > 0, "Transfer amount must be greater than zero");` statement at the beginning of the function. This preemptive check ensures that the function only proceeds with non-zero transfer amounts, avoiding wasteful operations and saving gas. Apply this optimization across all functions involving token transfers or similar operations to improve your contract's gas efficiency and operational effectiveness.

### Empty Blocks Should Be Removed Or Emit Something
The code should be refactored such that empty blocks no longer exist, or the block should do something useful, such as emitting an event or reverting. Empty blocks can introduce confusion and potential errors when the code is modified in the future.


```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue
80:        // validation is done at execution
81:    }
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L81), 


#### Recommendation

Remove empty blocks in Solidity code to avoid confusion and potential errors. If a block is necessary, ensure it performs a useful action, like emitting an event or reverting, to justify its presence and clarify its purpose.

### Overridden function has no body
In Solidity, when a function is overridden from a base contract, it's expected to have its own implementation or body. However, there are cases where an overridden function is left with an empty body, either intentionally or due to oversight. An empty body in an overridden function can be misleading and might indicate an incomplete implementation. If the function is intentionally left empty (for instance, in cases of interface adherence or specific design patterns), this should be clearly documented. Otherwise, it should be provided with a proper implementation to ensure the contract's functionality aligns with its intended design and behavior.

```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), 


#### Recommendation

Review all overridden functions in your Solidity contracts to ensure they have appropriate implementations. If an overridden function is intentionally left empty, include clear comments explaining the rationale. This approach prevents confusion and ensures that the contract's design and purpose are accurately conveyed. For functions that require an implementation, provide a meaningful body that aligns with the contract's intended functionality and the expectations set by the base contract or interface.

### Functions guaranteed to `revert` when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided.

```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue

132:    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue

137:    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L110), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L120), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L132-L132), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L137-L137), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

29:    function mint(address to, uint256 value) external virtual onlyOwner {	// @audit-issue

33:    function burn(address from, uint256 value) external virtual onlyOwner {	// @audit-issue

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L29-L29), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L33-L33), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue

50:    function mintScaled(address to, uint256 scaledAmount) external onlyOwner {	// @audit-issue

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue

64:    function burnScaled(address from, uint256 scaledAmount) external onlyOwner {	// @audit-issue

76:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L50-L50), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L64-L64), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L76-L76), 


#### Recommendation

Mark functions with access restrictions like 'onlyOwner' as 'payable' in Solidity. This reduces gas costs for legitimate callers by removing the compiler's checks for incoming payments, as the function will revert for unauthorized users anyway.

### Use calldata instead of memory for function arguments that do not get mutated
Mark data types as `calldata` instead of `memory` where possible. This makes it so that the data is not automatically loaded into memory. If the data passed into the function does not need to be changed (like updating values in an array), it can be passed in as `calldata`. The one exception to this is if the argument must later be passed into another function that takes an argument that specifies `memory` storage.

```solidity
Path: ./src/Size.sol

188:    function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {	// @audit-issue
```
[188](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L188-L188), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

32:    function isNull(LoanOffer memory self) internal pure returns (bool) {	// @audit-issue

39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {	// @audit-issue

48:    function getAPRByTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue

62:    function getRatePerTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue

76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L32-L32), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L39-L39), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L48-L48), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L62-L62), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L76-L76), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L90), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

148:    function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)	// @audit-issue

170:    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)	// @audit-issue
```
[148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L148-L148), [170](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L170-L170), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

38:    function isNull(YieldCurve memory self) internal pure returns (bool) {	// @audit-issue

50:    function validateYieldCurve(YieldCurve memory self, uint256 minTenor, uint256 maxTenor) internal pure {	// @audit-issue

87:    function getAdjustedAPR(int256 apr, uint256 marketRateMultiplier, VariablePoolBorrowRateParams memory params)	// @audit-issue

115:    function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L38-L38), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L50-L50), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L87-L87), [115](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L115-L115), 


```solidity
Path: ./src/libraries/Math.sol

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L51), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

70:    function validateInitializeFeeConfigParams(InitializeFeeConfigParams memory f) internal pure {	// @audit-issue

98:    function validateInitializeRiskConfigParams(InitializeRiskConfigParams memory r) internal pure {	// @audit-issue

132:    function validateInitializeOracleParams(InitializeOracleParams memory o) internal view {	// @audit-issue

146:    function validateInitializeDataParams(InitializeDataParams memory d) internal view {	// @audit-issue

175:    function validateInitialize(
176:        State storage,
177:        address owner,
178:        InitializeFeeConfigParams memory f,	// @audit-issue
179:        InitializeRiskConfigParams memory r,
180:        InitializeOracleParams memory o,
181:        InitializeDataParams memory d
182:    ) external view {

175:    function validateInitialize(
176:        State storage,
177:        address owner,
178:        InitializeFeeConfigParams memory f,
179:        InitializeRiskConfigParams memory r,	// @audit-issue
180:        InitializeOracleParams memory o,
181:        InitializeDataParams memory d
182:    ) external view {

175:    function validateInitialize(
176:        State storage,
177:        address owner,
178:        InitializeFeeConfigParams memory f,
179:        InitializeRiskConfigParams memory r,
180:        InitializeOracleParams memory o,	// @audit-issue
181:        InitializeDataParams memory d
182:    ) external view {

175:    function validateInitialize(
176:        State storage,
177:        address owner,
178:        InitializeFeeConfigParams memory f,
179:        InitializeRiskConfigParams memory r,
180:        InitializeOracleParams memory o,
181:        InitializeDataParams memory d	// @audit-issue
182:    ) external view {

193:    function executeInitializeFeeConfig(State storage state, InitializeFeeConfigParams memory f) internal {	// @audit-issue

207:    function executeInitializeRiskConfig(State storage state, InitializeRiskConfigParams memory r) internal {	// @audit-issue

222:    function executeInitializeOracle(State storage state, InitializeOracleParams memory o) internal {	// @audit-issue

230:    function executeInitializeData(State storage state, InitializeDataParams memory d) internal {	// @audit-issue

267:    function executeInitialize(
268:        State storage state,
269:        InitializeFeeConfigParams memory f,	// @audit-issue
270:        InitializeRiskConfigParams memory r,
271:        InitializeOracleParams memory o,
272:        InitializeDataParams memory d
273:    ) external {

267:    function executeInitialize(
268:        State storage state,
269:        InitializeFeeConfigParams memory f,
270:        InitializeRiskConfigParams memory r,	// @audit-issue
271:        InitializeOracleParams memory o,
272:        InitializeDataParams memory d
273:    ) external {

267:    function executeInitialize(
268:        State storage state,
269:        InitializeFeeConfigParams memory f,
270:        InitializeRiskConfigParams memory r,
271:        InitializeOracleParams memory o,	// @audit-issue
272:        InitializeDataParams memory d
273:    ) external {

267:    function executeInitialize(
268:        State storage state,
269:        InitializeFeeConfigParams memory f,
270:        InitializeRiskConfigParams memory r,
271:        InitializeOracleParams memory o,
272:        InitializeDataParams memory d	// @audit-issue
273:    ) external {
```
[70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L70-L70), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L98-L98), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L132-L132), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L146-L146), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L182), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L182), [180](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L182), [181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L182), [193](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L193-L193), [207](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L207-L207), [222](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L222-L222), [230](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L230-L230), [269](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L273), [270](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L273), [271](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L273), [272](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L273), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

121:    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)	// @audit-issue
```
[121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L121-L121), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

25:    constructor(
26:        IPool variablePool_,
27:        IERC20Metadata underlyingToken_,
28:        address owner_,
29:        string memory name_,	// @audit-issue
30:        string memory symbol_,
31:        uint8 decimals_
32:    ) NonTransferrableToken(owner_, name_, symbol_, decimals_) {

25:    constructor(
26:        IPool variablePool_,
27:        IERC20Metadata underlyingToken_,
28:        address owner_,
29:        string memory name_,
30:        string memory symbol_,	// @audit-issue
31:        uint8 decimals_
32:    ) NonTransferrableToken(owner_, name_, symbol_, decimals_) {
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L32), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L32), 


#### Recommendation

To optimize gas usage in your Solidity functions, mark data types as `calldata` instead of `memory` wherever applicable. This prevents unnecessary data loading into memory. Use `calldata` for function arguments that do not require changes within the function, except when passing them into another function that explicitly requires `memory` storage.

### Nesting `if` statements that uses `&&` saves gas
In Solidity, the way conditional checks are structured can impact the gas consumption of a transaction. When conditions are combined using `&&` within an `if` statement, Solidity short-circuits the evaluation, meaning that if the first condition is `false`, the subsequent conditions won't be evaluated. This behavior can lead to gas savings compared to using separate nested `if` statements because not all conditions might need to be checked every time. By efficiently structuring these conditional checks, contracts can optimize the gas required for execution, leading to reduced costs for users.


```solidity
Path: ./src/libraries/RiskLibrary.sol

22:        if (0 < credit && credit < state.riskConfig.minimumCreditBorrowAToken) {	// @audit-issue
```
[22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L22-L22), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

100:                state.feeConfig.swapFeeAPR != 0	// @audit-issue
101:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)

110:                state.feeConfig.swapFeeAPR != 0	// @audit-issue
111:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
```
[100](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L100-L101), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L110-L111), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

35:            params.token != address(state.data.underlyingCollateralToken)	// @audit-issue
36:                && params.token != address(state.data.underlyingBorrowToken)
```
[35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L35-L36), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

41:        if (msg.value != 0 && (msg.value != params.amount || params.token != address(state.data.weth))) {	// @audit-issue

47:            params.token != address(state.data.underlyingCollateralToken)	// @audit-issue
48:                && params.token != address(state.data.underlyingBorrowToken)
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L41-L41), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L47-L48), 


#### Recommendation


When multiple conditions need to be checked successively, try to combine them in a single `if` statement using `&&` instead of nesting separate `if` statements. This will leverage short-circuit evaluation for potential gas savings.


### Use `!= 0` Instead of `> 0` for Unsigned Integer Comparison
In Solidity, unsigned integers (e.g., `uint256`, `uint8`, etc.) represent non-negative whole numbers, ranging from 0 to a maximum value determined by their bit size. When performing comparisons on these numbers, especially to check if they are non-zero, developers have options. A common practice is to compare against zero using the `>` operator, as in `value > 0`. However, given the nature of unsigned integers, a more straightforward and slightly gas-efficient comparison is to use the `!=` operator, as in `value != 0`.

The primary rationale is that the `!=` comparison directly checks for non-equality, whereas the `>` comparison checks if one value is strictly greater than another. For unsigned integers, where negative values don't exist, the `!= 0` check is both semantically clearer and potentially more efficient at the EVM bytecode level.


```solidity
Path: ./src/libraries/actions/Compensate.sol

146:        if (exiterCreditRemaining > 0) {	// @audit-issue
```
[146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L146-L146), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

56:            if (amount > 0) {	// @audit-issue

61:            if (amount > 0) {	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L56-L56), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L61-L61), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

67:        if (msg.value > 0) {	// @audit-issue
```
[67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L67-L67), 


#### Recommendation

Use '!= 0' instead of '> 0' for comparing unsigned integers in Solidity. This is semantically clearer and slightly more gas-efficient, as it directly checks for non-zero values without considering unnecessary relational comparisons.

### Operator `>=`/`<=` costs less gas than operator `>`/`<`
The compiler uses opcodes `GT` and `ISZERO` for code that uses `>`, but only requires `LT` for `>=`. A similar behaviour applies for `>`, which uses opcodes `LT` and `ISZERO`, but only requires `GT` for `<=`.


```solidity
Path: ./src/libraries/actions/Initialize.sol

151:        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {	// @audit-issue

159:        if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {	// @audit-issue
```
[151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L151-L151), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L159-L159), 


#### Recommendation

Opt for '>=', '<=' operators over '>' and '<' in Solidity where logically appropriate, as they consume less gas. This is because '>=' and '<=' require only one opcode ('LT' or 'GT'), compared to the two opcodes ('GT'/'LT' and 'ISZERO') needed for '>' and '<'.

### Constructor Can Be Marked As Payable
`payable` functions cost less gas to execute, since the compiler does not have to add extra checks to ensure that a payment wasn't provided.

A `constructor` can safely be marked as `payable`, since only the deployer would be able to pass funds, and the project itself would not pass any funds.



```solidity
Path: ./src/Size.sol

83:    constructor() {	// @audit-issue
```
[83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L83-L83), 


```solidity
Path: ./src/oracle/PriceFeed.sol

36:    constructor(	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L36-L36), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

25:    constructor(	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L25), 


#### Recommendation

Mark constructors as 'payable' in Solidity contracts to reduce gas costs, as this eliminates the need for the compiler to add checks against incoming payments. This is safe because only the deployer can send funds during contract creation, and typically no funds are sent at this stage.

### Initializers Can Be Marked As Payable
`payable` functions cost less gas to execute, since the compiler does not have to add extra checks to ensure that a payment wasn't provided.

A `initializer` can safely be marked as `payable`, since only the deployer would be able to pass funds, and the project itself would not pass any funds.



```solidity
Path: ./src/Size.sol

87:    function initialize(	// @audit-issue
```
[87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L87), 


#### Recommendation

Mark initializers as 'payable' in Solidity contracts to reduce gas costs, as this eliminates the need for the compiler to add checks against incoming payments. This is safe because only the deployer can send funds during contract creation, and typically no funds are sent at this stage.

### Use `selfbalance()` instead of `address(this).balance`
In Solidity, contracts often need to query their own Ether balance. While `address(this).balance` has been a widely used approach to retrieve the current contract's balance, it is not the most gas-efficient method, especially with the introduction of the `SELFBALANCE` opcode in more recent EVM versions.

The `SELFBALANCE` opcode provides a more gas-efficient way to obtain the balance of the current contract. In Solidity, this can be invoked using the `selfbalance()` function. Compared to the `BALANCE` opcode used by `address(this).balance`, `SELFBALANCE` offers significant gas savings:

- `BALANCE`: The static gas cost is 0. If the accessed address is warm, the dynamic gas cost is 100. If the address is cold, the dynamic cost is 2,600.
  
- `SELFBALANCE`: Semantically equivalent to the `BALANCE` opcode when called on the contract's own address, but with a dramatically reduced minimum gas cost of 5.

Switching to `selfbalance()` can lead to significant gas savings, especially in operations or functions that frequently check the contract's balance.


```solidity
Path: ./src/libraries/actions/Deposit.sol

69:            amount = address(this).balance;	// @audit-issue
```
[69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L69-L69), 


#### Recommendation

To optimize gas usage when querying a contract's own Ether balance in Solidity, it's recommended to use the `selfbalance()` function, which utilizes the more gas-efficient `SELFBALANCE` opcode. This can result in significant gas savings compared to `address(this).balance`.

### Optimize names to save gas
`public`/`external` function names and `public` member variable names can be optimized to save gas. Below are the interfaces/abstract contracts that can be optimized so that the most frequently-called functions use the least amount of gas possible during method lookup. Method IDs that have two leading zero bytes can save 128 gas each during deployment, and renaming functions to have lower method IDs will save 22 gas per call, [per sorted position shifted](https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92).


```solidity
Path: ./src/Size.sol

62:contract Size is ISize, SizeView, Initializable, AccessControlUpgradeable, PausableUpgradeable, UUPSUpgradeable {	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L62-L62), 


```solidity
Path: ./src/SizeView.sol

37:abstract contract SizeView is SizeStorage, ISizeView {	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L37-L37), 


```solidity
Path: ./src/oracle/IPriceFeed.sol

7:interface IPriceFeed {	// @audit-issue
```
[7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L7-L7), 


```solidity
Path: ./src/oracle/PriceFeed.sol

24:contract PriceFeed is IPriceFeed {	// @audit-issue
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L24-L24), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

14:library RiskLibrary {	// @audit-issue
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L14-L14), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

56:library LoanLibrary {	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L56-L56), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

34:library YieldCurveLibrary {	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L34-L34), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

16:library AccountingLibrary {	// @audit-issue
```
[16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L16-L16), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

11:library CapsLibrary {	// @audit-issue
```
[11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L11-L11), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

15:library DepositTokenLibrary {	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L15-L15), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

25:library SetUserConfiguration {	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L25-L25), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

24:library SelfLiquidate {	// @audit-issue
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L24-L24), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

31:library Compensate {	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

40:library SellCreditMarket {	// @audit-issue
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

36:library UpdateConfig {	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L36-L36), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

29:library Liquidate {	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L29-L29), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

36:library LiquidateWithReplacement {	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L36-L36), 


```solidity
Path: ./src/libraries/actions/Claim.sol

23:library Claim {	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L23-L23), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

59:library Initialize {	// @audit-issue
```
[59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L59-L59), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

26:library Withdraw {	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L26-L26), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

19:library SellCreditLimit {	// @audit-issue
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L19-L19), 


```solidity
Path: ./src/libraries/actions/Repay.sol

24:library Repay {	// @audit-issue
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L24-L24), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

23:library BuyCreditLimit {	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L23-L23), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

40:library BuyCreditMarket {	// @audit-issue
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

29:library Deposit {	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L29-L29), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

14:contract NonTransferrableToken is Ownable, ERC20 {	// @audit-issue
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L14-L14), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

19:contract NonTransferrableScaledToken is NonTransferrableToken {	// @audit-issue
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L19-L19), 


#### Recommendation

Optimize gas usage by renaming 'public'/'external' functions and 'public' member variables in Solidity. Aim for shorter and more efficient names, especially for frequently called functions. This can save gas during deployment and reduce gas costs per call due to lower method ID sorting positions.

### Use `uint256(1)`/`uint256(2)` instead of `true`/`false` to save gas for changes
Avoids a Gsset (**20000 gas**) when changing from `false` to `true`, after having been `true` in the past. Since most of the bools aren't changed twice in one transaction, I've counted the amount of gas as half of the full amount, for each variable. Note that public state variables can be re-written to be `private` and use `uint256`, but have public getters returning `bool`s.

```solidity
Path: ./src/SizeStorage.sol

24:    bool allCreditPositionsForSaleDisabled;	// @audit-issue

94:    bool isMulticall;	// @audit-issue
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L24-L24), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L94-L94), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

36:    bool forSale;	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L36-L36), 


```solidity
Path: ./src/libraries/Events.sol

31:        bool exactAmountIn	// @audit-issue

43:        bool exactAmountIn	// @audit-issue

65:        bool indexed allCreditPositionsForSaleDisabled,	// @audit-issue

66:        bool indexed creditPositionIdsForSale,	// @audit-issue

92:    event UpdateCreditPosition(uint256 indexed creditPositionId, address indexed lender, uint256 credit, bool forSale);	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L31-L31), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L43-L43), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L65-L65), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L66-L66), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L92-L92), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

15:    bool allCreditPositionsForSaleDisabled;	// @audit-issue

17:    bool creditPositionIdsForSale;	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L15-L15), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L17-L17), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

33:    bool exactAmountIn;	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L33-L33), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

33:    bool exactAmountIn;	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L33-L33), 


#### Recommendation

To minimize gas overhead in your Solidity contracts, consider using `uint256(1)` and `uint256(2)` to represent `true` and `false`, respectively, instead of `bool` types for storage. This approach avoids additional `SLOAD` and `SSTORE` operations, resulting in more gas-efficient code.

### Consider activating `via-ir` for deploying
The IR-based code generator was developed to make code generation more performant by enabling optimization passes that can be applied across functions.

It is possible to activate the IR-based code generator through the command line by using the flag `--via-ir`or by including the option `{"viaIR": true}`.

Keep in mind that compiling with this option may take longer. However, you can simply test it before deploying your code. If you find that it provides better performance, you can add the `--via-ir` flag to your deploy command.
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

Consider activating `via-ir`.

### Optimize Deployment Size by Fine-tuning IPFS Hash
The Solidity compiler appends 53 bytes of metadata to the smart contract code, incurring an extra cost of 10,600 gas. This additional expense arises from 200 gas per bytecode, plus calldata cost, which amounts to 16 gas for non-zero bytes and 4 gas for zero bytes. This results in a maximum of 848 extra gas in calldata cost.

Reducing this cost is crucial for the following reasons:

The metadata's 53-byte addition leads to a deployment cost increase of 10,600 gas. It can also result in an additional calldata cost of up to 848 gas. Ways to Minimize Gas Consumption:

Employ the `--no-cbor-metadata` compiler option to exclude metadata. Be cautious as this might impact contract verification. Search for code comments that yield an IPFS hash with more zeros, thereby reducing calldata costs.

```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

To optimize deployment size and reduce associated costs, consider the following strategies:
1. **Exclude Metadata with Compiler Option**: Use the `solc` compiler’s `--metadata-hash none` or `--no-cbor-metadata` option to prevent the inclusion of metadata in the compiled bytecode. This action reduces the bytecode size, thus lowering deployment gas costs. However, exercise caution with this approach, as it might affect the ability to verify the contract on platforms like Etherscan.

2. **Optimize IPFS Hash for More Zeros**: If excluding metadata is not desirable, another approach involves optimizing code comments or elements that influence the metadata hash generation to achieve an IPFS hash with a higher proportion of zeros. Since calldata costs are lower for zero bytes, a metadata hash with more zeros can reduce the calldata costs associated with contract interactions.

Example for excluding metadata:
```bash
solc --metadata-hash none YourContract.sol
```


### Assembly: Use scratch space for building calldata
If an external call's calldata can fit into two or fewer words, use the scratch space to build the calldata, rather than allowing Solidity to do a memory expansion.

```solidity
Path: ./src/Size.sol

115:        state.validateUpdateConfig(params);	// @audit-issue

116:        state.executeUpdateConfig(params);	// @audit-issue

128:        emit Events.VariablePoolBorrowRateUpdated(oldBorrowRate, borrowRate);	// @audit-issue

149:        results = state.multicall(_data);	// @audit-issue

154:        state.validateDeposit(params);	// @audit-issue

155:        state.executeDeposit(params);	// @audit-issue

160:        state.validateWithdraw(params);	// @audit-issue

161:        state.executeWithdraw(params);	// @audit-issue

162:        state.validateUserIsNotBelowOpeningLimitBorrowCR(msg.sender);	// @audit-issue

167:        state.validateBuyCreditLimit(params);	// @audit-issue

168:        state.executeBuyCreditLimit(params);	// @audit-issue

173:        state.validateSellCreditLimit(params);	// @audit-issue

174:        state.executeSellCreditLimit(params);	// @audit-issue

179:        state.validateBuyCreditMarket(params);	// @audit-issue

180:        uint256 amount = state.executeBuyCreditMarket(params);	// @audit-issue

182:            state.validateUserIsNotBelowOpeningLimitBorrowCR(params.borrower);	// @audit-issue

184:        state.validateVariablePoolHasEnoughLiquidity(amount);	// @audit-issue

189:        state.validateSellCreditMarket(params);	// @audit-issue

190:        uint256 amount = state.executeSellCreditMarket(params);	// @audit-issue

192:            state.validateUserIsNotBelowOpeningLimitBorrowCR(msg.sender);	// @audit-issue

194:        state.validateVariablePoolHasEnoughLiquidity(amount);	// @audit-issue

199:        state.validateRepay(params);	// @audit-issue

200:        state.executeRepay(params);	// @audit-issue

205:        state.validateClaim(params);	// @audit-issue

206:        state.executeClaim(params);	// @audit-issue

217:        state.validateLiquidate(params);	// @audit-issue

218:        liquidatorProfitCollateralToken = state.executeLiquidate(params);	// @audit-issue

219:        state.validateMinimumCollateralProfit(params, liquidatorProfitCollateralToken);	// @audit-issue

224:        state.validateSelfLiquidate(params);	// @audit-issue

225:        state.executeSelfLiquidate(params);	// @audit-issue

237:        state.validateLiquidateWithReplacement(params);	// @audit-issue

240:            state.executeLiquidateWithReplacement(params);	// @audit-issue

241:        state.validateUserIsNotBelowOpeningLimitBorrowCR(params.borrower);	// @audit-issue

242:        state.validateMinimumCollateralProfit(params, liquidatorProfitCollateralToken);	// @audit-issue

243:        state.validateVariablePoolHasEnoughLiquidity(amount);	// @audit-issue

248:        state.validateCompensate(params);	// @audit-issue

249:        state.executeCompensate(params);	// @audit-issue

250:        state.validateUserIsNotUnderwater(msg.sender);	// @audit-issue

260:        state.validateSetUserConfiguration(params);	// @audit-issue

261:        state.executeSetUserConfiguration(params);	// @audit-issue
```
[115](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L115-L115), [116](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L116-L116), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L128-L128), [149](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L149-L149), [154](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L154-L154), [155](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L155-L155), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L160-L160), [161](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L161-L161), [162](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L162-L162), [167](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L167-L167), [168](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L168-L168), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L173-L173), [174](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L174-L174), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L179-L179), [180](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L180-L180), [182](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L182-L182), [184](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L184-L184), [189](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L189-L189), [190](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L190-L190), [192](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L192-L192), [194](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L194-L194), [199](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L199-L199), [200](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L200-L200), [205](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L205-L205), [206](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L206-L206), [217](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L217-L217), [218](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L218-L218), [219](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L219-L219), [224](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L224-L224), [225](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L225-L225), [237](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L237-L237), [240](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L240-L240), [241](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L241-L241), [242](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L242-L242), [243](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L243-L243), [248](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L248-L248), [249](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L249-L249), [250](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L250-L250), [260](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L260-L260), [261](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L261-L261), 


```solidity
Path: ./src/SizeView.sol

49:        return state.collateralRatio(user);	// @audit-issue

54:        return state.isUserUnderwater(user);	// @audit-issue

59:        return state.isDebtPositionLiquidatable(debtPositionId);	// @audit-issue

64:        return state.debtTokenAmountToCollateralTokenAmount(borrowATokenAmount);	// @audit-issue

69:        return state.feeConfigParams();	// @audit-issue

74:        return state.riskConfigParams();	// @audit-issue

79:        return state.oracleParams();	// @audit-issue

101:            collateralTokenBalance: state.data.collateralToken.balanceOf(user),	// @audit-issue

102:            borrowATokenBalance: state.data.borrowAToken.balanceOf(user),	// @audit-issue

103:            debtBalance: state.data.debtToken.balanceOf(user)	// @audit-issue

109:        return state.isDebtPositionId(debtPositionId);	// @audit-issue

114:        return state.isCreditPositionId(creditPositionId);	// @audit-issue

119:        return state.getDebtPosition(debtPositionId);	// @audit-issue

124:        return state.getCreditPosition(creditPositionId);	// @audit-issue

129:        return state.getLoanStatus(positionId);	// @audit-issue

143:        if (offer.isNull()) {	// @audit-issue

144:            revert Errors.NULL_OFFER();	// @audit-issue

146:        return offer.getAPRByTenor(	// @audit-issue

159:        if (offer.isNull()) {	// @audit-issue

160:            revert Errors.NULL_OFFER();	// @audit-issue

162:        return offer.getAPRByTenor(	// @audit-issue

174:        DebtPosition memory debtPosition = state.getDebtPosition(debtPositionId);	// @audit-issue

175:        return state.getDebtPositionAssignedCollateral(debtPosition);	// @audit-issue

181:            revert Errors.NULL_TENOR();	// @audit-issue

183:        return state.getSwapFee(cash, tenor);	// @audit-issue
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L49-L49), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L54-L54), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L59-L59), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L64-L64), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L69-L69), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L74-L74), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L79-L79), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L101-L101), [102](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L102-L102), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L103-L103), [109](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L109-L109), [114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L114-L114), [119](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L119-L119), [124](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L124-L124), [129](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L129-L129), [143](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L143-L143), [144](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L144-L144), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L146-L146), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L159-L159), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L160-L160), [162](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L162-L162), [174](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L174-L174), [175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L175-L175), [181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L181-L181), [183](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L183-L183), 


```solidity
Path: ./src/oracle/PriceFeed.sol

44:            revert Errors.NULL_ADDRESS();	// @audit-issue

49:            revert Errors.NULL_STALE_PRICE();	// @audit-issue

58:        if (base.decimals() != quote.decimals()) {	// @audit-issue

59:            revert Errors.INVALID_DECIMALS(quote.decimals());	// @audit-issue

66:            (, int256 answer, uint256 startedAt,,) = sequencerUptimeFeed.latestRoundData();	// @audit-issue

70:                revert Errors.SEQUENCER_DOWN();	// @audit-issue

75:                revert Errors.GRACE_PERIOD_NOT_OVER();	// @audit-issue

86:        (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();	// @audit-issue

88:        if (price <= 0) revert Errors.INVALID_PRICE(address(aggregator), price);	// @audit-issue

90:            revert Errors.STALE_PRICE(address(aggregator), updatedAt);	// @audit-issue

93:        return SafeCast.toUint256(price);	// @audit-issue
```
[44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L44-L44), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L49-L49), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L58-L58), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L59-L59), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L66-L66), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L70-L70), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L75-L75), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L86-L86), [88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L88-L88), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L90-L90), [93](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L93-L93), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

23:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(credit, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

33:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT_OPENING(credit, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

54:        uint256 collateral = state.data.collateralToken.balanceOf(account);	// @audit-issue

55:        uint256 debt = state.data.debtToken.balanceOf(account);	// @audit-issue

56:        uint256 debtWad = Math.amountToWad(debt, state.data.underlyingBorrowToken.decimals());	// @audit-issue

57:        uint256 price = state.oracle.priceFeed.getPrice();	// @audit-issue

78:        LoanStatus status = state.getLoanStatus(creditPositionId);	// @audit-issue

80:        return state.isCreditPositionId(creditPositionId)	// @audit-issue

94:        return state.getLoanStatus(creditPositionId) == LoanStatus.ACTIVE	// @audit-issue

95:            && !isUserUnderwater(state, state.getDebtPositionByCreditPositionId(creditPositionId).borrower);	// @audit-issue

106:        LoanStatus status = state.getLoanStatus(debtPositionId);	// @audit-issue

108:        return state.isDebtPositionId(debtPositionId)	// @audit-issue

131:            revert Errors.USER_IS_UNDERWATER(account, collateralRatio(state, account));	// @audit-issue

141:        uint256 openingLimitBorrowCR = Math.max(	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L23-L23), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L33-L33), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L54-L54), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L55-L55), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L56-L56), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L57-L57), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L78-L78), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L80-L80), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L94-L94), [95](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L95-L95), [106](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L106-L106), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L108-L108), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L131-L131), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L141-L141), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

33:        return self.maxDueDate == 0 && self.curveRelativeTime.isNull();	// @audit-issue

40:        return self.curveRelativeTime.isNull();	// @audit-issue

53:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue

68:        return Math.aprToRatePerTenor(apr, tenor);	// @audit-issue

81:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue

96:        return Math.aprToRatePerTenor(apr, tenor);	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L33-L33), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L40-L40), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L53-L53), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L68-L68), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L81-L81), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L96-L96), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

84:            revert Errors.INVALID_DEBT_POSITION_ID(debtPositionId);	// @audit-issue

101:            revert Errors.INVALID_CREDIT_POSITION_ID(creditPositionId);	// @audit-issue

130:            revert Errors.INVALID_POSITION_ID(positionId);	// @audit-issue

153:        uint256 debt = state.data.debtToken.balanceOf(debtPosition.borrower);	// @audit-issue

154:        uint256 collateral = state.data.collateralToken.balanceOf(debtPosition.borrower);	// @audit-issue
```
[84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L84-L84), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L101-L101), [130](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L130-L130), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L153-L153), [154](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L154-L154), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

52:            revert Errors.NULL_ARRAY();	// @audit-issue

55:            revert Errors.ARRAY_LENGTHS_MISMATCH();	// @audit-issue

65:                revert Errors.TENORS_NOT_STRICTLY_INCREASING();	// @audit-issue

93:            return SafeCast.toUint256(apr);	// @audit-issue

101:            revert Errors.STALE_RATE(params.variablePoolBorrowRateUpdatedAt);	// @audit-issue

103:            return SafeCast.toUint256(	// @audit-issue

104:                apr + SafeCast.toInt256(Math.mulDivDown(params.variablePoolBorrowRate, marketRateMultiplier, PERCENT))	// @audit-issue

124:            (uint256 low, uint256 high) = Math.binarySearch(curveRelativeTime.tenors, tenor);	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L52-L52), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L55-L55), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L65-L65), [93](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L93-L93), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L101-L101), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L103-L103), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L104-L104), [124](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L124-L124), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

31:        uint256 debtTokenAmountWad = Math.amountToWad(debtTokenAmount, state.data.underlyingBorrowToken.decimals());	// @audit-issue

33:            debtTokenAmountWad, 10 ** state.oracle.priceFeed.decimals(), state.oracle.priceFeed.getPrice()	// @audit-issue

43:        DebtPosition storage debtPosition = state.getDebtPosition(debtPositionId);	// @audit-issue

45:        state.data.debtToken.burn(debtPosition.borrower, repayAmount);	// @audit-issue

86:        state.validateMinimumCreditOpening(creditPosition.credit);	// @audit-issue

87:        state.validateTenor(dueDate - block.timestamp);	// @audit-issue

91:        state.data.debtToken.mint(borrower, futureValue);	// @audit-issue

106:        CreditPosition storage exitCreditPosition = state.getCreditPosition(exitCreditPositionId);	// @audit-issue

123:            state.validateMinimumCreditOpening(creditPosition.credit);	// @audit-issue

138:        CreditPosition storage creditPosition = state.getCreditPosition(creditPositionId);	// @audit-issue

140:        state.validateMinimumCredit(creditPosition.credit);	// @audit-issue

200:                revert Errors.NOT_ENOUGH_CASH(maxCashAmountOut, fees);	// @audit-issue

210:                revert Errors.NOT_ENOUGH_CASH(maxCashAmountOut, fees);	// @audit-issue

215:            revert Errors.NOT_ENOUGH_CREDIT(creditAmountIn, maxCredit);	// @audit-issue

261:            revert Errors.NOT_ENOUGH_CASH(maxCashAmountOutFragmentation, cashAmountOut);	// @audit-issue

291:                revert Errors.NOT_ENOUGH_CASH(state.feeConfig.fragmentationFee, cashAmountIn);	// @audit-issue

299:            revert Errors.NOT_ENOUGH_CREDIT(maxCashAmountIn, cashAmountIn);	// @audit-issue

331:            revert Errors.NOT_ENOUGH_CREDIT(creditAmountOut, maxCredit);	// @audit-issue

335:            revert Errors.NOT_ENOUGH_CASH(cashAmountIn, fees);	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L31-L31), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L33-L33), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L43-L43), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L45-L45), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L86-L86), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L87-L87), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L91-L91), [106](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L106-L106), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L123-L123), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L138-L138), [140](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L140-L140), [200](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L200-L200), [210](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L210-L210), [215](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L215-L215), [261](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L261-L261), [291](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L291-L291), [299](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L299-L299), [331](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L331-L331), [335](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L335-L335), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

37:                revert Errors.BORROW_ATOKEN_INCREASE_EXCEEDS_DEBT_TOKEN_DECREASE(	// @audit-issue

53:        if (state.data.borrowAToken.totalSupply() > state.riskConfig.borrowATokenCap) {	// @audit-issue

54:            revert Errors.BORROW_ATOKEN_CAP_EXCEEDED(	// @audit-issue

55:                state.riskConfig.borrowATokenCap, state.data.borrowAToken.totalSupply()	// @audit-issue

68:        uint256 liquidity = state.data.underlyingBorrowToken.balanceOf(address(state.data.variablePool));	// @audit-issue

70:            revert Errors.NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(liquidity, amount);	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L37-L37), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L53-L53), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L54-L54), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L55-L55), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L68-L68), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L70-L70), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

26:        state.data.collateralToken.mint(to, amount);	// @audit-issue

38:        state.data.collateralToken.burn(from, amount);	// @audit-issue

39:        underlyingCollateralToken.safeTransfer(to, amount);	// @audit-issue

55:            IAToken(state.data.variablePool.getReserveData(address(state.data.underlyingBorrowToken)).aTokenAddress);	// @audit-issue

57:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue

59:        state.data.underlyingBorrowToken.forceApprove(address(state.data.variablePool), amount);	// @audit-issue

62:        uint256 scaledAmount = aToken.scaledBalanceOf(address(this)) - scaledBalanceBefore;	// @audit-issue

64:        state.data.borrowAToken.mintScaled(to, scaledAmount);	// @audit-issue

78:            IAToken(state.data.variablePool.getReserveData(address(state.data.underlyingBorrowToken)).aTokenAddress);	// @audit-issue

80:        uint256 scaledBalanceBefore = aToken.scaledBalanceOf(address(this));	// @audit-issue

85:        uint256 scaledAmount = scaledBalanceBefore - aToken.scaledBalanceOf(address(this));	// @audit-issue

87:        state.data.borrowAToken.burnScaled(from, scaledAmount);	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L26-L26), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L39-L39), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L55-L55), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L57-L57), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L59-L59), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L62-L62), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L64-L64), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L78-L78), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L80-L80), [85](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L85-L85), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L87-L87), 


```solidity
Path: ./src/libraries/Multicall.sol

29:        uint256 borrowATokenSupplyBefore = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue

30:        uint256 debtTokenSupplyBefore = state.data.debtToken.totalSupply();	// @audit-issue

34:            results[i] = Address.functionDelegateCall(address(this), data[i]);	// @audit-issue

37:        uint256 borrowATokenSupplyAfter = state.data.borrowAToken.balanceOf(address(this));	// @audit-issue

38:        uint256 debtTokenSupplyAfter = state.data.debtToken.totalSupply();	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L30-L30), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L34-L34), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L38-L38), 


```solidity
Path: ./src/libraries/Math.sol

16:        return FixedPointMathLib.min(a, b);	// @audit-issue

20:        return FixedPointMathLib.max(a, b);	// @audit-issue
```
[16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L16-L16), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L20-L20), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

49:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionIds[i]);	// @audit-issue

51:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionIds[i]);	// @audit-issue

54:            if (state.getLoanStatus(params.creditPositionIds[i]) != LoanStatus.ACTIVE) {	// @audit-issue

55:                revert Errors.LOAN_NOT_ACTIVE(params.creditPositionIds[i]);	// @audit-issue

70:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionIds[i]);	// @audit-issue
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L49-L49), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L51-L51), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L54-L54), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L55-L55), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L70-L70), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

35:        CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

36:        DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

39:        if (!state.isCreditPositionSelfLiquidatable(params.creditPositionId)) {	// @audit-issue

42:                state.collateralRatio(debtPosition.borrower),	// @audit-issue

43:                state.getLoanStatus(params.creditPositionId)	// @audit-issue

46:        if (state.collateralRatio(debtPosition.borrower) >= PERCENT) {	// @audit-issue

47:            revert Errors.LIQUIDATION_NOT_AT_LOSS(params.creditPositionId, state.collateralRatio(debtPosition.borrower));	// @audit-issue

52:            revert Errors.LIQUIDATOR_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue

60:        emit Events.SelfLiquidate(params.creditPositionId);	// @audit-issue

62:        CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

63:        DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

65:        uint256 assignedCollateral = state.getCreditPositionProRataAssignedCollateral(creditPosition);	// @audit-issue
```
[35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L35-L35), [36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L36-L36), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L39-L39), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L42-L42), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L43-L43), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L46-L46), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L47-L47), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L52-L52), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L60-L60), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L62-L62), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L63-L63), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L65-L65), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

44:            state.getCreditPosition(params.creditPositionWithDebtToRepayId);	// @audit-issue

46:            state.getDebtPositionByCreditPositionId(params.creditPositionWithDebtToRepayId);	// @audit-issue

48:        uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);	// @audit-issue

51:        if (state.getLoanStatus(params.creditPositionWithDebtToRepayId) != LoanStatus.ACTIVE) {	// @audit-issue

52:            revert Errors.LOAN_NOT_ACTIVE(params.creditPositionWithDebtToRepayId);	// @audit-issue

65:                state.getCreditPosition(params.creditPositionToCompensateId);	// @audit-issue

67:                state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId);	// @audit-issue

68:            if (!state.isCreditPositionTransferrable(params.creditPositionToCompensateId)) {	// @audit-issue

71:                    state.getLoanStatus(params.creditPositionToCompensateId),	// @audit-issue

72:                    state.collateralRatio(debtPositionToCompensate.borrower)	// @audit-issue

77:                    < state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId).dueDate	// @audit-issue

79:                revert Errors.DUE_DATE_NOT_COMPATIBLE(	// @audit-issue

84:                revert Errors.INVALID_LENDER(creditPositionToCompensate.lender);	// @audit-issue

87:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionToCompensateId);	// @audit-issue

89:            amountToCompensate = Math.min(amountToCompensate, creditPositionToCompensate.credit);	// @audit-issue

94:            revert Errors.COMPENSATOR_IS_NOT_BORROWER(msg.sender, debtPositionToRepay.borrower);	// @audit-issue

99:            revert Errors.NULL_AMOUNT();	// @audit-issue

112:            state.getCreditPosition(params.creditPositionWithDebtToRepayId);	// @audit-issue

114:            state.getDebtPositionByCreditPositionId(params.creditPositionWithDebtToRepayId);	// @audit-issue

116:        uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);	// @audit-issue

127:            creditPositionToCompensate = state.getCreditPosition(params.creditPositionToCompensateId);	// @audit-issue

128:            amountToCompensate = Math.min(amountToCompensate, creditPositionToCompensate.credit);	// @audit-issue

148:            uint256 fragmentationFeeInCollateral = Math.min(	// @audit-issue

149:                state.debtTokenAmountToCollateralTokenAmount(state.feeConfig.fragmentationFee),	// @audit-issue

150:                state.data.collateralToken.balanceOf(msg.sender)	// @audit-issue
```
[44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L44-L44), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L46-L46), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L48-L48), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L52-L52), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L65-L65), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L67-L67), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L68-L68), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L71-L71), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L72-L72), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L77-L77), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L79-L79), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L84-L84), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L87-L87), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L89-L89), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L94-L94), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L99-L99), [112](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L112-L112), [114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L114-L114), [116](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L116-L116), [127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L127-L127), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L128-L128), [148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L148-L148), [149](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L149-L149), [150](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L150-L150), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

59:        if (loanOffer.isNull()) {	// @audit-issue

60:            revert Errors.INVALID_LOAN_OFFER(params.lender);	// @audit-issue

72:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

73:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

75:                revert Errors.BORROWER_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue

77:            if (!state.isCreditPositionTransferrable(params.creditPositionId)) {	// @audit-issue

80:                    state.getLoanStatus(params.creditPositionId),	// @audit-issue

81:                    state.collateralRatio(debtPosition.borrower)	// @audit-issue

88:                revert Errors.NOT_ENOUGH_CREDIT(params.amount, creditPosition.credit);	// @audit-issue

94:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

99:            revert Errors.DUE_DATE_GREATER_THAN_MAX_DUE_DATE(block.timestamp + tenor, loanOffer.maxDueDate);	// @audit-issue

104:            revert Errors.PAST_DEADLINE(params.deadline);	// @audit-issue

108:        uint256 apr = loanOffer.getAPRByTenor(	// @audit-issue

117:            revert Errors.APR_GREATER_THAN_MAX_APR(apr, params.maxAPR);	// @audit-issue

141:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

142:            creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

147:        uint256 ratePerTenor = state.data.users[params.lender].loanOffer.getRatePerTenor(	// @audit-issue

175:                    : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),	// @audit-issue

177:                    ? Math.mulDivUp(cashAmountOut, PERCENT + ratePerTenor, PERCENT - state.getSwapFeePercent(tenor))	// @audit-issue
```
[59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L59-L59), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L60-L60), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L73-L73), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L75-L75), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L77-L77), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L80-L80), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L81-L81), [88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L88-L88), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L94-L94), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L99-L99), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L104-L104), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L108-L108), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L117-L117), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L141-L141), [142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L142-L142), [147](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L147-L147), [175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L175-L175), [177](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L177-L177), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

87:        if (Strings.equal(params.key, "crOpening")) {	// @audit-issue

89:        } else if (Strings.equal(params.key, "crLiquidation")) {	// @audit-issue

91:                revert Errors.INVALID_COLLATERAL_RATIO(params.value);	// @audit-issue

94:        } else if (Strings.equal(params.key, "minimumCreditBorrowAToken")) {	// @audit-issue

96:        } else if (Strings.equal(params.key, "borrowATokenCap")) {	// @audit-issue

98:        } else if (Strings.equal(params.key, "minTenor")) {	// @audit-issue

103:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue

108:        } else if (Strings.equal(params.key, "maxTenor")) {	// @audit-issue

113:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue

118:        } else if (Strings.equal(params.key, "swapFeeAPR")) {	// @audit-issue

120:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue

125:        } else if (Strings.equal(params.key, "fragmentationFee")) {	// @audit-issue

127:        } else if (Strings.equal(params.key, "liquidationRewardPercent")) {	// @audit-issue

129:        } else if (Strings.equal(params.key, "overdueCollateralProtocolPercent")) {	// @audit-issue

131:        } else if (Strings.equal(params.key, "collateralProtocolPercent")) {	// @audit-issue

133:        } else if (Strings.equal(params.key, "feeRecipient")) {	// @audit-issue

135:        } else if (Strings.equal(params.key, "priceFeed")) {	// @audit-issue

137:        } else if (Strings.equal(params.key, "variablePoolBorrowRateStaleRateInterval")) {	// @audit-issue

140:            revert Errors.INVALID_KEY(params.key);	// @audit-issue

143:        Initialize.validateInitializeFeeConfigParams(feeConfigParams(state));	// @audit-issue

144:        Initialize.validateInitializeRiskConfigParams(riskConfigParams(state));	// @audit-issue

145:        Initialize.validateInitializeOracleParams(oracleParams(state));	// @audit-issue

147:        emit Events.UpdateConfig(params.key, params.value);	// @audit-issue
```
[87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L87-L87), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L89-L89), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L91-L91), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L94-L94), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L96-L96), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L98-L98), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L103-L103), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L108-L108), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L113-L113), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L118-L118), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L120-L120), [125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L125-L125), [127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L127-L127), [129](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L129-L129), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L131-L131), [133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L133-L133), [135](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L135-L135), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L137-L137), [140](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L140-L140), [143](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L143-L143), [144](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L144-L144), [145](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L145-L145), [147](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L147-L147), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

38:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue

44:        if (!state.isDebtPositionLiquidatable(params.debtPositionId)) {	// @audit-issue

47:                state.collateralRatio(debtPosition.borrower),	// @audit-issue

48:                state.getLoanStatus(params.debtPositionId)	// @audit-issue

65:            revert Errors.LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(	// @audit-issue

79:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue

80:        LoanStatus loanStatus = state.getLoanStatus(params.debtPositionId);	// @audit-issue

81:        uint256 collateralRatio = state.collateralRatio(debtPosition.borrower);	// @audit-issue

86:        uint256 collateralProtocolPercent = state.isUserUnderwater(debtPosition.borrower)	// @audit-issue

90:        uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);	// @audit-issue

91:        uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);	// @audit-issue

96:            uint256 liquidatorReward = Math.min(	// @audit-issue

110:            collateralRemainder = Math.min(collateralRemainder, collateralRemainderCap);	// @audit-issue

124:        debtPosition.liquidityIndexAtRepayment = state.data.borrowAToken.liquidityIndex();	// @audit-issue

125:        state.repayDebt(params.debtPositionId, debtPosition.futureValue);	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L38-L38), [44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L44-L44), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L47-L47), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L48-L48), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L65-L65), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L79-L79), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L80-L80), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L81-L81), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L86-L86), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L90-L90), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L91-L91), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L96-L96), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L110-L110), [124](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L124-L124), [125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L125-L125), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

51:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue

55:        state.validateLiquidate(	// @audit-issue

63:        if (state.getLoanStatus(params.debtPositionId) != LoanStatus.ACTIVE) {	// @audit-issue

64:            revert Errors.LOAN_NOT_ACTIVE(params.debtPositionId);	// @audit-issue

72:        if (borrowOffer.isNull()) {	// @audit-issue

73:            revert Errors.INVALID_BORROW_OFFER(params.borrower);	// @audit-issue

78:            revert Errors.PAST_DEADLINE(params.deadline);	// @audit-issue

82:        uint256 apr = borrowOffer.getAPRByTenor(	// @audit-issue

91:            revert Errors.APR_LOWER_THAN_MIN_APR(apr, params.minAPR);	// @audit-issue

126:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue

131:        liquidatorProfitCollateralToken = state.executeLiquidate(	// @audit-issue

138:        uint256 ratePerTenor = borrowOffer.getRatePerTenor(	// @audit-issue

160:        state.data.debtToken.mint(params.borrower, debtPosition.futureValue);	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L51-L51), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L55-L55), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L63-L63), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L64-L64), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L73-L73), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L78-L78), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L82-L82), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L91-L91), [126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L126-L126), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L131-L131), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L138-L138), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L160-L160), 


```solidity
Path: ./src/libraries/actions/Claim.sol

32:        CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

37:        if (state.getLoanStatus(params.creditPositionId) != LoanStatus.REPAID) {	// @audit-issue

38:            revert Errors.LOAN_NOT_REPAID(params.creditPositionId);	// @audit-issue

41:            revert Errors.CREDIT_POSITION_ALREADY_CLAIMED(params.creditPositionId);	// @audit-issue

49:        CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

50:        DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

53:            creditPosition.credit, state.data.borrowAToken.liquidityIndex(), debtPosition.liquidityIndexAtRepayment	// @audit-issue

55:        state.reduceCredit(params.creditPositionId, creditPosition.credit);	// @audit-issue

58:        emit Events.Claim(params.creditPositionId, creditPosition.debtPositionId);	// @audit-issue
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L32-L32), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L38-L38), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L41-L41), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L49-L49), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L50-L50), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L53-L53), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L55-L55), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L58-L58), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

64:            revert Errors.NULL_ADDRESS();	// @audit-issue

82:            revert Errors.INVALID_COLLATERAL_PERCENTAGE_PREMIUM(f.overdueCollateralProtocolPercent);	// @audit-issue

87:            revert Errors.INVALID_COLLATERAL_PERCENTAGE_PREMIUM(f.collateralProtocolPercent);	// @audit-issue

92:            revert Errors.NULL_ADDRESS();	// @audit-issue

101:            revert Errors.INVALID_COLLATERAL_RATIO(r.crOpening);	// @audit-issue

106:            revert Errors.INVALID_COLLATERAL_RATIO(r.crLiquidation);	// @audit-issue

109:            revert Errors.INVALID_LIQUIDATION_COLLATERAL_RATIO(r.crOpening, r.crLiquidation);	// @audit-issue

114:            revert Errors.NULL_AMOUNT();	// @audit-issue

122:            revert Errors.NULL_AMOUNT();	// @audit-issue

126:            revert Errors.INVALID_MAXIMUM_TENOR(r.maxTenor);	// @audit-issue

135:            revert Errors.NULL_ADDRESS();	// @audit-issue

138:        IPriceFeed(o.priceFeed).getPrice();	// @audit-issue

149:            revert Errors.NULL_ADDRESS();	// @audit-issue

151:        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {	// @audit-issue

152:            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingCollateralToken).decimals());	// @audit-issue

157:            revert Errors.NULL_ADDRESS();	// @audit-issue

159:        if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {	// @audit-issue

160:            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingBorrowToken).decimals());	// @audit-issue

165:            revert Errors.NULL_ADDRESS();	// @audit-issue

241:            string.concat("Size ", IERC20Metadata(state.data.underlyingCollateralToken).name()),	// @audit-issue

242:            string.concat("sz", IERC20Metadata(state.data.underlyingCollateralToken).symbol()),	// @audit-issue

243:            IERC20Metadata(state.data.underlyingCollateralToken).decimals()	// @audit-issue

249:            string.concat("Size Scaled ", IERC20Metadata(state.data.underlyingBorrowToken).name()),	// @audit-issue

250:            string.concat("sza", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),	// @audit-issue

251:            IERC20Metadata(state.data.underlyingBorrowToken).decimals()	// @audit-issue

255:            string.concat("Size Debt ", IERC20Metadata(state.data.underlyingBorrowToken).name()),	// @audit-issue

256:            string.concat("szDebt", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),	// @audit-issue

257:            IERC20Metadata(state.data.underlyingBorrowToken).decimals()	// @audit-issue
```
[64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L64-L64), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L82-L82), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L87-L87), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L92-L92), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L101-L101), [106](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L106-L106), [109](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L109-L109), [114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L114-L114), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L122-L122), [126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L126-L126), [135](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L135-L135), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L138-L138), [149](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L149-L149), [151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L151-L151), [152](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L152-L152), [157](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L157-L157), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L159-L159), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L160-L160), [165](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L165-L165), [241](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L241-L241), [242](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L242-L242), [243](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L243-L243), [249](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L249-L249), [250](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L250-L250), [251](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L251-L251), [255](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L255-L255), [256](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L256-L256), [257](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L257-L257), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

38:            revert Errors.INVALID_TOKEN(params.token);	// @audit-issue

43:            revert Errors.NULL_AMOUNT();	// @audit-issue

48:            revert Errors.NULL_ADDRESS();	// @audit-issue

55:            amount = Math.min(params.amount, state.data.borrowAToken.balanceOf(msg.sender));	// @audit-issue

60:            amount = Math.min(params.amount, state.data.collateralToken.balanceOf(msg.sender));	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L38-L38), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L43-L43), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L48-L48), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L55-L55), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L60-L60), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

29:        if (!borrowOffer.isNull()) {	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L29-L29), 


```solidity
Path: ./src/libraries/actions/Repay.sol

35:        if (state.getLoanStatus(params.debtPositionId) == LoanStatus.REPAID) {	// @audit-issue

36:            revert Errors.LOAN_ALREADY_REPAID(params.debtPositionId);	// @audit-issue

47:        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);	// @audit-issue

50:        debtPosition.liquidityIndexAtRepayment = state.data.borrowAToken.liquidityIndex();	// @audit-issue

51:        state.repayDebt(params.debtPositionId, debtPosition.futureValue);	// @audit-issue

53:        emit Events.Repay(params.debtPositionId);	// @audit-issue
```
[35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L35-L35), [36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L36-L36), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L47-L47), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L50-L50), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L51-L51), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L53-L53), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

34:        if (!loanOffer.isNull()) {	// @audit-issue

40:                revert Errors.NULL_MAX_DUE_DATE();	// @audit-issue

43:                revert Errors.PAST_MAX_DUE_DATE(params.maxDueDate);	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L34-L34), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L40-L40), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L43-L43), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

65:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

66:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

67:            if (!state.isCreditPositionTransferrable(params.creditPositionId)) {	// @audit-issue

70:                    state.getLoanStatus(params.creditPositionId),	// @audit-issue

71:                    state.collateralRatio(debtPosition.borrower)	// @audit-issue

76:                revert Errors.CREDIT_NOT_FOR_SALE(params.creditPositionId);	// @audit-issue

86:        if (borrowOffer.isNull()) {	// @audit-issue

87:            revert Errors.INVALID_BORROW_OFFER(borrower);	// @audit-issue

92:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

97:            revert Errors.PAST_DEADLINE(params.deadline);	// @audit-issue

101:        uint256 apr = borrowOffer.getAPRByTenor(	// @audit-issue

110:            revert Errors.APR_LOWER_THAN_MIN_APR(apr, params.minAPR);	// @audit-issue

137:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);	// @audit-issue

138:            creditPosition = state.getCreditPosition(params.creditPositionId);	// @audit-issue

144:        uint256 ratePerTenor = state.data.users[borrower].borrowOffer.getRatePerTenor(	// @audit-issue
```
[65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L65-L65), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L66-L66), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L67-L67), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L70-L70), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L71-L71), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L76-L76), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L86-L86), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L87-L87), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L92-L92), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L97-L97), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L101-L101), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L110-L110), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L137-L137), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L138-L138), [144](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L144-L144), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

42:            revert Errors.INVALID_MSG_VALUE(msg.value);	// @audit-issue

50:            revert Errors.INVALID_TOKEN(params.token);	// @audit-issue

55:            revert Errors.NULL_AMOUNT();	// @audit-issue

60:            revert Errors.NULL_ADDRESS();	// @audit-issue

72:            state.data.weth.forceApprove(address(this), amount);	// @audit-issue

81:                state.validateBorrowATokenCap();	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L42-L42), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L50-L50), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L55-L55), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L60-L60), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L72-L72), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L81-L81), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

23:            revert Errors.NULL_AMOUNT();	// @audit-issue

51:        revert Errors.NOT_SUPPORTED();	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L23-L23), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L51-L51), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

34:            revert Errors.NULL_ADDRESS();	// @audit-issue

43:        revert Errors.NOT_SUPPORTED();	// @audit-issue

57:        revert Errors.NOT_SUPPORTED();	// @audit-issue

91:        return super.balanceOf(account);	// @audit-issue

112:        return super.totalSupply();	// @audit-issue

124:        return variablePool.getReserveNormalizedIncome(address(underlyingToken));	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L34-L34), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L43-L43), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L57-L57), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L91-L91), [112](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L112-L112), [124](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L124-L124), 


#### Recommendation

Review your smart contracts to identify and remove `block.number` and `block.timestamp` from the parameters of emitted events. Instead of manually adding these fields, rely on the Ethereum blockchain's inherent inclusion of this information within the block context. Simplify your event definitions to include only the essential data specific to the event's purpose, excluding universally available block metadata.

### Use assembly to check for `address(0)`
In Solidity, it's a common practice to check whether an Ethereum address variable is set to the zero address (`address(0)`) to handle various scenarios, such as token transfers or contract interactions. Typically, this check is performed using a conditional statement like `if (addressVariable == address(0))`.

However, using this approach in high-frequency or gas-sensitive operations can lead to unnecessary gas costs. A more gas-efficient alternative is to use inline assembly to perform the zero address check, which can significantly reduce gas consumption, especially in loops or complex contract logic.

By utilizing inline assembly for this specific check, you can optimize gas usage and make your Solidity code more efficient.

```solidity
Path: ./src/oracle/PriceFeed.sol

43:        if (_base == address(0) || _quote == address(0)) {	// @audit-issue

64:        if (address(sequencerUptimeFeed) != address(0)) {	// @audit-issue
```
[43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L43-L43), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L64-L64), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

63:        if (owner == address(0)) {	// @audit-issue

91:        if (f.feeRecipient == address(0)) {	// @audit-issue

134:        if (o.priceFeed == address(0)) {	// @audit-issue

148:        if (d.underlyingCollateralToken == address(0)) {	// @audit-issue

156:        if (d.underlyingBorrowToken == address(0)) {	// @audit-issue

164:        if (d.variablePool == address(0)) {	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L63-L63), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L91-L91), [134](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L134-L134), [148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L148-L148), [156](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L156-L156), [164](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L164-L164), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

47:        if (params.to == address(0)) {	// @audit-issue
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L47-L47), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

59:        if (params.to == address(0)) {	// @audit-issue
```
[59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L59-L59), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

33:        if (address(variablePool_) == address(0) || address(underlyingToken_) == address(0)) {	// @audit-issue

52:        emit TransferUnscaled(address(0), to, _unscale(scaledAmount));	// @audit-issue

66:        emit TransferUnscaled(from, address(0), _unscale(scaledAmount));	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L33-L33), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L52-L52), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L66-L66), 


#### Recommendation

To optimize gas usage in your Solidity code, consider using inline assembly for checking `address(0)`. This approach can significantly reduce gas costs, especially in high-frequency or gas-sensitive operations, leading to more efficient contract execution.

### Use assembly to check for `0`
Using assembly to check for zero can save gas by allowing more direct access to the evm and reducing some of the overhead associated with high-level operations in solidity.

```solidity
Path: ./src/SizeView.sol

180:        if (tenor == 0) {	// @audit-issue
```
[180](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L180-L180), 


```solidity
Path: ./src/oracle/PriceFeed.sol

48:        if (_baseStalePriceInterval == 0 || _quoteStalePriceInterval == 0) {	// @audit-issue

88:        if (price <= 0) revert Errors.INVALID_PRICE(address(aggregator), price);	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L48-L48), [88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L88-L88), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

22:        if (0 < credit && credit < state.riskConfig.minimumCreditBorrowAToken) {	// @audit-issue

59:        if (debt != 0) {	// @audit-issue
```
[22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L22-L22), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L59-L59), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

33:        return self.maxDueDate == 0 && self.curveRelativeTime.isNull();	// @audit-issue

53:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue

81:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L33-L33), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L53-L53), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L81-L81), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

133:        if (debtPosition.futureValue == 0) {	// @audit-issue

156:        if (debt != 0) {	// @audit-issue

181:        if (debtPositionFutureValue != 0) {	// @audit-issue
```
[133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L133-L133), [156](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L156-L156), [181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L181-L181), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

39:        return self.tenors.length == 0 && self.aprs.length == 0 && self.marketRateMultipliers.length == 0;	// @audit-issue

51:        if (self.tenors.length == 0 || self.aprs.length == 0 || self.marketRateMultipliers.length == 0) {	// @audit-issue

63:        for (uint256 i = self.tenors.length; i != 0; i--) {	// @audit-issue

92:        if (marketRateMultiplier == 0) {	// @audit-issue

95:            params.variablePoolBorrowRateStaleRateInterval == 0	// @audit-issue
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L39-L39), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L51-L51), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L63-L63), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L92-L92), [95](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L95-L95), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

98:        if (amountToCompensate == 0) {	// @audit-issue

146:        if (exiterCreditRemaining > 0) {	// @audit-issue
```
[98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L98-L98), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L146-L146), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

100:                state.feeConfig.swapFeeAPR != 0	// @audit-issue

110:                state.feeConfig.swapFeeAPR != 0	// @audit-issue
```
[100](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L100-L100), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L110-L110), 


```solidity
Path: ./src/libraries/actions/Claim.sol

40:        if (creditPosition.credit == 0) {	// @audit-issue
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

113:        if (r.minimumCreditBorrowAToken == 0) {	// @audit-issue

121:        if (r.minTenor == 0) {	// @audit-issue
```
[113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L113-L113), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L121-L121), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

42:        if (params.amount == 0) {	// @audit-issue

56:            if (amount > 0) {	// @audit-issue

61:            if (amount > 0) {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L56-L56), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L61-L61), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

39:            if (params.maxDueDate == 0) {	// @audit-issue
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L39-L39), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

41:        if (msg.value != 0 && (msg.value != params.amount || params.token != address(state.data.weth))) {	// @audit-issue

54:        if (params.amount == 0) {	// @audit-issue

67:        if (msg.value > 0) {	// @audit-issue
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L41-L41), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L54-L54), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L67-L67), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

22:        if (decimals_ == 0) {	// @audit-issue
```
[22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L22-L22), 


#### Recommendation

To optimize gas usage in your Solidity code, consider using inline assembly for checking `0`. This approach can significantly reduce gas costs, especially in high-frequency or gas-sensitive operations, leading to more efficient contract execution.

### Use assembly to write `address` storage values
Using assembly `{ sstore(state.slot, addr)}` instead of `state = addr` can save gas.


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

37:        variablePool = variablePool_;	// @audit-issue

38:        underlyingToken = underlyingToken_;	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L38-L38), 


#### Recommendation

To reduce gas costs in your Solidity code, consider using assembly with `{ sstore(state.slot, addr) }` for writing `address` storage values instead of `state = addr`. This approach can result in significant gas savings.

### Use assembly to emit an `event`
To efficiently emit events, it's possible to utilize assembly by making use of scratch space and the free memory pointer. This approach has the advantage of potentially avoiding the costs associated with memory expansion.

However, it's important to note that in order to safely optimize this process, it is preferable to cache and restore the free memory pointer.

A good example of such practice can be seen in [Solady's](https://github.com/Vectorized/solady/blob/main/src/tokens/ERC1155.sol#L167) codebase.


```solidity
Path: ./src/Size.sol

128:        emit Events.VariablePoolBorrowRateUpdated(oldBorrowRate, borrowRate);	// @audit-issue
```
[128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L128-L128), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

48:        emit Events.UpdateDebtPosition(	// @audit-issue

75:        emit Events.CreateDebtPosition(debtPositionId, borrower, lender, futureValue, dueDate);	// @audit-issue

89:        emit Events.CreateCreditPosition(creditPositionId, lender, debtPositionId, RESERVED_ID, creditPosition.credit);	// @audit-issue

110:            emit Events.UpdateCreditPosition(	// @audit-issue

125:            emit Events.CreateCreditPosition(creditPositionId, lender, debtPositionId, exitCreditPositionId, credit);	// @audit-issue

142:        emit Events.UpdateCreditPosition(	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L48-L48), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L75-L75), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L89-L89), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L110-L110), [125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L125-L125), [142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L142-L142), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

72:            emit Events.UpdateCreditPosition(	// @audit-issue

77:        emit Events.SetUserConfiguration(	// @audit-issue
```
[72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L72-L72), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L77-L77), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

60:        emit Events.SelfLiquidate(params.creditPositionId);	// @audit-issue
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L60-L60), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

107:        emit Events.Compensate(	// @audit-issue
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L107-L107), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

131:        emit Events.SellCreditMarket(	// @audit-issue
```
[131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L131-L131), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

147:        emit Events.UpdateConfig(params.key, params.value);	// @audit-issue
```
[147](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L147-L147), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

83:        emit Events.Liquidate(params.debtPositionId, params.minimumCollateralProfit, collateralRatio, loanStatus);	// @audit-issue
```
[83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L83-L83), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

124:        emit Events.LiquidateWithReplacement(params.debtPositionId, params.borrower, params.minimumCollateralProfit);	// @audit-issue

153:        emit Events.UpdateDebtPosition(	// @audit-issue
```
[124](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L124-L124), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L153-L153), 


```solidity
Path: ./src/libraries/actions/Claim.sol

58:        emit Events.Claim(params.creditPositionId, creditPosition.debtPositionId);	// @audit-issue
```
[58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L58-L58), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

278:        emit Events.Initialize(f, r, o, d);	// @audit-issue
```
[278](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L278-L278), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

66:        emit Events.Withdraw(params.token, params.to, amount);	// @audit-issue
```
[66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L66-L66), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

46:        emit Events.SellCreditLimit(	// @audit-issue
```
[46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L46-L46), 


```solidity
Path: ./src/libraries/actions/Repay.sol

53:        emit Events.Repay(params.debtPositionId);	// @audit-issue
```
[53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L53-L53), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

60:        emit Events.BuyCreditLimit(	// @audit-issue
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L60-L60), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

125:        emit Events.BuyCreditMarket(	// @audit-issue
```
[125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L125-L125), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

87:        emit Events.Deposit(params.token, params.to, amount);	// @audit-issue
```
[87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L87-L87), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

52:        emit TransferUnscaled(address(0), to, _unscale(scaledAmount));	// @audit-issue

66:        emit TransferUnscaled(from, address(0), _unscale(scaledAmount));	// @audit-issue

82:        emit TransferUnscaled(from, to, value);	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L52-L52), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L66-L66), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L82-L82), 


#### Recommendation

To optimize event emission in your Solidity code, consider using assembly with scratch space and the free memory pointer. This approach can help reduce gas costs by avoiding memory expansion expenses. However, it's crucial to ensure safe optimization by caching and restoring the free memory pointer, as demonstrated in examples like Solady's codebase.

### Use assembly to validate `msg.sender`
We can use assembly to efficiently validate `msg.sender` with the least amount of opcodes necessary. For more details check the following report [Here](https://code4rena.com/reports/2023-05-juicebox#g-06-use-assembly-to-validate-msgsender)


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

50:            if (creditPosition.lender != msg.sender) {	// @audit-issue
```
[50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L50-L50), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

51:        if (msg.sender != creditPosition.lender) {	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L51-L51), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

93:        if (msg.sender != debtPositionToRepay.borrower) {	// @audit-issue
```
[93](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L93-L93), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

74:            if (msg.sender != creditPosition.lender) {	// @audit-issue
```
[74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L74-L74), 


#### Recommendation

To optimize the validation of `msg.sender` in your Solidity code, consider using assembly to achieve this with the minimum number of opcodes required. You can refer to the detailed report [Here](https://code4rena.com/reports/2023-05-juicebox#g-06-use-assembly-to-validate-msgsender) for more insights and examples on efficient implementation.
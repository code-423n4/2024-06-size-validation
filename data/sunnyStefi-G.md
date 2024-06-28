## [G-01] Unused custom errors

The errors `Errors::NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required)` and `Errors::NULL_STALE_RATE()` are currently defined in [L49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Errors.sol#L49) and [L74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Errors.sol#L74) but they are never used in the codebase. To ensure efficient use of resources, it is recommended to remove these errors. This will simplify the code while reducing complexity and improving maintainability and performance.

```diff
library Errors {
-   error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);
-   error NULL_STALE_RATE();
```

## [G-02] `constant` variables should be preferred in place of literals

The constant literal value `18` is used twice in the `Initialize` contract in [L151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Initialize.sol#L151) and [L159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Initialize.sol#L159). To improve code maintainability and facilitate future updates, consider defining this value as a _constant_. This can be defined as `UNDERLYING_TOKEN_SUPPORTED_DECIMALS`, which represents the number of allowed decimals for the underlying collateral token (WETH). This value is also utilized in the `Math` library [L32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Math.sol#L32) and `PriceFeed` [L28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/oracle/PriceFeed.sol#L28). By defining `18` as a constant, consistency is ensured across the codebase making it easier to support additional collateral tokens in the future.

```diff
library Initialize {
+	public constant UNDERLYING_TOKEN_SUPPORTED_DECIMALS = 18; // or define this in a constant file
	...

 function validateInitializeDataParams(InitializeDataParams memory d) internal view {
        ...
-        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {
+        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > UNDERLYING_TOKEN_SUPPORTED_DECIMALS) {
            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingCollateralToken).decimals());
        }
```

## [G-03] Extensive use of `storage state` can lead to excessive gas usage

The `AccountingLibrary::getCreditAmountIn` function frequently accesses the storage value `state.feeConfig.fragmentationFee` in [L240](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/AccountingLibrary.sol#L240), [L241](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/AccountingLibrary.sol#L241), [L254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/AccountingLibrary.sol#L254), [L256](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/AccountingLibrary.sol#L256). Each access to storage significantly increases gas consumption. To optimize this, consider storing the value in a _memory variable_ for repeated use within the function. This change will reduce gas costs and improve overall efficiency.

```diff
function getCreditAmountIn(
        State storage state,
        uint256 cashAmountOut,
        uint256 maxCashAmountOut,
        uint256 maxCredit,
        uint256 ratePerTenor,
        uint256 tenor
    ) internal view returns (uint256 creditAmountIn, uint256 fees) {
        //...
+       uint256 fragmentationFee = state.feeConfig.fragmentationFee;

-       if (maxCashAmountOut >= state.feeConfig.fragmentationFee) {
+       if (maxCashAmountOut >= fragmentationFee) {
            maxCashAmountOutFragmentation = maxCashAmountOut - fragmentationFee;
        }
```

Similar scenarios can be observed whenever the `storage` keyword is used as a function parameter. By identifying these patterns and storing frequently accessed storage variables in memory, gas usage can be minimized, resulting in more efficient and cost-effective code execution.

## [G-04] Counting down in `for` statements increases gas efficiency

In `YieldCurveLibrary::validateYieldCurve` [L63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/YieldCurveLibrary.sol#L63), the `for` statement counts down to save gas. This optimization occurs because the condition `i != 0` is cheaper to evaluate than `i < data.length`. To ensure consistency throughout the codebase and improve gas efficiency, similar optimizations should be applied to `Multicall::multicall` [L33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Multicall.sol#L33), `SetUserConfiguration::validateSetUserConfiguration` [L48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SetUserConfiguration.sol#L48), and `SetUserConfiguration::executeSetUserConfiguration` [L69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SetUserConfiguration.sol#L69).

```diff
function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {
        //...
-       for (uint256 i = 0; i < data.length; i++) {
+ 		for (uint256 i = data.length; i != 0; i--) {
```

## [G-05] Libraries that use `public` and `external` functions can be restricted to `internal`

`RiskLibrary` and similar libraries often contain `public` and `external` functions. However, since these functions are only accessed by the contracts that use the library, their visibility can be optimized by changing their visibility to `internal`.

```diff
- function validateMinimumCredit(State storage state, uint256 credit) public view {
+ function validateMinimumCredit(State storage state, uint256 credit) internal view {
```

The same optimisation can be applied within: `AccountingLibrary`, `BuyCreditLimit`, `BuyCreditMarket`, `CapsLibrary`, `Claim`, `Compensate`, `Deposit`, `DepositTokenLibrary`, `Initialize`, `Liquidate`, `LiquidateWithReplacement`, `LoanLibrary`, `Multicall`, `OfferLibrary`, `Repay`, `RiskLibrary`, `SelfLiquidate`, `SellCreditLimit`, `SellCreditMarket`, `SetUserConfiguration`, `UpdateConfig`, `Withdraw`

## [G-06] Constant `PriceFeed::decimals` can be declared `private`

The `PriceFeed::decimal` [L 28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/oracle/PriceFeed.sol#L28) is a public constant value, which means the EVM automatically generates getter functions and adds more entries to the method ID table. To improve efficiency, it is better to read these values directly from the source code and specify them as `private`. Additionally, it is conventional to write constants in uppercase text for better readability.

```diff
+uint256 private constant DECIMALS = 18;
-uint256 public constant decimals = 18;
```

**Similar Issues**

`Size::KEEPER_ROLE`, `Size::PAUSER_ROLE`, `Size::BORROW_RATE_UPDATER_ROLE`,

## [G-07] Using shift operator instead of division in `Math::binarySearch`

`Math::binarySearch` [L58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Math.sol#L58) currently uses division `DIV` by 2, which costs 5 gas. The same result can be achieved using the shift operator `SHR`, which costs only 3 gas. Since this operation is performed every time an APR is calculated, switching to the shift operator can provide a gas cost advantage.

```diff
function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {
        ...
        while (low <= high) {
-           uint256 mid = (low + high) / 2;
+			uint256 mid = (low + high) >> 1;

```

## [G-08] `NonTransferrableToken` and `NonTransferrableScaledToken` `OnlyOwner` functions can be `payable`

The protocol documentation states that the owner of the `NonTransferrableToken` and `NonTransferrableScaledToken` contracts is expected to be a trusted entity. Therefore, is safe to assume that the owner is also trusted and he will not send unexpected Ether when calling `onlyOwner` functions. By marking these functions as `payable`, the gas cost for legitimate callers will be lower, as the compiler will omit checks related to Ether transfers.

`NonTransferrableToken.sol` [L29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/token/NonTransferrableToken.sol#L29):

```diff
-    function mint(address to, uint256 value) external virtual onlyOwner {
+    function mint(address to, uint256 value) external payable virtual onlyOwner {
        _mint(to, value);
    }
```

`NonTransferrableScaledToken.sol` [L42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/token/NonTransferrableScaledToken.sol#L42):

```diff
-    function mint(address, uint256) external view override onlyOwner {
+	 function mint(address, uint256) external payable override onlyOwner {
        revert Errors.NOT_SUPPORTED();
    }
```

## [G-09] Checking the length of an array in every for loop condition can be optimized

In `Multicall::multicall` [L33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Multicall.sol#L33), the `data.length` is accessed and evaluated during each iteration of the loop. Each access incurs a gas cost due to reading the array length from storage or memory. By storing the array length in a memory variable before the loop begins, these costs are reduced, as accessing a local memory variable is cheaper than repeatedly accessing the array length.

```diff
library Multicall {
   // ...
function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {
		//...
+		uint256 dataLength = data.length;
		//...
-       for (uint256 i = 0; i < data.length; i++) {
+       for (uint256 i = 0; i < dataLength; i++) {
            results[i] = Address.functionDelegateCall(address(this), data[i]);
        }

```
**Proof of concept**
```javascript
function test_dataLength() public {
        uint256[] memory data = new uint256[](100);
        uint256 gasStart1 = gasleft();
        for (uint256 i = 0; i < data.length; i++) {
            uint256 a = 1;
        }
        uint256 gas1 = gasStart1 - gasleft(); // cost: 4645 gas

        uint256 gasStart2 = gasleft();
        uint256 length = data.length;
        for (uint256 i = 0; i < length; i++) {
            uint256 a = 1;
        }
        uint256 gas2 = gasStart2 - gasleft(); // cost: 4351 gas

        assertGt(gas1, gas2); // almost 300 gas are saved on a multicall of length 100
    }
```
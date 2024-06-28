## [L-01] Unchecked calldata length and gas usage can cause Denial of Service (DoS) attacks

`Multicall::multicall` [L33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Multicall.sol#L33) allows multiple function calls to be aggregated into a single transaction. However, there is no limit on the length of the calldata array, nor is there a gas monitoring mechanism implemented.

```javascript
    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {
       //...
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            results[i] = Address.functionDelegateCall(address(this), data[i]);
        }
```

### Impact

The current situation can lead to Denial of Service (DoS) attacks where a malicious actor exhausts the [maximum block gas limit](https://etherscan.io/chart/gaslimit), by either batching a large number of calls or by sending few calls with high gas consumption. This would prevent other legitimate transactions from being included in the same block, causing delays and increasing transaction fees due to mempool congestion. Additionally, if another protocol relies on the `Size::multicall` function, this vulnerability could be exploited to halt the functionality of that protocol.

### Recommended Mitigation

Consider implementing checks on `data.length` before initializing the `results` array to prevent excessive calls. Additionally, incorporate a gas monitoring mechanism to track gas usage for each request, and revert the transaction if a specified threshold is exceeded.

### Similar issues

The functions `SetUserConfiguration::validateSetUserConfiguration` [L48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SetUserConfiguration.sol#L48) and `SetUserConfiguration::executeSetUserConfiguration` [L69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SetUserConfiguration.sol#L69) loop through the user's `creditPositionIds` without previous length validation, potentially leading to a DoS attack. If such an attack occurs in `SetUserConfiguration::executeSetUserConfiguration`, the `Event.UpdateCreditPosition` event will be emitted even though the transaction ultimately reverts, misleading any event observers.

## [L-02] Binary search `low + high` can cause overflow

The `Math::binarySearch` function is used to calculate the APR using `tenors` values as parameters in [L124](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/YieldCurveLibrary.sol#L124). Although these values are unlikely to cause direct overflow, it's still advisable to use safe arithmetic practices in all calculations to prevent potential issues and ensure future code robustness.

The expression `(low +high)/2` can be changed for its equivalent `low + (high - low) / 2`

```diff
while (low <= high) {
-   uint256 mid = (low + high) / 2;
+   uint256 mid = low + (high - low) / 2;
   if (array[mid] == value) {
      return (mid, mid);
      ...
```

## [L-03] Unnecessary calls are performed when `executeSelfLiquidate` has assigned collateral equals to `zero`

The `SelfLiquidate::executeSelfLiquidate` function [L65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SelfLiquidate.sol#L65) uses `getCreditPositionProRataAssignedCollateral` to determine the value of the collateral assigned to a credit position. If this value is zero, calling the subsequent functions `reduceDebtAndCredit` and `transferFrom` will not result in any state change.

### Impact

This scenario can result in gas inefficiencies and misleading logs due to the execution of unnecessary operations. For instance, in the `AccountingLibrary::repayDebt` function [L48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/AccountingLibrary.sol#L48), the `Events.UpdateDebtPosition` event will be emitted even if the debt position value remains unchanged. This can confuse event observers, as the emitted event will not accurately reflect any effective update to the `debtPosition.futureValue`.

### Recommended Mitigation

Implement checks to ensure that `reduceDebtAndCredit` and `transferFrom` are only called when the assigned collateral value is greater than zero:

```diff
    function executeSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external {
       ...
+        if (assignedCollateral > 0) {
            // debt and credit reduction
            state.reduceDebtAndCredit(creditPosition.debtPositionId, params.creditPositionId, creditPosition.credit);
            state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);
+        }

```

## [L-04] Multiple `PERCENTAGE` import statements from the same source

The contracts `Initialize` [L11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Initialize.sol#L11), `Liquidate` [L6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L6), and `LiquidateWithReplacement` [L6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/LiquidateWithReplacement.sol#L6) import the constant `PERCENT` twice, once through the Math library and once directly:

```javascript
import { Math } from "@src/libraries/Math.sol";

import { PERCENT } from "@src/libraries/Math.sol";
```

In addition to the value being imported more times than necessary, this can reduce readability and make the code less concise and clear. Separate import statements might suggest that the `PERCENT` constant is part of a different contract. Consolidating imports into a single line clarifies that both elements come from the `Math` library. Additionally, if the import path changes, only one line needs to be updated instead of multiple, simplifying maintenance.

### Recommended Mitigation

The two separate import statements can be consolidated into a single one:

```diff
-import {Math} from "@src/libraries/Math.sol";
-import {PERCENT} from "@src/libraries/Math.sol";
+import {Math, PERCENT} from "@src/libraries/Math.sol";
```

Alternatively, the library name can be used directly when calling its constant. For example, the `Liquidate` contract can be simplified as follows:

```diff
import {Math} from "@src/libraries/Math.sol";
-import {PERCENT} from "@src/libraries/Math.sol";
...
function executeLiquidate(State storage state, LiquidateParams calldata params)
        external
        returns (uint256 liquidatorProfitCollateralToken){
         //...
-         uint256 collateralRemainderCap = Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, PERCENT);
+         uint256 collateralRemainderCap = Math.mulDivDown(debtInCollateralToken, state.riskConfig.crLiquidation, Math.PERCENT);

```

## [L-05] Inconsistent function naming

The function `LoanLibrary::getDebtPosition` [L80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/LoanLibrary.sol#L80) is not as self-explanatory as `LoanLibrary::getDebtPositionByCreditPositionId` [L109](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/LoanLibrary.sol#L109).

The former implicitly assumes that a `debtPositionId` will be passed as a parameter, while the latter explicitly refers to the `creditPositionId` in its name. Similarly, the boolean function `isCreditPositionId` clearly indicates that it requires a `positionId` as a parameter.

This inconsistency in naming conventions reduces code readability. To enhance consistency and clarity, consider adopting a uniform naming convention, such as `isDebtPositionById` instead of `isDebtPositionId` [L63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/LoanLibrary.sol#L63C14-L63C30) and `getDebtPositionById` instead of `LoanLibrary::getDebtPosition` [L80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/LoanLibrary.sol#L80).

## [L-06] `Claim::transferFrom` is not used to transfer tokens on behalf of an address

`Claim::executeClaim` [L56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Claim.sol#L56) uses the `transferFrom(address(this),to, amount)` function to transfer address from the contract itself to the lender of the credit position.

The `transferFrom` method is typically used to transfer tokens on behalf of an address. In the current case, the intent is to transfer tokens held by the contract itself, making the `transfer` function more suitable for this situation. Additionally, the `transferFrom` function requires prior approval because it relies on the allowance mechanism. This requirement is implicitly invoked in `DepositTokenLibrary::depositUnderlyingBorrowTokenToVariablePool`[L59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/DepositTokenLibrary.sol#L59), meaning that necessary approvals must be in place beforehand for these functions to execute successfully, reducing readability and diminishing code clarity.

### Recommended Mitigation

Following the guidelines in the [OpenZeppelin library documentation](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-transfer-address-uint256-), using the `transfer` function instead of `transferFrom` can be preferable:

```diff
function executeClaim(State storage state, ClaimParams calldata params) external {
        //...
-       state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);
+       state.data.borrowAToken.transfer(creditPosition.lender, claimAmount);
```

## [L-07] The comparison `params.creditPositionId == RESERVED_ID` repetition hinders both `BuyCreditMarket` and `SellCreditMarket` functions' readability

The functions in the `BuyCreditMarket` and `SellCreditMarket` contracts could be streamline and to improve readability, it's preferable to abstract the statement `params.creditPositionId == RESERVED_ID`, which is repeated six times in each contract.

This can be done in two ways, by either using a memory variable or implementing a helper function. The first approach is recommended since `params.creditPositionId` will never change during the function execution. This will be also cheaper in terms of gas consumption since the comparison (3 gas) is performed only once in the assignment (tot 9 gas) but its consequent access (5 times in a single function) is negligible.

The following change can be applied to `BuyCreditMarket::executeBuyCreditMarket`, starting from [L133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/BuyCreditMarket.sol#L133):

```diff
 function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
        external
        returns (uint256 cashAmountIn)
    {
        //...
+       bool isCreditPositionIdReservedId = params.creditPositionId == RESERVED_ID;

        //...
-       if (params.creditPositionId == RESERVED_ID) {
+       if (isCreditPositionIdReservedId) {
         ..
```

## [L-08] Scientific notation can improve code clarity and consistency in `PriceFeed::getPrice`

The `PriceFeed::getPrice` function [L80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/oracle/PriceFeed.sol#L80C60-L80C68) currently uses exponential notation `10 ** decimals`. While this method can be more gas-efficient, it is recommended to use scientific notation (`1e18`) for better code clarity and alignment with best practices in Solidity development. This change will ensure consistency with other parts of the protocol, such as `Math::PERCENT`. Additionally, using constants instead of literal numbers enhances readability and maintainability.

```diff
contract PriceFeed is IPriceFeed {
+   uint256 private constant DECIMALS_FACTOR = 1e18;`
    //...
function getPrice() external view returns (uint256) {
       //...
        return Math.mulDivDown(
-            _getPrice(base, baseStalePriceInterval), 10 ** decimals, _getPrice(quote, quoteStalePriceInterval)
+            _getPrice(base, baseStalePriceInterval), DECIMALS_FACTOR, _getPrice(quote, quoteStalePriceInterval)
        );
    }
```

## [L-09] Single-step ownership `transfer` leaves room for unrecoverable errors during ownership transfers

OpenZeppelin's `Ownable` contract uses a single-step ownership transfer method, which is implemented in both `NonTransferrableToken` [L19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/token/NonTransferrableToken.sol#L19) and `NonTransferrableScaledToken`. This approach carries the risk that if an incorrect address is provided during the transfer, the ownership role could be permanently lost. This could impact all functions restricted by `onlyOwner` throughout the protocol, including core functionalities.

Even though the owner is a trusted entity, it is best practice to reduce the risk of errors during ownership transfer. It's recommended to use OpenZeppelin's [`Ownable2Step`](https://docs.openzeppelin.com/contracts/5.x/api/access#Ownable2Step) contract, which implements a two-step ownership transfer pattern. In this case, the new owner must explicitly claim his new right using `acceptOwnership`. If the new owner fails to do so, the current owner can retain control of the contract.

## [L-10] The `SellCreditMarket` event is being emitted with incorrect arguments

The event `Events.SellCreditMarket` is emitted in `SellCreditMarket::executeSellCreditMarket` [L131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SellCreditMarket.sol#L131) with `params.tenor` instead of `params.deadline`, as specified in its declaration in the `Events` library:

```
 event SellCreditMarket(
        address indexed lender,
        uint256 indexed creditPositionId,
        uint256 indexed tenor,
        uint256 amount,
        uint256 dueDate, // parameter used incorrectly
        bool exactAmountIn
    );
```

This can be solved by making the following change in `SellCreditMarket::executeSellCreditMarket`:

```diff
- emit Events.SellCreditMarket( params.lender, params.creditPositionId, params.tenor, params.amount, params.tenor, params.exactAmountIn);
+ emit Events.SellCreditMarket( params.lender, params.creditPositionId, params.tenor, params.amount, params.deadline, params.exactAmountIn);
```

### Impact

This issue significantly compromises the integrity of the emitted event content, potentially causing downstream problems for external systems and users relying on accurate event information. Addressing this error will enhance the precision of the contract's event logs.

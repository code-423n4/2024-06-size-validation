## Medium Issues


*Total <b>14</b> instances over <b>3</b> issues:*

|ID|Issue|Instances|
|-|:-|:-:|
| [M-1](#M-1) | No way to retrieve ETH from the contract | 1 |
| [M-2](#M-2) | Return values of `transfer()`/`transferFrom()` not checked | 11 |
| [M-3](#M-3) | latestRoundData() has no check for round completeness | 2 |

## Low Issues


*Total <b>203</b> instances over <b>24</b> issues:*

|ID|Issue|Instances|
|-|:-|:-:|
| [L-1](#L-1) | Missing zero address check in functions with address parameters | 21 |
| [L-2](#L-2) | Constructor / initialization function lacks parameter validation | 3 |
| [L-3](#L-3) | Loss of precision in divisions | 1 |
| [L-4](#L-4) | Missing checks for state variable assignments | 5 |
| [L-5](#L-5) | Missing zero address check in constructor | 1 |
| [L-6](#L-6) | Consider some checks for `address(0)` when setting address state variables | 2 |
| [L-7](#L-7) | Check storage gap for upgradable contracts | 1 |
| [L-8](#L-8) | Numbers downcast to `address`es may result in collisions | 2 |
| [L-9](#L-9) | Owner can renounce ownership | 1 |
| [L-10](#L-10) | Solidity version 0.8.20 or above may not work on other chains due to PUSH0 | 34 |
| [L-11](#L-11) | Some tokens may revert when large transfers are made | 14 |
| [L-12](#L-12) | Some tokens may revert when zero value transfers are made | 14 |
| [L-13](#L-13) | Tokens may be minted to `address(0)` | 2 |
| [L-14](#L-14) | Unsafe downcast | 4 |
| [L-15](#L-15) | Use Ownable2Step instead of Ownable | 1 |
| [L-16](#L-16) | Using zero as a parameter | 3 |
| [L-17](#L-17) | Missing zero address check in initializer | 1 |
| [L-18](#L-18) | Centralization Risk for trusted owners | 15 |
| [L-19](#L-19) | `decimals()` is not a part of the ERC-20 standard | 7 |
| [L-20](#L-20) | Initializers could be front-run | 1 |
| [L-21](#L-21) | `name()` is not a part of the ERC-20 standard | 3 |
| [L-22](#L-22) | `symbol()` is not a part of the ERC-20 standard | 3 |
| [L-23](#L-23) | Chainlink answer is not compared against min/max values | 2 |
| [L-24](#L-24) | Library has public/external functions | 62 |

## Medium Issues

<a name="M-1"></a> 
### [M-1] No way to retrieve ETH from the contract
The following contracts contain at least one `payable` function, yet the function does not utilise forwarded ETH, and the contract is missing functionality to withdraw ETH from the contract. This means that funds may become trapped in the contract indefinitely. Consider adding a withdraw/sweep function to contracts that are capable of receiving ether.

*There is one instance of this issue:*
```solidity
File: ./src/Size.sol

/// @audit Contract can receive Ether but there is lack of functionality to claim the Ethers from the contract
62: contract Size is ISize, SizeView, Initializable, AccessControlUpgradeable, PausableUpgradeable, UUPSUpgradeable {

```


<a name="M-2"></a> 
### [M-2] Return values of `transfer()`/`transferFrom()` not checked
Not all ERC20 implementations `revert()` when there's a failure in `transfer()` or `transferFrom()`. The function signature has a boolean return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually transfer anything.

<details>
<summary>
There are <b>11</b> instances (click to show):
</summary>

```solidity
File: ./src/libraries/actions/BuyCreditMarket.sol

195:         state.data.borrowAToken.transferFrom(msg.sender, borrower, cashAmountIn - fees);

196:         state.data.borrowAToken.transferFrom(msg.sender, state.feeConfig.feeRecipient, fees);

```

```solidity
File: ./src/libraries/actions/Claim.sol

56:         state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);

```

```solidity
File: ./src/libraries/actions/Liquidate.sol

118:         state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);

119:         state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);

```

```solidity
File: ./src/libraries/actions/LiquidateWithReplacement.sol

161:         state.data.borrowAToken.transferFrom(address(this), params.borrower, issuanceValue);

162:         state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);

```

```solidity
File: ./src/libraries/actions/Repay.sol

49:         state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);

```

```solidity
File: ./src/libraries/actions/SelfLiquidate.sol

70:         state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);

```

```solidity
File: ./src/libraries/actions/SellCreditMarket.sol

201:         state.data.borrowAToken.transferFrom(params.lender, msg.sender, cashAmountOut);

202:         state.data.borrowAToken.transferFrom(params.lender, state.feeConfig.feeRecipient, fees);

```

</details>


<a name="M-3"></a> 
### [M-3] latestRoundData() has no check for round completeness
No check for round completeness could lead to stale prices and wrong price return value, or outdated price. The functions rely on accurate price feed might not work as expected, sometimes can lead to fund loss.

*There are <b>2</b> instances of this issue:*
```solidity
File: ./src/oracle/PriceFeed.sol

66:             (, int256 answer, uint256 startedAt,,) = sequencerUptimeFeed.latestRoundData();

86:         (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();

```



## Low Issues

<a name="L-1"></a> 
### [L-1] Missing zero address check in functions with address parameters
Adding a zero address check for each address type parameter can prevent errors.

<details>
<summary>
There are <b>21</b> instances (click to show):
</summary>

```solidity
File: ./src/Size.sol

/// @audit missing zero check for `owner`
87:     function initialize(
            address owner,
            InitializeFeeConfigParams calldata f,
            InitializeRiskConfigParams calldata r,
            InitializeOracleParams calldata o,
            InitializeDataParams calldata d
        ) external initializer {

/// @audit missing zero check for `newImplementation`
107:     function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}

```

```solidity
File: ./src/SizeView.sol

/// @audit missing zero check for `borrower`
141:     function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {

/// @audit missing zero check for `lender`
157:     function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {

```

```solidity
File: ./src/libraries/AccountingLibrary.sol

/// @audit missing zero check for `lender`
/// @audit missing zero check for `borrower`
62:     function createDebtAndCreditPositions(
            State storage state,
            address lender,
            address borrower,
            uint256 futureValue,
            uint256 dueDate
        ) external returns (CreditPosition memory creditPosition) {

/// @audit missing zero check for `lender`
103:     function createCreditPosition(State storage state, uint256 exitCreditPositionId, address lender, uint256 credit)
             external
         {

```

```solidity
File: ./src/libraries/DepositTokenLibrary.sol

/// @audit missing zero check for `from`
/// @audit missing zero check for `to`
23:     function depositUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount) external {

/// @audit missing zero check for `from`
/// @audit missing zero check for `to`
34:     function withdrawUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount)
            external
        {

/// @audit missing zero check for `from`
/// @audit missing zero check for `to`
49:     function depositUnderlyingBorrowTokenToVariablePool(State storage state, address from, address to, uint256 amount)
            external
        {

/// @audit missing zero check for `from`
/// @audit missing zero check for `to`
74:     function withdrawUnderlyingTokenFromVariablePool(State storage state, address from, address to, uint256 amount)
            external
        {

```

```solidity
File: ./src/libraries/RiskLibrary.sol

/// @audit missing zero check for `account`
121:     function isUserUnderwater(State storage state, address account) public view returns (bool) {

/// @audit missing zero check for `account`
129:     function validateUserIsNotUnderwater(State storage state, address account) external view {

/// @audit missing zero check for `account`
140:     function validateUserIsNotBelowOpeningLimitBorrowCR(State storage state, address account) external view {

```

```solidity
File: ./src/token/NonTransferrableScaledToken.sol

/// @audit missing zero check for `to`
50:     function mintScaled(address to, uint256 scaledAmount) external onlyOwner {

/// @audit missing zero check for `from`
64:     function burnScaled(address from, uint256 scaledAmount) external onlyOwner {

/// @audit missing zero check for `from`
/// @audit missing zero check for `to`
76:     function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {

```

```solidity
File: ./src/token/NonTransferrableToken.sol

/// @audit missing zero check for `to`
29:     function mint(address to, uint256 value) external virtual onlyOwner {

/// @audit missing zero check for `from`
33:     function burn(address from, uint256 value) external virtual onlyOwner {

/// @audit missing zero check for `from`
/// @audit missing zero check for `to`
37:     function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {

/// @audit missing zero check for `to`
42:     function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {

/// @audit missing zero check for `spender`
46:     function allowance(address, address spender) public view virtual override returns (uint256) {

```

</details>


<a name="L-2"></a> 
### [L-2] Constructor / initialization function lacks parameter validation
Constructors and initialization functions play a critical role in contracts by setting important initial states when the contract is first deployed before the system starts. The parameters passed to the constructor and initialization functions directly affect the behavior of the contract / protocol. If incorrect parameters are provided, the system may fail to run, behave abnormally, be unstable, or lack security. Therefore, it's crucial to carefully check each parameter in the constructor and initialization functions. If an exception is found, the transaction should be rolled back.

*There are <b>3</b> instances of this issue:*
```solidity
File: ./src/Size.sol

/// @audit `owner` not validated
/// @audit `f` not validated
/// @audit `r` not validated
/// @audit `o` not validated
/// @audit `d` not validated
87:     function initialize(

```

```solidity
File: ./src/token/NonTransferrableScaledToken.sol

/// @audit `owner_` not validated
/// @audit `name_` not validated
/// @audit `symbol_` not validated
/// @audit `decimals_` not validated
25:     constructor(

```

```solidity
File: ./src/token/NonTransferrableToken.sol

/// @audit `name_` not validated
/// @audit `symbol_` not validated
18:     constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)

```


<a name="L-3"></a> 
### [L-3] Loss of precision in divisions
Division by large numbers may result in the result being zero, due to solidity not supporting fractions. Consider requiring a minimum amount for the numerator to ensure that it is always larger than the denominator.

*There is one instance of this issue:*
```solidity
File: ./src/libraries/actions/SellCreditMarket.sol

175:                     : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),

```


<a name="L-4"></a> 
### [L-4] Missing checks for state variable assignments
There are some missing checks in these functions, and this could lead to unexpected scenarios. Consider always adding a sanity check for state variables.

*There are <b>5</b> instances of this issue:*
```solidity
File: ./src/oracle/PriceFeed.sol

/// @audit `_baseStalePriceInterval`
55:         baseStalePriceInterval = _baseStalePriceInterval;

/// @audit `_quoteStalePriceInterval`
56:         quoteStalePriceInterval = _quoteStalePriceInterval;

```

```solidity
File: ./src/token/NonTransferrableScaledToken.sol

/// @audit `variablePool_`
37:         variablePool = variablePool_;

/// @audit `underlyingToken_`
38:         underlyingToken = underlyingToken_;

```

```solidity
File: ./src/token/NonTransferrableToken.sol

/// @audit `decimals_`
26:         _decimals = decimals_;

```


<a name="L-5"></a> 
### [L-5] Missing zero address check in constructor
Constructors often take address parameters to initialize important components of a contract, such as owner or linked contracts. However, without a checking, there's a risk that an address parameter could be mistakenly set to the zero address (0x0). This could be due to an error or oversight during contract deployment. A zero address in a crucial role can cause serious issues, as it cannot perform actions like a normal address, and any funds sent to it will be irretrievable. It's therefore crucial to include a zero address check in constructors to prevent such potential problems. If a zero address is detected, the constructor should revert the transaction.

*There is one instance of this issue:*
```solidity
File: ./src/token/NonTransferrableScaledToken.sol

/// @audit missing zero check for `owner_`
25:     constructor(
            IPool variablePool_,
            IERC20Metadata underlyingToken_,
            address owner_,
            string memory name_,
            string memory symbol_,
            uint8 decimals_
        ) NonTransferrableToken(owner_, name_, symbol_, decimals_) {

```


<a name="L-6"></a> 
### [L-6] Consider some checks for `address(0)` when setting address state variables
Check for zero-address to avoid the risk of setting `address(0)` for state variables after an update.

*There are <b>2</b> instances of this issue:*
```solidity
File: ./src/token/NonTransferrableScaledToken.sol

/// @audit `variablePool_`
37:         variablePool = variablePool_;

/// @audit `underlyingToken_`
38:         underlyingToken = underlyingToken_;

```


<a name="L-7"></a> 
### [L-7] Check storage gap for upgradable contracts
Each upgradable contract should include a state variable (usually named `__gap`) to provide reserved space in storage. This allows the team to freely add new state variables in the future upgrades without compromising the storage compatibility with existing deployments.

*There is one instance of this issue:*
```solidity
File: ./src/Size.sol

62: contract Size is ISize, SizeView, Initializable, AccessControlUpgradeable, PausableUpgradeable, UUPSUpgradeable {

```


<a name="L-8"></a> 
### [L-8] Numbers downcast to `address`es may result in collisions
If a number is downcast to an `address` the upper bytes are truncated, which may mean that more than one value will map to the `address`.

*There are <b>2</b> instances of this issue:*
```solidity
File: ./src/libraries/actions/UpdateConfig.sol

134:             state.feeConfig.feeRecipient = address(uint160(params.value));

136:             state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));

```


<a name="L-9"></a> 
### [L-9] Owner can renounce ownership
Each of the following contracts implements or inherits the `renounceOwnership()` function. This can represent a certain risk if the ownership is renounced for any other reason than by design. Renouncing ownership will leave the contract without an owner, thereby removing any functionality that is only available to the owner.

*There is one instance of this issue:*
```solidity
File: ./src/token/NonTransferrableToken.sol

14: contract NonTransferrableToken is Ownable, ERC20 {

```


<a name="L-10"></a> 
### [L-10] Solidity version 0.8.20 or above may not work on other chains due to PUSH0
Solidity version 0.8.20 or above uses the new [Shanghai EVM](https://blog.soliditylang.org/2023/05/10/solidity-0.8.20-release-announcement/#important-note) which introduces the PUSH0 opcode. This op code may not yet be implemented on all evm-chains or Layer2s, so deployment on these chains will fail. Consider using an earlier solidity version.

<details>
<summary>
There are <b>34</b> instances (click to show):
</summary>

```solidity
File: ./src/Size.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/SizeStorage.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/SizeView.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/SizeViewData.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/AccountingLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/CapsLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/DepositTokenLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/Errors.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/Events.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/LoanLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/Math.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/Multicall.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/OfferLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/RiskLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/YieldCurveLibrary.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/BuyCreditLimit.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/BuyCreditMarket.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Claim.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Compensate.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Deposit.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Initialize.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Liquidate.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/LiquidateWithReplacement.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Repay.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/SelfLiquidate.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/SellCreditLimit.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/SellCreditMarket.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/SetUserConfiguration.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/UpdateConfig.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/libraries/actions/Withdraw.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/oracle/IPriceFeed.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/oracle/PriceFeed.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/token/NonTransferrableScaledToken.sol

2: pragma solidity 0.8.23;

```

```solidity
File: ./src/token/NonTransferrableToken.sol

2: pragma solidity 0.8.23;

```

</details>


<a name="L-11"></a> 
### [L-11] Some tokens may revert when large transfers are made
Tokens such as COMP or UNI will revert when an address' balance reaches `type(uint96).max`. Ensure that the calls below can be broken up into smaller batches if necessary.

<details>
<summary>
There are <b>14</b> instances (click to show):
</summary>

```solidity
File: ./src/libraries/DepositTokenLibrary.sol

25:         underlyingCollateralToken.safeTransferFrom(from, address(this), amount);

39:         underlyingCollateralToken.safeTransfer(to, amount);

52:         state.data.underlyingBorrowToken.safeTransferFrom(from, address(this), amount);

```

```solidity
File: ./src/libraries/actions/BuyCreditMarket.sol

195:         state.data.borrowAToken.transferFrom(msg.sender, borrower, cashAmountIn - fees);

196:         state.data.borrowAToken.transferFrom(msg.sender, state.feeConfig.feeRecipient, fees);

```

```solidity
File: ./src/libraries/actions/Claim.sol

56:         state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);

```

```solidity
File: ./src/libraries/actions/Liquidate.sol

118:         state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);

119:         state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);

```

```solidity
File: ./src/libraries/actions/LiquidateWithReplacement.sol

161:         state.data.borrowAToken.transferFrom(address(this), params.borrower, issuanceValue);

162:         state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);

```

```solidity
File: ./src/libraries/actions/Repay.sol

49:         state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);

```

```solidity
File: ./src/libraries/actions/SelfLiquidate.sol

70:         state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);

```

```solidity
File: ./src/libraries/actions/SellCreditMarket.sol

201:         state.data.borrowAToken.transferFrom(params.lender, msg.sender, cashAmountOut);

202:         state.data.borrowAToken.transferFrom(params.lender, state.feeConfig.feeRecipient, fees);

```

</details>


<a name="L-12"></a> 
### [L-12] Some tokens may revert when zero value transfers are made
Despite the fact that [EIP-20 states](https://github.com/ethereum/EIPs/blob/7500ac4fc1bbdfaf684e7ef851f798f6b667b2fe/EIPS/eip-20.md?plain=1#L116) that zero-value transfers must be accepted, some tokens, such as LEND, will revert if this is attempted, which may cause transactions that involve other tokens (such as batch operations) to fully revert. Consider skipping the transfer if the amount is zero, which will also save gas.

<details>
<summary>
There are <b>14</b> instances (click to show):
</summary>

```solidity
File: ./src/libraries/DepositTokenLibrary.sol

25:         underlyingCollateralToken.safeTransferFrom(from, address(this), amount);

39:         underlyingCollateralToken.safeTransfer(to, amount);

52:         state.data.underlyingBorrowToken.safeTransferFrom(from, address(this), amount);

```

```solidity
File: ./src/libraries/actions/BuyCreditMarket.sol

195:         state.data.borrowAToken.transferFrom(msg.sender, borrower, cashAmountIn - fees);

196:         state.data.borrowAToken.transferFrom(msg.sender, state.feeConfig.feeRecipient, fees);

```

```solidity
File: ./src/libraries/actions/Claim.sol

56:         state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);

```

```solidity
File: ./src/libraries/actions/Liquidate.sol

118:         state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);

119:         state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);

```

```solidity
File: ./src/libraries/actions/LiquidateWithReplacement.sol

161:         state.data.borrowAToken.transferFrom(address(this), params.borrower, issuanceValue);

162:         state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);

```

```solidity
File: ./src/libraries/actions/Repay.sol

49:         state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);

```

```solidity
File: ./src/libraries/actions/SelfLiquidate.sol

70:         state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);

```

```solidity
File: ./src/libraries/actions/SellCreditMarket.sol

201:         state.data.borrowAToken.transferFrom(params.lender, msg.sender, cashAmountOut);

202:         state.data.borrowAToken.transferFrom(params.lender, state.feeConfig.feeRecipient, fees);

```

</details>


<a name="L-13"></a> 
### [L-13] Tokens may be minted to `address(0)`

*There are <b>2</b> instances of this issue:*
```solidity
File: ./src/token/NonTransferrableScaledToken.sol

42:     function mint(address, uint256) external view override onlyOwner {
            revert Errors.NOT_SUPPORTED();
        }

```

```solidity
File: ./src/token/NonTransferrableToken.sol

29:     function mint(address to, uint256 value) external virtual onlyOwner {
            _mint(to, value);
        }

```


<a name="L-14"></a> 
### [L-14] Unsafe downcast
When a type is downcast to a smaller type, the higher order bits are truncated, effectively applying a modulo to the original value. Without any other checks, this wrapping will lead to unexpected behavior and bugs.

*There are <b>4</b> instances of this issue:*
```solidity
File: ./src/Size.sol

/// @audit uint256 -> uint64
127:         state.oracle.variablePoolBorrowRateUpdatedAt = uint64(block.timestamp);

```

```solidity
File: ./src/libraries/actions/UpdateConfig.sol

/// @audit uint256 -> uint160
134:             state.feeConfig.feeRecipient = address(uint160(params.value));

/// @audit uint256 -> uint160
136:             state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));

/// @audit uint256 -> uint64
138:             state.oracle.variablePoolBorrowRateStaleRateInterval = uint64(params.value);

```


<a name="L-15"></a> 
### [L-15] Use Ownable2Step instead of Ownable
`Ownable2Step` and `Ownable2StepUpgradeable` prevent the contract ownership from mistakenly being transferred to an address that cannot handle it (e.g. due to a typo in the address), by requiring that the recipient of the owner permissions actively accept via a contract call of its own.

*There is one instance of this issue:*
```solidity
File: ./src/token/NonTransferrableToken.sol

14: contract NonTransferrableToken is Ownable, ERC20 {

```


<a name="L-16"></a> 
### [L-16] Using zero as a parameter
Taking `0` as a valid argument in Solidity without checks can lead to severe security issues. A historical example is the infamous `0x0` address bug where numerous tokens were lost. This happens because 0 can be interpreted as an uninitialized `address`, leading to transfers to the 0x0 address, effectively burning tokens. Moreover, `0` as a denominator in division operations would cause a runtime exception. It's also often indicative of a logical error in the caller's code. It's important to always validate input and handle edge cases like `0` appropriately. Use `require()` statements to enforce conditions and provide clear error messages to facilitate debugging and safer code.

*There are <b>3</b> instances of this issue:*
```solidity
File: ./src/libraries/AccountingLibrary.sol

70:             DebtPosition({borrower: borrower, futureValue: futureValue, dueDate: dueDate, liquidityIndexAtRepayment: 0});

```

```solidity
File: ./src/token/NonTransferrableScaledToken.sol

52:         emit TransferUnscaled(address(0), to, _unscale(scaledAmount));

66:         emit TransferUnscaled(from, address(0), _unscale(scaledAmount));

```


<a name="L-17"></a> 
### [L-17] Missing zero address check in initializer

*There is one instance of this issue:*
```solidity
File: ./src/Size.sol

/// @audit missing zero check for `owner`
87:     function initialize(
            address owner,
            InitializeFeeConfigParams calldata f,
            InitializeRiskConfigParams calldata r,
            InitializeOracleParams calldata o,
            InitializeDataParams calldata d
        ) external initializer {

```


<a name="L-18"></a> 
### [L-18] Centralization Risk for trusted owners
Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details>
<summary>
There are <b>15</b> instances (click to show):
</summary>

```solidity
File: ./src/Size.sol

107:     function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}

110:     function updateConfig(UpdateConfigParams calldata params)
             external
             override(ISizeAdmin)
             onlyRole(DEFAULT_ADMIN_ROLE)

120:     function setVariablePoolBorrowRate(uint128 borrowRate)
             external
             override(ISizeAdmin)
             onlyRole(BORROW_RATE_UPDATER_ROLE)

132:     function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {

137:     function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {

229:     function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)
             external
             payable
             override(ISize)
             whenNotPaused
             onlyRole(KEEPER_ROLE)

```

```solidity
File: ./src/token/NonTransferrableScaledToken.sol

42:     function mint(address, uint256) external view override onlyOwner {

50:     function mintScaled(address to, uint256 scaledAmount) external onlyOwner {

56:     function burn(address, uint256) external view override onlyOwner {

64:     function burnScaled(address from, uint256 scaledAmount) external onlyOwner {

76:     function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {

```

```solidity
File: ./src/token/NonTransferrableToken.sol

29:     function mint(address to, uint256 value) external virtual onlyOwner {

33:     function burn(address from, uint256 value) external virtual onlyOwner {

37:     function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {

42:     function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {

```

</details>


<a name="L-19"></a> 
### [L-19] `decimals()` is not a part of the ERC-20 standard
The `decimals()` function is not a part of the ERC-20 standard, and was added later as an optional extension. As such, some valid ERC20 tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.

*There are <b>7</b> instances of this issue:*
```solidity
File: ./src/libraries/actions/Initialize.sol

151:         if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {

152:             revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingCollateralToken).decimals());

159:         if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {

160:             revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingBorrowToken).decimals());

243:             IERC20Metadata(state.data.underlyingCollateralToken).decimals()

251:             IERC20Metadata(state.data.underlyingBorrowToken).decimals()

257:             IERC20Metadata(state.data.underlyingBorrowToken).decimals()

```


<a name="L-20"></a> 
### [L-20] Initializers could be front-run
Initializers could be front-run, allowing an attacker to either set their own values, take ownership of the contract, and in the best case forcing a re-deployment

*There is one instance of this issue:*
```solidity
File: ./src/Size.sol

87:     function initialize(

```


<a name="L-21"></a> 
### [L-21] `name()` is not a part of the ERC-20 standard
The `name()` function is not a part of the ERC-20 standard, and was added later as an optional extension. As such, some valid ERC20 tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.

*There are <b>3</b> instances of this issue:*
```solidity
File: ./src/libraries/actions/Initialize.sol

241:             string.concat("Size ", IERC20Metadata(state.data.underlyingCollateralToken).name()),

249:             string.concat("Size Scaled ", IERC20Metadata(state.data.underlyingBorrowToken).name()),

255:             string.concat("Size Debt ", IERC20Metadata(state.data.underlyingBorrowToken).name()),

```


<a name="L-22"></a> 
### [L-22] `symbol()` is not a part of the ERC-20 standard
The `symbol()` function is not a part of the ERC-20 standard, and was added later as an optional extension. As such, some valid ERC20 tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.

*There are <b>3</b> instances of this issue:*
```solidity
File: ./src/libraries/actions/Initialize.sol

242:             string.concat("sz", IERC20Metadata(state.data.underlyingCollateralToken).symbol()),

250:             string.concat("sza", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),

256:             string.concat("szDebt", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),

```


<a name="L-23"></a> 
### [L-23] Chainlink answer is not compared against min/max values
Some check like this can be added to avoid returning of the min price or the max price in case of the price crashes:
```solidity
	require(answer < _maxPrice, "Upper price bound breached");
	require(answer > _minPrice, "Lower price bound breached");
```

*There are <b>2</b> instances of this issue:*
```solidity
File: ./src/oracle/PriceFeed.sol

66:             (, int256 answer, uint256 startedAt,,) = sequencerUptimeFeed.latestRoundData();

86:         (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();

```


<a name="L-24"></a> 
### [L-24] Library has public/external functions
Libraries in Solidity are meant to be static collections of functions. They are deployed once and their code is reused by contracts through DELEGATECALL, which executes the library's code in the context of the calling contract. Public or external functions in libraries can be misleading because libraries are not supposed to maintain state or have external interactions in the way contracts do. Designing libraries with these kinds of functions goes against their intended purpose and can lead to confusion or misuse. For state management or external interactions, using contracts instead of libraries is more appropriate. Libraries should focus on utility functions that can operate on the data passed to them without side effects, enhancing code reuse and gas efficiency.

<details>
<summary>
There are <b>62</b> instances (click to show):
</summary>

```solidity
File: ./src/libraries/AccountingLibrary.sol

42:     function repayDebt(State storage state, uint256 debtPositionId, uint256 repayAmount) public {

62:     function createDebtAndCreditPositions(
            State storage state,
            address lender,
            address borrower,
            uint256 futureValue,
            uint256 dueDate
        ) external returns (CreditPosition memory creditPosition) {

103:     function createCreditPosition(State storage state, uint256 exitCreditPositionId, address lender, uint256 credit)
             external
         {

137:     function reduceCredit(State storage state, uint256 creditPositionId, uint256 amount) public {

```

```solidity
File: ./src/libraries/CapsLibrary.sol

19:     function validateBorrowATokenIncreaseLteDebtTokenDecrease(
            State storage state,
            uint256 borrowATokenSupplyBefore,
            uint256 debtTokenSupplyBefore,
            uint256 borrowATokenSupplyAfter,
            uint256 debtTokenSupplyAfter
        ) external view {

52:     function validateBorrowATokenCap(State storage state) external view {

67:     function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {

```

```solidity
File: ./src/libraries/DepositTokenLibrary.sol

23:     function depositUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount) external {

34:     function withdrawUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount)
            external
        {

49:     function depositUnderlyingBorrowTokenToVariablePool(State storage state, address from, address to, uint256 amount)
            external
        {

74:     function withdrawUnderlyingTokenFromVariablePool(State storage state, address from, address to, uint256 amount)
            external
        {

```

```solidity
File: ./src/libraries/LoanLibrary.sol

82:     function getDebtPosition(State storage state, uint256 debtPositionId) public view returns (DebtPosition memory) {

97:     function getCreditPosition(State storage state, uint256 creditPositionId)
            public
            view
            returns (CreditPosition memory)
        {

116:     function getDebtPositionByCreditPositionId(State storage state, uint256 creditPositionId)
             public
             view
             returns (DebtPosition memory)
         {

129:     function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {

140:     function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)
             public
             view
             returns (uint256)
         {

162:     function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)
             public
             view
             returns (uint256)
         {

```

```solidity
File: ./src/libraries/RiskLibrary.sol

21:     function validateMinimumCredit(State storage state, uint256 credit) public view {

31:     function validateMinimumCreditOpening(State storage state, uint256 credit) public view {

41:     function validateTenor(State storage state, uint256 tenor) public view {

53:     function collateralRatio(State storage state, address account) public view returns (uint256) {

71:     function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)
            public
            view
            returns (bool)
        {

104:     function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {

121:     function isUserUnderwater(State storage state, address account) public view returns (bool) {

129:     function validateUserIsNotUnderwater(State storage state, address account) external view {

140:     function validateUserIsNotBelowOpeningLimitBorrowCR(State storage state, address account) external view {

```

```solidity
File: ./src/libraries/YieldCurveLibrary.sol

115:     function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)
             external
             view
             returns (uint256)
         {

```

```solidity
File: ./src/libraries/actions/BuyCreditLimit.sol

29:     function validateBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external view {

57:     function executeBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/BuyCreditMarket.sol

51:     function validateBuyCreditMarket(State storage state, BuyCreditMarketParams calldata params) external view {

121:     function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
             external
             returns (uint256 cashAmountIn)
         {

```

```solidity
File: ./src/libraries/actions/Claim.sol

31:     function validateClaim(State storage state, ClaimParams calldata params) external view {

48:     function executeClaim(State storage state, ClaimParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/Compensate.sol

42:     function validateCompensate(State storage state, CompensateParams calldata params) external view {

106:     function executeCompensate(State storage state, CompensateParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/Deposit.sol

36:     function validateDeposit(State storage state, DepositParams calldata params) external view {

64:     function executeDeposit(State storage state, DepositParams calldata params) public {

```

```solidity
File: ./src/libraries/actions/Initialize.sol

175:     function validateInitialize(
             State storage,
             address owner,
             InitializeFeeConfigParams memory f,
             InitializeRiskConfigParams memory r,
             InitializeOracleParams memory o,
             InitializeDataParams memory d
         ) external view {

267:     function executeInitialize(
             State storage state,
             InitializeFeeConfigParams memory f,
             InitializeRiskConfigParams memory r,
             InitializeOracleParams memory o,
             InitializeDataParams memory d
         ) external {

```

```solidity
File: ./src/libraries/actions/Liquidate.sol

37:     function validateLiquidate(State storage state, LiquidateParams calldata params) external view {

59:     function validateMinimumCollateralProfit(
            State storage,
            LiquidateParams calldata params,
            uint256 liquidatorProfitCollateralToken
        ) external pure {

75:     function executeLiquidate(State storage state, LiquidateParams calldata params)
            external
            returns (uint256 liquidatorProfitCollateralToken)
        {

```

```solidity
File: ./src/libraries/actions/LiquidateWithReplacement.sol

47:     function validateLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)
            external
            view
        {

99:     function validateMinimumCollateralProfit(
            State storage state,
            LiquidateWithReplacementParams calldata params,
            uint256 liquidatorProfitCollateralToken
        ) external pure {

120:     function executeLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)
             external
             returns (uint256 issuanceValue, uint256 liquidatorProfitCollateralToken, uint256 liquidatorProfitBorrowToken)
         {

```

```solidity
File: ./src/libraries/actions/Repay.sol

33:     function validateRepay(State storage state, RepayParams calldata params) external view {

46:     function executeRepay(State storage state, RepayParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/SelfLiquidate.sol

34:     function validateSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external view {

59:     function executeSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/SellCreditLimit.sol

25:     function validateSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external view {

44:     function executeSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/SellCreditMarket.sol

51:     function validateSellCreditMarket(State storage state, SellCreditMarketParams calldata params) external view {

127:     function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)
             external
             returns (uint256 cashAmountOut)
         {

```

```solidity
File: ./src/libraries/actions/SetUserConfiguration.sol

31:     function validateSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params)
            external
            view
        {

63:     function executeSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/UpdateConfig.sol

42:     function feeConfigParams(State storage state) public view returns (InitializeFeeConfigParams memory) {

56:     function riskConfigParams(State storage state) public view returns (InitializeRiskConfigParams memory) {

70:     function oracleParams(State storage state) public view returns (InitializeOracleParams memory) {

79:     function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {

86:     function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {

```

```solidity
File: ./src/libraries/actions/Withdraw.sol

29:     function validateWithdraw(State storage state, WithdrawParams calldata params) external view {

52:     function executeWithdraw(State storage state, WithdrawParams calldata params) public {

```

</details>



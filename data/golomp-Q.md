## L-1: Centralization Risk for trusted owners

Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details><summary>16 Found Instances</summary>


- Found in src/Size.sol [Line: 107](src/Size.sol#L107)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}
	```

- Found in src/Size.sol [Line: 113](src/Size.sol#L113)

	```solidity
	        onlyRole(DEFAULT_ADMIN_ROLE)
	```

- Found in src/Size.sol [Line: 123](src/Size.sol#L123)

	```solidity
	        onlyRole(BORROW_RATE_UPDATER_ROLE)
	```

- Found in src/Size.sol [Line: 132](src/Size.sol#L132)

	```solidity
	    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {
	```

- Found in src/Size.sol [Line: 137](src/Size.sol#L137)

	```solidity
	    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {
	```

- Found in src/Size.sol [Line: 234](src/Size.sol#L234)

	```solidity
	        onlyRole(KEEPER_ROLE)
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 42](src/token/NonTransferrableScaledToken.sol#L42)

	```solidity
	    function mint(address, uint256) external view override onlyOwner {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 50](src/token/NonTransferrableScaledToken.sol#L50)

	```solidity
	    function mintScaled(address to, uint256 scaledAmount) external onlyOwner {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 56](src/token/NonTransferrableScaledToken.sol#L56)

	```solidity
	    function burn(address, uint256) external view override onlyOwner {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 64](src/token/NonTransferrableScaledToken.sol#L64)

	```solidity
	    function burnScaled(address from, uint256 scaledAmount) external onlyOwner {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 76](src/token/NonTransferrableScaledToken.sol#L76)

	```solidity
	    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 14](src/token/NonTransferrableToken.sol#L14)

	```solidity
	contract NonTransferrableToken is Ownable, ERC20 {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 29](src/token/NonTransferrableToken.sol#L29)

	```solidity
	    function mint(address to, uint256 value) external virtual onlyOwner {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 33](src/token/NonTransferrableToken.sol#L33)

	```solidity
	    function burn(address from, uint256 value) external virtual onlyOwner {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 37](src/token/NonTransferrableToken.sol#L37)

	```solidity
	    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 42](src/token/NonTransferrableToken.sol#L42)

	```solidity
	    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {
	```

</details>



## L-2: Unsafe ERC20 Operations should not be used

ERC20 functions may not behave as expected. For example: return values are not always meaningful. It is recommended to use OpenZeppelin's SafeERC20 library.

<details><summary>13 Found Instances</summary>


- Found in src/libraries/actions/BuyCreditMarket.sol [Line: 195](src/libraries/actions/BuyCreditMarket.sol#L195)

	```solidity
	        state.data.borrowAToken.transferFrom(msg.sender, borrower, cashAmountIn - fees);
	```

- Found in src/libraries/actions/BuyCreditMarket.sol [Line: 196](src/libraries/actions/BuyCreditMarket.sol#L196)

	```solidity
	        state.data.borrowAToken.transferFrom(msg.sender, state.feeConfig.feeRecipient, fees);
	```

- Found in src/libraries/actions/Claim.sol [Line: 56](src/libraries/actions/Claim.sol#L56)

	```solidity
	        state.data.borrowAToken.transferFrom(address(this), creditPosition.lender, claimAmount);
	```

- Found in src/libraries/actions/Compensate.sol [Line: 152](src/libraries/actions/Compensate.sol#L152)

	```solidity
	            state.data.collateralToken.transferFrom(
	```

- Found in src/libraries/actions/Liquidate.sol [Line: 118](src/libraries/actions/Liquidate.sol#L118)

	```solidity
	        state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);
	```

- Found in src/libraries/actions/Liquidate.sol [Line: 119](src/libraries/actions/Liquidate.sol#L119)

	```solidity
	        state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
	```

- Found in src/libraries/actions/Liquidate.sol [Line: 120](src/libraries/actions/Liquidate.sol#L120)

	```solidity
	        state.data.collateralToken.transferFrom(
	```

- Found in src/libraries/actions/LiquidateWithReplacement.sol [Line: 161](src/libraries/actions/LiquidateWithReplacement.sol#L161)

	```solidity
	        state.data.borrowAToken.transferFrom(address(this), params.borrower, issuanceValue);
	```

- Found in src/libraries/actions/LiquidateWithReplacement.sol [Line: 162](src/libraries/actions/LiquidateWithReplacement.sol#L162)

	```solidity
	        state.data.borrowAToken.transferFrom(address(this), state.feeConfig.feeRecipient, liquidatorProfitBorrowToken);
	```

- Found in src/libraries/actions/Repay.sol [Line: 49](src/libraries/actions/Repay.sol#L49)

	```solidity
	        state.data.borrowAToken.transferFrom(msg.sender, address(this), debtPosition.futureValue);
	```

- Found in src/libraries/actions/SelfLiquidate.sol [Line: 70](src/libraries/actions/SelfLiquidate.sol#L70)

	```solidity
	        state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, assignedCollateral);
	```

- Found in src/libraries/actions/SellCreditMarket.sol [Line: 201](src/libraries/actions/SellCreditMarket.sol#L201)

	```solidity
	        state.data.borrowAToken.transferFrom(params.lender, msg.sender, cashAmountOut);
	```

- Found in src/libraries/actions/SellCreditMarket.sol [Line: 202](src/libraries/actions/SellCreditMarket.sol#L202)

	```solidity
	        state.data.borrowAToken.transferFrom(params.lender, state.feeConfig.feeRecipient, fees);
	```

</details>



## L-3: `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>21 Found Instances</summary>


- Found in src/Size.sol [Line: 132](src/Size.sol#L132)

	```solidity
	    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {
	```

- Found in src/Size.sol [Line: 137](src/Size.sol#L137)

	```solidity
	    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {
	```

- Found in src/Size.sol [Line: 142](src/Size.sol#L142)

	```solidity
	    function multicall(bytes[] calldata _data)
	```

- Found in src/Size.sol [Line: 153](src/Size.sol#L153)

	```solidity
	    function deposit(DepositParams calldata params) public payable override(ISize) whenNotPaused {
	```

- Found in src/libraries/CapsLibrary.sol [Line: 67](src/libraries/CapsLibrary.sol#L67)

	```solidity
	    function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {
	```

- Found in src/libraries/LoanLibrary.sol [Line: 122](src/libraries/LoanLibrary.sol#L122)

	```solidity
	    function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {
	```

- Found in src/libraries/LoanLibrary.sol [Line: 170](src/libraries/LoanLibrary.sol#L170)

	```solidity
	    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)
	```

- Found in src/libraries/RiskLibrary.sol [Line: 21](src/libraries/RiskLibrary.sol#L21)

	```solidity
	    function validateMinimumCredit(State storage state, uint256 credit) public view {
	```

- Found in src/libraries/RiskLibrary.sol [Line: 31](src/libraries/RiskLibrary.sol#L31)

	```solidity
	    function validateMinimumCreditOpening(State storage state, uint256 credit) public view {
	```

- Found in src/libraries/RiskLibrary.sol [Line: 41](src/libraries/RiskLibrary.sol#L41)

	```solidity
	    function validateTenor(State storage state, uint256 tenor) public view {
	```

- Found in src/libraries/RiskLibrary.sol [Line: 71](src/libraries/RiskLibrary.sol#L71)

	```solidity
	    function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)
	```

- Found in src/libraries/RiskLibrary.sol [Line: 104](src/libraries/RiskLibrary.sol#L104)

	```solidity
	    function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {
	```

- Found in src/libraries/actions/Deposit.sol [Line: 64](src/libraries/actions/Deposit.sol#L64)

	```solidity
	    function executeDeposit(State storage state, DepositParams calldata params) public {
	```

- Found in src/libraries/actions/Withdraw.sol [Line: 52](src/libraries/actions/Withdraw.sol#L52)

	```solidity
	    function executeWithdraw(State storage state, WithdrawParams calldata params) public {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 76](src/token/NonTransferrableScaledToken.sol#L76)

	```solidity
	    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 105](src/token/NonTransferrableScaledToken.sol#L105)

	```solidity
	    function balanceOf(address account) public view override returns (uint256) {
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 117](src/token/NonTransferrableScaledToken.sol#L117)

	```solidity
	    function totalSupply() public view override returns (uint256) {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 42](src/token/NonTransferrableToken.sol#L42)

	```solidity
	    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 46](src/token/NonTransferrableToken.sol#L46)

	```solidity
	    function allowance(address, address spender) public view virtual override returns (uint256) {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 50](src/token/NonTransferrableToken.sol#L50)

	```solidity
	    function approve(address, uint256) public virtual override returns (bool) {
	```

- Found in src/token/NonTransferrableToken.sol [Line: 54](src/token/NonTransferrableToken.sol#L54)

	```solidity
	    function decimals() public view virtual override returns (uint8) {
	```

</details>



## L-4: Define and use `constant` variables instead of using literals

If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

<details><summary>2 Found Instances</summary>


- Found in src/libraries/actions/Initialize.sol [Line: 151](src/libraries/actions/Initialize.sol#L151)

	```solidity
	        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 159](src/libraries/actions/Initialize.sol#L159)

	```solidity
	        if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {
	```

</details>



## L-5: Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>12 Found Instances</summary>


- Found in src/libraries/Events.sol [Line: 18](src/libraries/Events.sol#L18)

	```solidity
	    event Initialize(
	```

- Found in src/libraries/Events.sol [Line: 21](src/libraries/Events.sol#L21)

	```solidity
	    event Deposit(address indexed token, address indexed to, uint256 amount);
	```

- Found in src/libraries/Events.sol [Line: 22](src/libraries/Events.sol#L22)

	```solidity
	    event Withdraw(address indexed token, address indexed to, uint256 amount);
	```

- Found in src/libraries/Events.sol [Line: 23](src/libraries/Events.sol#L23)

	```solidity
	    event UpdateConfig(string indexed key, uint256 value);
	```

- Found in src/libraries/Events.sol [Line: 33](src/libraries/Events.sol#L33)

	```solidity
	    event SellCreditLimit(
	```

- Found in src/libraries/Events.sol [Line: 45](src/libraries/Events.sol#L45)

	```solidity
	    event BuyCreditLimit(
	```

- Found in src/libraries/Events.sol [Line: 53](src/libraries/Events.sol#L53)

	```solidity
	    event Liquidate(
	```

- Found in src/libraries/Events.sol [Line: 57](src/libraries/Events.sol#L57)

	```solidity
	    event LiquidateWithReplacement(
	```

- Found in src/libraries/Events.sol [Line: 60](src/libraries/Events.sol#L60)

	```solidity
	    event Compensate(
	```

- Found in src/libraries/Events.sol [Line: 89](src/libraries/Events.sol#L89)

	```solidity
	    event UpdateDebtPosition(
	```

- Found in src/libraries/Events.sol [Line: 92](src/libraries/Events.sol#L92)

	```solidity
	    event UpdateCreditPosition(uint256 indexed creditPositionId, address indexed lender, uint256 credit, bool forSale);
	```

- Found in src/token/NonTransferrableScaledToken.sol [Line: 23](src/token/NonTransferrableScaledToken.sol#L23)

	```solidity
	    event TransferUnscaled(address indexed from, address indexed to, uint256 value);
	```

</details>



## L-6: Empty Block

Consider removing empty blocks.

<details><summary>2 Found Instances</summary>


- Found in src/Size.sol [Line: 107](src/Size.sol#L107)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}
	```

- Found in src/libraries/actions/UpdateConfig.sol [Line: 79](src/libraries/actions/UpdateConfig.sol#L79)

	```solidity
	    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {
	```

</details>



## L-7: Internal functions called only once can be inlined

Instead of separating the logic into a separate function, consider inlining the logic into the calling function. This can reduce the number of function calls and improve readability.

<details><summary>12 Found Instances</summary>


- Found in src/libraries/Math.sol [Line: 27](src/libraries/Math.sol#L27)

	```solidity
	    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {
	```

- Found in src/libraries/OfferLibrary.sol [Line: 48](src/libraries/OfferLibrary.sol#L48)

	```solidity
	    function getAPRByTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
	```

- Found in src/libraries/OfferLibrary.sol [Line: 76](src/libraries/OfferLibrary.sol#L76)

	```solidity
	    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
	```

- Found in src/libraries/actions/Initialize.sol [Line: 62](src/libraries/actions/Initialize.sol#L62)

	```solidity
	    function validateOwner(address owner) internal pure {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 70](src/libraries/actions/Initialize.sol#L70)

	```solidity
	    function validateInitializeFeeConfigParams(InitializeFeeConfigParams memory f) internal pure {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 98](src/libraries/actions/Initialize.sol#L98)

	```solidity
	    function validateInitializeRiskConfigParams(InitializeRiskConfigParams memory r) internal pure {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 132](src/libraries/actions/Initialize.sol#L132)

	```solidity
	    function validateInitializeOracleParams(InitializeOracleParams memory o) internal view {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 146](src/libraries/actions/Initialize.sol#L146)

	```solidity
	    function validateInitializeDataParams(InitializeDataParams memory d) internal view {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 193](src/libraries/actions/Initialize.sol#L193)

	```solidity
	    function executeInitializeFeeConfig(State storage state, InitializeFeeConfigParams memory f) internal {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 207](src/libraries/actions/Initialize.sol#L207)

	```solidity
	    function executeInitializeRiskConfig(State storage state, InitializeRiskConfigParams memory r) internal {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 222](src/libraries/actions/Initialize.sol#L222)

	```solidity
	    function executeInitializeOracle(State storage state, InitializeOracleParams memory o) internal {
	```

- Found in src/libraries/actions/Initialize.sol [Line: 230](src/libraries/actions/Initialize.sol#L230)

	```solidity
	    function executeInitializeData(State storage state, InitializeDataParams memory d) internal {
	```

</details>



## L-8: Unused Custom Error

it is recommended that the definition be removed when custom error is unused

<details><summary>2 Found Instances</summary>


- Found in src/libraries/Errors.sol [Line: 49](src/libraries/Errors.sol#L49)

	```solidity
	    error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);
	```

- Found in src/libraries/Errors.sol [Line: 74](src/libraries/Errors.sol#L74)

	```solidity
	    error NULL_STALE_RATE();
	```

</details>

## L-9: Unused named return

Declaring named returns, but not using them, is confusing to the reader. Consider either completely removing them (by declaring just the type without a name), or remove the return statement and do a variable assignment. This would improve the readability of the code, and it may also help reduce regressions during future code refactors.

<details><summary>5 Found Instances</summary>


- Found in src/Size.sol [Line: 147](src/Size.sol#L147)

	```solidity
	function multicall(bytes[] calldata _data)
        public
        payable
        override(IMulticall)
        whenNotPaused
        returns (bytes[] memory results)
    {
        results = state.multicall(_data);
    }
	```
- Found in src/Size.sol [Line: 215](src/Size.sol#L215)

	```solidity
    function liquidate(LiquidateParams calldata params)
        external
        payable
        override(ISize)
        whenNotPaused
        returns (uint256 liquidatorProfitCollateralToken)
    {
        state.validateLiquidate(params);
        liquidatorProfitCollateralToken = state.executeLiquidate(params);
        state.validateMinimumCollateralProfit(params, liquidatorProfitCollateralToken);
    }
	```

- Found in src/Size.sol [Line: 235](src/Size.sol#L235)

	```solidity
     function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)
        external
        payable
        override(ISize)
        whenNotPaused
        onlyRole(KEEPER_ROLE)
        returns (uint256 liquidatorProfitCollateralToken, uint256 liquidatorProfitBorrowToken)
    {
        state.validateLiquidateWithReplacement(params);
        uint256 amount;
        (amount, liquidatorProfitCollateralToken, liquidatorProfitBorrowToken) =
            state.executeLiquidateWithReplacement(params);
        state.validateUserIsNotBelowOpeningLimitBorrowCR(params.borrower);
        state.validateMinimumCollateralProfit(params, liquidatorProfitCollateralToken);
        state.validateVariablePoolHasEnoughLiquidity(amount);
    }
	```

- Found in src/AccountingLibrary.sol [Line: 29](src/AccountingLibrary.sol#L29)

	```solidity
    function debtTokenAmountToCollateralTokenAmount(State storage state, uint256 debtTokenAmount)
        internal
        view
        returns (uint256 collateralTokenAmount)
    {
        uint256 debtTokenAmountWad = Math.amountToWad(debtTokenAmount, state.data.underlyingBorrowToken.decimals());
        collateralTokenAmount = Math.mulDivUp(
            debtTokenAmountWad, 10 ** state.oracle.priceFeed.decimals(), state.oracle.priceFeed.getPrice()
        );
    }
	```

- Found in src/AccountingLibrary.sol [Line: 62](src/AccountingLibrary.sol#L62)

	```solidity
    function createDebtAndCreditPositions(
        State storage state,
        address lender,
        address borrower,
        uint256 futureValue,
        uint256 dueDate
    ) external returns (CreditPosition memory creditPosition) {
        DebtPosition memory debtPosition =
            DebtPosition({borrower: borrower, futureValue: futureValue, dueDate: dueDate, liquidityIndexAtRepayment: 0});

        uint256 debtPositionId = state.data.nextDebtPositionId++;
        state.data.debtPositions[debtPositionId] = debtPosition;

        emit Events.CreateDebtPosition(debtPositionId, borrower, lender, futureValue, dueDate);

        creditPosition = CreditPosition({
            lender: lender,
            credit: debtPosition.futureValue,
            debtPositionId: debtPositionId,
            forSale: true
        });

        uint256 creditPositionId = state.data.nextCreditPositionId++;
        state.data.creditPositions[creditPositionId] = creditPosition;
        state.validateMinimumCreditOpening(creditPosition.credit);
        state.validateTenor(dueDate - block.timestamp);

        emit Events.CreateCreditPosition(creditPositionId, lender, debtPositionId, RESERVED_ID, creditPosition.credit);

        state.data.debtToken.mint(borrower, futureValue);
    }
	```


</details>

## L-10: Missing NatSpec from contract declarations

Some contracts miss a @dev/@notice NatSpec, which should be a best practice to add as a documentation.

<details><summary>3 Found Instances</summary>


- Found in src/Size.sol [Line: 62](src/libraries/Errors.sol#L62)

	```solidity
	    /// @title Size
		/// @custom:security-contact security@size.credit
		/// @author Size (https://size.credit/)
		/// @notice See the documentation in {ISize}.
	```
- Found in src/SizeStorage.sol

- Found in src/SizeViewData.sol

</details>

## L-11: Lack of Event Emission for Critical Actions

Critical actions such as deposit, withdraw, buyCreditLimit, sellCreditLimit, buyCreditMarket, sellCreditMarket, repay, claim, liquidate, selfLiquidate, liquidateWithReplacement, compensate, and setUserConfiguration should emit events to allow for better tracking and auditing. Emitting events helps with off-chain logging and monitoring.

<details><summary>1 Found Instances</summary>

- Found in src/Size.sol

</details>

## L-12: Function visibility order does not comply with best practices

Description
This is a best practice* that should be followed.

Functions should be grouped according to their visibility and ordered in all contracts.

constructor
receive function (if exists)
fallback function (if exists)
external
public
internal
private
Within a grouping, place the view and pure functions last.

*best practice
https://docs.soliditylang.org/en/latest/style-guide.html#order-of-functions
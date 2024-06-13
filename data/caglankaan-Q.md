### Contracts are vulnerable to fee-on-transfer accounting-related issues
The function(s) below transfer funds from the caller to the receiver via `transferFrom()`, but do not ensure that the actual number of tokens received is the same as the input amount to the transfer. If the token is a fee-on-transfer token, the balance after the transfer will be smaller than expected, leading to accounting issues. Even if there are checks later, related to a secondary transfer, an attacker may be able to use latent funds (e.g. mistakenly sent by another user) in order to get a free credit. One way to address this problem is to measure the balance before and after the transfer, and use the difference as the amount, rather than the stated amount.

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

25:        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L25-L25), 


#### Recommendation

To mitigate potential vulnerabilities and ensure accurate accounting with fee-on-transfer tokens, modify your contract's token transfer logic to measure the recipient's balance before and after the transfer. Use this observed difference as the actual transferred amount for any further logic or calculations. For example:
```solidity
function transferTokenAndPerformAction(address token, address from, address to, uint256 amount) public {
    uint256 balanceBefore = IERC20(token).balanceOf(to);

    // Perform the token transfer
    IERC20(token).transferFrom(from, to, amount);

    uint256 balanceAfter = IERC20(token).balanceOf(to);
    uint256 actualReceived = balanceAfter - balanceBefore;

    // Proceed with logic using actualReceived instead of the initial amount
    require(actualReceived >= minimumRequiredAmount, "Received amount is less than required");

    // Further logic here
}
```
### Missing L2 sequencer checks for Chainlink oracle
Using Chainlink in L2 chains such as Arbitrum [requires](https://docs.chain.link/data-feeds/l2-sequencer-feeds) to check if the sequencer is down to avoid prices from looking like they are fresh although they are not.

The bug could be leveraged by malicious actors to take advantage of the sequencer downtime.

```solidity
Path: ./src/oracle/PriceFeed.sol

86:        (, int256 price,, uint256 updatedAt,) = aggregator.latestRoundData();	// @audit-issue
```
[86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L86-L86), 


#### Recommendation

Example how to fix:
```solidity
(
    /*uint80 roundID*/,
    int256 answer,
    uint256 startedAt,
    /*uint256 updatedAt*/,
    /*uint80 answeredInRound*/
) = sequencerUptimeFeed.latestRoundData();

// Answer == 0: Sequencer is up
// Answer == 1: Sequencer is down
bool isSequencerUp = answer == 0;
if (!isSequencerUp) {
    revert SequencerDown();
}
```
### Centralization risk for privileged functions
Contracts with privileged functions need owner/admin to be trusted not to perform malicious updates or drain funds. This may also cause a single point failure.


```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue

110:    function updateConfig(UpdateConfigParams calldata params)
111:        external
112:        override(ISizeAdmin)
113:        onlyRole(DEFAULT_ADMIN_ROLE)	// @audit-issue

120:    function setVariablePoolBorrowRate(uint128 borrowRate)
121:        external
122:        override(ISizeAdmin)
123:        onlyRole(BORROW_RATE_UPDATER_ROLE)	// @audit-issue

132:    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue

137:    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)
230:        external
231:        payable
232:        override(ISize)
233:        whenNotPaused
234:        onlyRole(KEEPER_ROLE)	// @audit-issue
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L113), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L123), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L132-L132), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L137-L137), [234](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L234), 


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

To mitigate centralization risks associated with privileged functions, consider implementing a multi-signature or decentralized governance mechanism. Instead of relying solely on a single owner/administrator, distribute control and decision-making authority among multiple parties or stakeholders. This approach enhances security, reduces the risk of malicious actions by a single entity, and prevents single points of failure. Explore governance solutions and smart contract frameworks that support decentralized control and decision-making to enhance the trustworthiness and resilience of your contract.

### Missing checks for `address(0)` in constructor/initializers
In Solidity, the Ethereum address `0x0000000000000000000000000000000000000000` is known as the "zero address". This address has significance because it's the default value for uninitialized address variables and is often used to represent an invalid or non-existent address. The "
Missing zero address control" issue arises when a Solidity smart contract does not properly check or prevent interactions with the zero address, leading to unintended behavior.
For instance, a contract might allow tokens to be sent to the zero address without any checks, which essentially burns those tokens as they become irretrievable. While sometimes this is intentional, without proper control or checks, accidental transfers could occur.    
        

```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

38:        underlyingToken = underlyingToken_;	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L38-L38), 


#### Recommendation

It is strongly recommended to implement checks to prevent the zero address from being set during the initialization of contracts. This can be achieved by adding require statements that ensure address parameters are not the zero address. 

### Missing checks for `address(0)`  when updating state variables
In Solidity, the Ethereum address `0x0000000000000000000000000000000000000000` is known as the "zero address". This address has significance because it's the default value for uninitialized address variables and is often used to represent an invalid or non-existent address. The "
Missing zero address control" issue arises when a Solidity smart contract does not properly check or prevent interactions with the zero address, leading to unintended behavior.
For instance, a contract might allow tokens to be sent to the zero address without any checks, which essentially burns those tokens as they become irretrievable. While sometimes this is intentional, without proper control or checks, accidental transfers could occur.    
        

```solidity
Path: ./src/libraries/actions/Initialize.sol

184:        validateInitializeFeeConfigParams(f);	// @audit-issue

185:        validateInitializeRiskConfigParams(r);	// @audit-issue

186:        validateInitializeOracleParams(o);	// @audit-issue

187:        validateInitializeDataParams(d);	// @audit-issue
```
[184](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L184-L184), [185](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L185-L185), [186](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L186-L186), [187](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L187-L187), 


#### Recommendation

It is strongly recommended to implement checks to prevent the zero address from being set during the initialization of contracts. This can be achieved by adding require statements that ensure address parameters are not the zero address. 

### The `constructor`/`initialize` function lacks parameter validation.
Constructors and initialization functions play a critical role in contracts by setting important initial states when the contract is first deployed before the system starts. The parameters passed to the constructor and initialization functions directly affect the behavior of the contract / protocol. If incorrect parameters are provided, the system may fail to run, behave abnormally, be unstable, or lack security. Therefore, it's crucial to carefully check each parameter in the constructor and initialization functions. If an exception is found, the transaction should be rolled back.

```solidity
Path: ./src/oracle/PriceFeed.sol

56:        quoteStalePriceInterval = _quoteStalePriceInterval;	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L56-L56), 


#### Recommendation

Incorporate comprehensive parameter validation in your contract's constructor and initialization functions. This should include checks for value ranges, address validity, and any other condition that defines a parameter as valid. Use `require` statements for validation, providing clear error messages for each condition. If any validation fails, the `require` statement will revert the transaction, preventing the contract from being deployed or initialized with invalid parameters.
Example:
```solidity
contract MyContract {
    address public owner;
    uint256 public someValue;

    constructor(address _owner, uint256 _someValue) {
        require(_owner != address(0), "Owner cannot be the zero address");
        require(_someValue > 0 && _someValue < 100, "SomeValue must be between 1 and 99");

        owner = _owner;
        someValue = _someValue;
    }
}

// For contracts using the proxy pattern and requiring initialization functions:
function initialize(address _owner, uint256 _someValue) public {
    require(_owner != address(0), "Owner cannot be the zero address");
    require(_someValue > 0 && _someValue < 100, "SomeValue must be between 1 and 99");

    owner = _owner;
    someValue = _someValue;
}
```


### Use `Ownable2Step` rather than `Ownable`
[Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3d7a93876a2e5e1d7fe29b5a0e96e222afdc4cfa/contracts/access/Ownable2Step.sol#L31-L56) and [Ownable2StepUpgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/25aabd286e002a1526c345c8db259d57bdf0ad28/contracts/access/Ownable2StepUpgradeable.sol#L47-L63) prevent the contract ownership from mistakenly being transferred to an address that cannot handle it (e.g. due to a typo in the address), by requiring that the recipient of the owner permissions actively accept via a contract call of its own.


```solidity
Path: ./src/token/NonTransferrableToken.sol

14:contract NonTransferrableToken is Ownable, ERC20 {	// @audit-issue
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L14-L14), 


#### Recommendation

Consider using `Ownable2Step` or `Ownable2StepUpgradeable` from OpenZeppelin Contracts to enhance the security of your contract ownership management. These contracts prevent the accidental transfer of ownership to an address that cannot handle it, such as due to a typo, by requiring the recipient of owner permissions to actively accept ownership via a contract call. This two-step ownership transfer process adds an additional layer of security to your contract's ownership management.

### `decimals()` is not a part of the `ERC-20` standard
The `decimals()` function is not a part of the [ERC-20](https://eips.ethereum.org/EIPS/eip-20) standard, and was added later as an [optional extension](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/IERC20Metadata.sol). As such, some valid `ERC20` tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.


```solidity
Path: ./src/libraries/actions/Initialize.sol

151:        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {	// @audit-issue

152:            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingCollateralToken).decimals());	// @audit-issue

159:        if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {	// @audit-issue

160:            revert Errors.INVALID_DECIMALS(IERC20Metadata(d.underlyingBorrowToken).decimals());	// @audit-issue

243:            IERC20Metadata(state.data.underlyingCollateralToken).decimals()	// @audit-issue

251:            IERC20Metadata(state.data.underlyingBorrowToken).decimals()	// @audit-issue

257:            IERC20Metadata(state.data.underlyingBorrowToken).decimals()	// @audit-issue
```
[151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L151-L151), [152](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L152-L152), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L159-L159), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L160-L160), [243](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L243-L243), [251](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L251-L251), [257](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L257-L257), 


#### Recommendation

When working with ERC-20 tokens in your Solidity code, be aware that the `decimals()` function is not a part of the ERC-20 standard and is considered an optional extension. Not all valid ERC-20 tokens implement this interface, so avoid blindly casting all tokens to this interface and calling the `decimals()` function. Instead, check the token's documentation or contract to determine whether it supports this extension before using it.

### `symbol()` is not a part of the `ERC-20` standard
The `symbol()` function is not a part of the [ERC-20](https://eips.ethereum.org/EIPS/eip-20) standard, and was added later as an [optional extension](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/IERC20Metadata.sol). As such, some valid `ERC20` tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.


```solidity
Path: ./src/libraries/actions/Initialize.sol

242:            string.concat("sz", IERC20Metadata(state.data.underlyingCollateralToken).symbol()),	// @audit-issue

250:            string.concat("sza", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),	// @audit-issue

256:            string.concat("szDebt", IERC20Metadata(state.data.underlyingBorrowToken).symbol()),	// @audit-issue
```
[242](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L242-L242), [250](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L250-L250), [256](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L256-L256), 


#### Recommendation

When working with ERC-20 tokens in your Solidity code, be aware that the `symbol()` function is not a part of the ERC-20 standard and is considered an optional extension. Not all valid ERC-20 tokens implement this interface, so avoid blindly casting all tokens to this interface and calling the `symbol()` function. Instead, check the token's documentation or contract to determine whether it supports this extension before using it.

### Int casting `block.timestamp` can reduce the lifespan of a contract
In Solidity, `block.timestamp` returns the Unix timestamp of the current block, which is a continuously increasing value. Casting `block.timestamp` to a smaller integer type, like `uint32`, can limit the maximum value it can hold. As time progresses and the Unix timestamp exceeds this maximum value, the casted value will overflow, leading to incorrect and potentially harmful behavior in the contract. This issue can significantly reduce the functional lifespan of a contract, as it becomes unreliable and potentially insecure once the timestamp exceeds the capacity of the casted integer type.

```solidity
Path: ./src/Size.sol

127:        state.oracle.variablePoolBorrowRateUpdatedAt = uint64(block.timestamp);	// @audit-issue
```
[127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L127-L127), 


#### Recommendation

Avoid casting `block.timestamp` to smaller integer types in your Solidity contracts. Use `uint256` to store timestamp values to accommodate the increasing nature of Unix timestamps and ensure the contract remains functional in the long term. Regularly review and update your contracts to remove any unnecessary casting of `block.timestamp` that could lead to overflows and reduce the contract's lifespan. By doing so, you maintain the contract's reliability and integrity well into the future.

### Functions calling contracts/addresses with transfer hooks are missing reentrancy guards
Even if the function follows the best practice of check-effects-interaction, not using a reentrancy guard when there may be transfer hooks will open the users of this protocol up to [read-only reentrancies](https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/) with no way to protect against it, except by block-listing the whole protocol.

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

25:        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);	// @audit-issue

39:        underlyingCollateralToken.safeTransfer(to, amount);	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L25-L25), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L39-L39), 


#### Recommendation

To ensure the security of your protocol and protect users from potential vulnerabilities, consider implementing reentrancy guards in functions that call contracts or addresses with transfer hooks. Even if your function follows the check-effects-interaction pattern, using reentrancy guards is essential to prevent read-only reentrancy attacks, as demonstrated in incidents like the [Curve LP Oracle Manipulation](https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/). Implementing reentrancy guards adds an extra layer of protection and safeguards your protocol against such attacks.

### Consider using OpenZeppelin’s `SafeCast` library to prevent unexpected overflows when casting from various type int/uint values
In Solidity, when casting from `int` to `uint` or vice versa, there is a risk of unexpected overflow or underflow, especially when dealing with large values. To mitigate this risk and ensure safe type conversions, you can consider using OpenZeppelin’s `SafeCast` library. This library provides functions that check for overflow/underflow conditions before performing the cast, helping you prevent potential issues related to type conversion in your smart contracts.

```solidity
Path: ./src/Size.sol

127:        state.oracle.variablePoolBorrowRateUpdatedAt = uint64(block.timestamp);	// @audit-issue
```
[127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L127-L127), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

134:            state.feeConfig.feeRecipient = address(uint160(params.value));	// @audit-issue

136:            state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));	// @audit-issue

138:            state.oracle.variablePoolBorrowRateStaleRateInterval = uint64(params.value);	// @audit-issue
```
[134](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L134-L134), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L136-L136), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L138-L138), 


#### Recommendation

To enhance the safety and reliability of your Solidity smart contracts, it is advisable to utilize OpenZeppelin’s `SafeCast` library when casting between `int` and `uint` types. Incorporating this library into your codebase will help prevent unexpected overflows and underflows during type conversion, reducing the risk of vulnerabilities and ensuring secure contract execution.

### Numbers downcast to `addresses` may result in collisions
If a number is downcast to an `address` the upper bytes are truncated, which may mean that more than one value will map to the `address`


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

134:            state.feeConfig.feeRecipient = address(uint160(params.value));	// @audit-issue

136:            state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));	// @audit-issue
```
[134](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L134-L134), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L136-L136), 


#### Recommendation

When downcasting numbers to `addresses` in Solidity, be cautious of potential collisions. Downcasting truncates the upper bytes of the number, which can lead to multiple values mapping to the same `address`. To avoid collisions, consider using a more robust mapping strategy or additional checks to ensure unique mappings.

### Initializers can be front-run
Initializers could be front-run, allowing an attacker to either set their own values, take ownership of the contract, and in the best case forcing a re-deployment.


```solidity
Path: ./src/Size.sol

87:    function initialize(	// @audit-issue
```
[87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L87), 


#### Recommendation

To mitigate the risk of initializers being front-run, ensure that they are only callable by a trusted entity, such as the deployer, or through a secure, multi-step initialization process. One approach is to use a constructor for initial setup, which is inherently protected against front-running. If a separate initializer is necessary, consider using a modifier that restricts the initializer function to be called only once by a predefined address, typically the deployer's address. Additionally, implement robust checks within the initializer to validate inputs and reject suspicious or unauthorized initialization attempts.

### Revert on transfer to the zero address
It's good practice to revert a token transfer transaction if the recipient's address is the zero address. This can prevent unintentional transfers to the zero address due to accidental operations or programming errors. Many token contracts implement such a safeguard, such as [OpenZeppelin - ERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9e3f4d60c581010c4a3979480e07cc7752f124cc/contracts/token/ERC20/ERC20.sol#L232), [OpenZeppelin - ERC721](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9e3f4d60c581010c4a3979480e07cc7752f124cc/contracts/token/ERC721/ERC721.sol#L142).

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

39:        underlyingCollateralToken.safeTransfer(to, amount);	// @audit-issue
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L39-L39), 


#### Recommendation

To enhance the security and reliability of your token contracts, it's advisable to implement a safeguard that reverts token transfer transactions if the recipient's address is the zero address. This practice helps prevent unintentional transfers to the zero address, reducing the risk of fund loss due to accidental operations or programming errors. Many token contracts, including [OpenZeppelin's ERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9e3f4d60c581010c4a3979480e07cc7752f124cc/contracts/token/ERC20/ERC20.sol#L232) and [ERC721](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9e3f4d60c581010c4a3979480e07cc7752f124cc/contracts/token/ERC721/ERC721.sol#L142), incorporate this safeguard for added security.

### State variables not limited to reasonable values
Consider adding appropriate minimum/maximum value checks to ensure that the following state variables can never be used to excessively harm users, including via griefing.

```solidity
Path: ./src/Size.sol

126:        state.oracle.variablePoolBorrowRate = borrowRate;	// @audit-issue
```
[126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L126-L126), 


#### Recommendation


Implement validation checks for state variables to enforce minimum and maximum value limits. This can be achieved by adding modifiers or require statements in Solidity functions that modify these state variables. Ensure that these limits are reasonable and reflect the intended use of the contract. Additionally, consider implementing a mechanism to update these limits through a governance process or a trusted role, if applicable, to maintain flexibility and adaptability of the contract over time.

### External calls in an unbounded loop can result in a DoS
Consider limiting the number of iterations in loops that make external calls, as just a single one of them failing will result in a revert.

```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

65:                revert Errors.TENORS_NOT_STRICTLY_INCREASING();	// @audit-issue
```
[65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L65-L65), 


```solidity
Path: ./src/libraries/Multicall.sol

34:            results[i] = Address.functionDelegateCall(address(this), data[i]);	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L34-L34), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

49:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionIds[i]);	// @audit-issue

51:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionIds[i]);	// @audit-issue

54:            if (state.getLoanStatus(params.creditPositionIds[i]) != LoanStatus.ACTIVE) {	// @audit-issue

55:                revert Errors.LOAN_NOT_ACTIVE(params.creditPositionIds[i]);	// @audit-issue

70:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionIds[i]);	// @audit-issue

72:            emit Events.UpdateCreditPosition(	// @audit-issue

73:                params.creditPositionIds[i], creditPosition.lender, creditPosition.credit, creditPosition.forSale	// @audit-issue
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L49-L49), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L51-L51), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L54-L54), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L55-L55), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L70-L70), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L73-L73), 


#### Recommendation

To mitigate the risk of Denial-of-Service (DoS) attacks in your Solidity code, it's important to limit the number of iterations in loops that involve external calls. A single failed external call in an unbounded loop can lead to a revert, causing disruptions in contract execution. Consider implementing safeguards, such as setting a maximum loop iteration count or employing strategies like batch processing, to reduce the impact of potential external call failures.

### Upgradeable contract is missing `__gap[..]` storage variable
Storage gaps are needed to not break storage layouts when adding new variables to base contracts. Listed contracts may not be inherited from but it is good practice to add it now rather than forgetting later.


```solidity
Path: ./src/Size.sol

62:contract Size is ISize, SizeView, Initializable, AccessControlUpgradeable, PausableUpgradeable, UUPSUpgradeable {	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L62-L62), 


#### Recommendation

When designing upgradeable contracts, it's important to include `__gap[..]` storage variables as storage gaps to prevent issues with storage layout changes when adding new variables to base contracts in the future. While the listed contracts may not be directly inherited from, it is a good practice to add `__gap[..]` storage variables now to ensure a robust and future-proof upgradeable contract design. This proactive approach helps avoid potential complications and storage layout conflicts in later stages of contract development.

### Contracts are designed to receive ETH but do not implement function for withdrawal
The following contracts can receive ETH but do not provide a function for withdrawal. This means that any ETH sent to these contracts will be permanently stuck, unable to be retrieved by the contract owner or any other party. Additionally, this issue can also apply to baseTokens, resulting in locked tokens and potential loss of funds.


```solidity
Path: ./src/Size.sol

142:    function multicall(bytes[] calldata _data)	// @audit-issue
```
[142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L142), 


#### Recommendation

To prevent ETH and token lock-up and potential loss of funds, ensure that contracts designed to receive ETH or tokens implement a function for withdrawal. This function should allow contract owners and users to retrieve their funds when needed. Failure to provide a withdrawal mechanism can lead to locked assets and permanent loss, posing a significant risk to contract users and owners.

### Critical functions should have a timelock
Critical functions, especially those affecting protocol parameters or user funds, are potential points of failure or exploitation. To mitigate risks, incorporating a timelock on such functions can be beneficial. A timelock requires a waiting period between the time an action is initiated and when it's executed, giving stakeholders time to react, potentially vetoing malicious or erroneous changes. To implement, integrate a smart contract like OpenZeppelin's `TimelockController` or build a custom mechanism. This ensures governance decisions or administrative changes are transparent and allows for community or multi-signature interventions, enhancing protocol security and trustworthiness.

```solidity
Path: ./src/Size.sol

120:    function setVariablePoolBorrowRate(uint128 borrowRate)
121:        external
122:        override(ISizeAdmin)
123:        onlyRole(BORROW_RATE_UPDATER_ROLE)
124:    {
125:        uint128 oldBorrowRate = state.oracle.variablePoolBorrowRate;
126:        state.oracle.variablePoolBorrowRate = borrowRate;	// @audit-issue
```
[126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L126), 


#### Recommendation

Integrate a timelock mechanism into your Solidity contracts for critical functions, especially those controlling protocol parameters or managing user funds. Consider using established solutions like OpenZeppelin's `TimelockController` for robustness and reliability. Alternatively, develop a custom timelock mechanism tailored to your specific requirements. Ensure that the timelock duration is appropriate for your contract's use case and stakeholder needs, providing sufficient time for review and intervention. Clearly document and communicate the presence and workings of the timelock mechanism to stakeholders to maintain transparency and trust in your contract's operations.

### Consider bounding input array length
If the number of for loop iterations is unbounded, then it may lead to the transaction to run out of gas. While the function will revert if it eventually runs out of gas, it may be a nicer user experience to require() that the length of the array is below some reasonable maximum, so that the user doesn't have to use up a full transaction's gas only to see that the transaction reverts.

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

Implement a check at the beginning of your Solidity functions to enforce a maximum length on input arrays. Use a `require()` statement to validate that the length of any input array does not exceed a predetermined limit, which should be chosen based on the function's complexity and typical gas usage. This ensures that the function will not attempt to process more data than it can handle within reasonable gas limits, thereby preventing out-of-gas errors and improving the overall user experience. Clearly document this behavior and the rationale behind the chosen array size limit to inform users and developers interacting with your contract.

### Large transfers may not work with some `ERC20` tokens
Not all IERC20 implementations are totally compliant, and some (e.g UNI, COMP) may fail if the valued passed is larger than uint96. [Source](https://github.com/d-xo/weird-erc20#revert-on-large-approvals--transfers)


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

25:        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);	// @audit-issue

39:        underlyingCollateralToken.safeTransfer(to, amount);	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L25-L25), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L39-L39), 


#### Recommendation

When transferring tokens using ERC-20 contracts, such as UNI or COMP, it's important to be aware that not all IERC20 implementations are fully compliant, and they may not support large transfer values exceeding uint96. To avoid potential issues, it's essential to check the documentation or source code of the specific token you're using and ensure that your transfers adhere to the token's limitations.

### Possible loss of precision
Division by large numbers may result in precision loss due to rounding down, or even the result being erroneously equal to zero. Consider adding checks on the numerator to ensure precision loss is handled appropriately.


```solidity
Path: ./src/libraries/Math.sol

28:        return FixedPointMathLib.mulDiv(x, y, z);	// @audit-issue
```
[28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L28-L28), 


#### Recommendation

Incorporate strategies in your Solidity contracts to mitigate precision loss in division operations. This can include:
1. Performing checks on the numerator and denominator to ensure they are within a reasonable range to avoid significant rounding errors.
2. Considering the use of fixed-point arithmetic libraries or scaling factors to handle divisions with higher precision.
3. Clearly documenting any inherent limitations of your division logic and providing guidelines for inputs to minimize unexpected behavior.
Always thoroughly test division operations under various scenarios to ensure that the outcomes are consistent with your contract's intended logic and accuracy requirements.

### Library function isn't `internal` or `private`
In a library, using an external or public visibility means that we won't be going through the library with a DELEGATECALL but with a CALL. This changes the context and should be done carefully.

```solidity
Path: ./src/libraries/RiskLibrary.sol

21:    function validateMinimumCredit(State storage state, uint256 credit) public view {	// @audit-issue

31:    function validateMinimumCreditOpening(State storage state, uint256 credit) public view {	// @audit-issue

41:    function validateTenor(State storage state, uint256 tenor) public view {	// @audit-issue

53:    function collateralRatio(State storage state, address account) public view returns (uint256) {	// @audit-issue

71:    function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)	// @audit-issue

104:    function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {	// @audit-issue

121:    function isUserUnderwater(State storage state, address account) public view returns (bool) {	// @audit-issue

129:    function validateUserIsNotUnderwater(State storage state, address account) external view {	// @audit-issue

140:    function validateUserIsNotBelowOpeningLimitBorrowCR(State storage state, address account) external view {	// @audit-issue
```
[21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L21-L21), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L31-L31), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L41-L41), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L53-L53), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L71-L71), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L104-L104), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L121-L121), [129](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L129-L129), [140](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L140-L140), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

80:    function getDebtPosition(State storage state, uint256 debtPositionId) public view returns (DebtPosition storage) {	// @audit-issue

93:    function getCreditPosition(State storage state, uint256 creditPositionId)	// @audit-issue

109:    function getDebtPositionByCreditPositionId(State storage state, uint256 creditPositionId)	// @audit-issue

122:    function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {	// @audit-issue

148:    function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)	// @audit-issue

170:    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)	// @audit-issue
```
[80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L80-L80), [93](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L93-L93), [109](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L109-L109), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L122-L122), [148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L148-L148), [170](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L170-L170), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

115:    function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue
```
[115](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L115-L115), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

42:    function repayDebt(State storage state, uint256 debtPositionId, uint256 repayAmount) public {	// @audit-issue

62:    function createDebtAndCreditPositions(	// @audit-issue

103:    function createCreditPosition(State storage state, uint256 exitCreditPositionId, address lender, uint256 credit)	// @audit-issue

137:    function reduceCredit(State storage state, uint256 creditPositionId, uint256 amount) public {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L42-L42), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L62-L62), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L103-L103), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L137-L137), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

19:    function validateBorrowATokenIncreaseLteDebtTokenDecrease(	// @audit-issue

52:    function validateBorrowATokenCap(State storage state) external view {	// @audit-issue

67:    function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {	// @audit-issue
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L19-L19), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L52-L52), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L67-L67), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

23:    function depositUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount) external {	// @audit-issue

34:    function withdrawUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount)	// @audit-issue

49:    function depositUnderlyingBorrowTokenToVariablePool(State storage state, address from, address to, uint256 amount)	// @audit-issue

74:    function withdrawUnderlyingTokenFromVariablePool(State storage state, address from, address to, uint256 amount)	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L23-L23), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L34-L34), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L49-L49), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L74-L74), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

31:    function validateSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params)	// @audit-issue

63:    function executeSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params) external {	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L31-L31), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L63-L63), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

34:    function validateSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external view {	// @audit-issue

59:    function executeSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external {	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L34-L34), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L59-L59), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

42:    function validateCompensate(State storage state, CompensateParams calldata params) external view {	// @audit-issue

106:    function executeCompensate(State storage state, CompensateParams calldata params) external {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L42-L42), [106](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L106-L106), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

51:    function validateSellCreditMarket(State storage state, SellCreditMarketParams calldata params) external view {	// @audit-issue

127:    function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L51-L51), [127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L127-L127), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

42:    function feeConfigParams(State storage state) public view returns (InitializeFeeConfigParams memory) {	// @audit-issue

56:    function riskConfigParams(State storage state) public view returns (InitializeRiskConfigParams memory) {	// @audit-issue

70:    function oracleParams(State storage state) public view returns (InitializeOracleParams memory) {	// @audit-issue

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue

86:    function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L56-L56), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L70-L70), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L79), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L86-L86), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

37:    function validateLiquidate(State storage state, LiquidateParams calldata params) external view {	// @audit-issue

59:    function validateMinimumCollateralProfit(	// @audit-issue

75:    function executeLiquidate(State storage state, LiquidateParams calldata params)	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L37-L37), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L59-L59), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L75-L75), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

47:    function validateLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)	// @audit-issue

99:    function validateMinimumCollateralProfit(	// @audit-issue

120:    function executeLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)	// @audit-issue
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L47-L47), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L99-L99), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L120-L120), 


```solidity
Path: ./src/libraries/actions/Claim.sol

31:    function validateClaim(State storage state, ClaimParams calldata params) external view {	// @audit-issue

48:    function executeClaim(State storage state, ClaimParams calldata params) external {	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L31-L31), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L48-L48), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

175:    function validateInitialize(	// @audit-issue

267:    function executeInitialize(	// @audit-issue
```
[175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L175), [267](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L267), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

29:    function validateWithdraw(State storage state, WithdrawParams calldata params) external view {	// @audit-issue

52:    function executeWithdraw(State storage state, WithdrawParams calldata params) public {	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L29-L29), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

25:    function validateSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external view {	// @audit-issue

44:    function executeSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external {	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L25-L25), [44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L44-L44), 


```solidity
Path: ./src/libraries/actions/Repay.sol

33:    function validateRepay(State storage state, RepayParams calldata params) external view {	// @audit-issue

46:    function executeRepay(State storage state, RepayParams calldata params) external {	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L33-L33), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L46-L46), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

29:    function validateBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external view {	// @audit-issue

57:    function executeBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external {	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L29-L29), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L57-L57), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

51:    function validateBuyCreditMarket(State storage state, BuyCreditMarketParams calldata params) external view {	// @audit-issue

121:    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L51-L51), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L121-L121), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

36:    function validateDeposit(State storage state, DepositParams calldata params) external view {	// @audit-issue

64:    function executeDeposit(State storage state, DepositParams calldata params) public {	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L36-L36), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L64-L64), 


#### Recommendation

Carefully assess the intended use of functions within your libraries and opt for `internal` or `private` visibility unless there's a specific reason to expose them via `external` or `public`. This practice encourages the use of `DELEGATECALL` where appropriate, maintaining the execution context of the calling contract and ensuring consistent access to storage variables.
### Non-compliant `IERC20` tokens may revert with `transfer`
Some `IERC20` tokens (e.g. `BNB`, `OMG`, `USDT`) do not implement the standard properly, but they are still accepted by most code that accepts `ERC20` tokens.

For example, `USDT` transfer functions on L1 do not return booleans: when casted to `IERC20`, their function signatures do not match, and therefore the calls made will revert.

```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

25:        underlyingCollateralToken.safeTransferFrom(from, address(this), amount);	// @audit-issue

39:        underlyingCollateralToken.safeTransfer(to, amount);	// @audit-issue
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L25-L25), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L39-L39), 


#### Recommendation

When dealing with tokens that may not fully comply with the `IERC20` standard, it's important to exercise caution and carefully verify the behavior of the specific token in question. In cases where tokens do not return booleans for transfer functions, consider implementing additional error-handling mechanisms to account for potential failures. This may include checking the token balance before and after the transfer to detect any discrepancies or using try-catch blocks to handle potential exceptions. Ensuring robust error handling can help your smart contract interact more gracefully with non-compliant tokens while maintaining security and reliability.

### Ownership Irrevocability Vulnerability
The smart contract under inspection inherits from the `Ownable` library, which provides basic authorization control functions, simplifying the implementation of user permissions. However, the contract does not provide a mechanism to transfer ownership to another address or account, and it retains the default `renounceOwnership` function from `Ownable`.  
Given this, once the owner renounces ownership using the `renounceOwnership` function, the contract becomes ownerless. As evidenced in the provided transaction logs, after the `renounceOwnership` function is called, attempts to call functions that require owner permissions fail with the error message: `"Ownable: caller is not the owner."`  
This state renders the contract's adjustable parameters immutable and potentially makes the contract useless for any future administrative changes that might be necessary.

```solidity
Path: ./src/token/NonTransferrableToken.sol

14:contract NonTransferrableToken is Ownable, ERC20 {	// @audit-issue
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L14-L14), 


#### Recommendation


To mitigate this vulnerability:
* Override the renounceOwnership function to revert transactions: By overriding this function to simply revert any transaction, it will become impossible for the contract owner to unintentionally (or intentionally) render the contract ownerless and thus immutable.
* Implement an ownership transfer function: While the Ownable library does provide a transferOwnership function, if this is not present or has been removed from the current contract, it should be re-implemented to ensure there is a way to transfer ownership in future scenarios.


### Unneeded initializations of integer variable to `0`.
In Solidity, it is common practice to initialize variables with default values when declaring them. However, initializing integer variables to `0` when they are not subsequently used in the code can lead to unnecessary gas consumption and code clutter. This issue points out instances where such initializations are present but serve no functional purpose.

```solidity
Path: ./src/libraries/AccountingLibrary.sol

238:        uint256 maxCashAmountOutFragmentation = 0;	// @audit-issue
```
[238](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L238-L238), 


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


```solidity
Path: ./src/libraries/actions/Liquidate.sol

92:        uint256 protocolProfitCollateralToken = 0;	// @audit-issue
```
[92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L92-L92), 


#### Recommendation


It is recommended not to initialize integer variables to `0` to save some Gas.


### Unused arguments should be removed or implemented
In Solidity, functions often have arguments that are intended for specific operations or logic within the function. However, sometimes these arguments remain unused in the function body, either due to changes during development or oversight. Unused arguments in functions can lead to confusion, making the code less readable and potentially misleading for other developers who might use or audit the contract. Moreover, they can create a false impression of the function's purpose and behavior. It's crucial to either implement these arguments in the function's logic as originally intended or remove them to ensure clarity and efficiency of the code.

```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue: newImplementation
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue: None
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L79), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

60:        State storage,	// @audit-issue: None
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L60-L60), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

176:        State storage,	// @audit-issue: None
```
[176](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L176-L176), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue: None

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue: None
```
[46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue: None

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue: None
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), 


#### Recommendation

Eliminate unused arguments in Solidity function definitions to enhance code clarity and efficiency. If an argument is currently not utilized in the function's logic, assess its potential future utility. If it serves no purpose, removing it simplifies the function signature and aligns the code more closely with its actual operation, contributing to a cleaner and more understandable contract structure.

### `else`-block not required
One level of nesting can be removed by not having an `else` block when the `if`-block returns, and `if (foo) { return 1; } else { return 2; }` becomes `if (foo) { return 1; } return 2;`


```solidity
Path: ./src/libraries/RiskLibrary.sol

59:        if (debt != 0) {
60:            return Math.mulDivDown(collateral, price, debtWad);
61:        } else {	// @audit-issue
62:            return type(uint256).max;
63:        }
```
[61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L59-L63), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

81:        if (isDebtPositionId(state, debtPositionId)) {
82:            return state.data.debtPositions[debtPositionId];
83:        } else {	// @audit-issue
84:            revert Errors.INVALID_DEBT_POSITION_ID(debtPositionId);
85:        }

98:        if (isCreditPositionId(state, creditPositionId)) {
99:            return state.data.creditPositions[creditPositionId];
100:        } else {	// @audit-issue
101:            revert Errors.INVALID_CREDIT_POSITION_ID(creditPositionId);
102:        }

135:        } else if (block.timestamp > debtPosition.dueDate) {
136:            return LoanStatus.OVERDUE;
137:        } else {	// @audit-issue
138:            return LoanStatus.ACTIVE;
139:        }

156:        if (debt != 0) {
157:            return Math.mulDivDown(collateral, debtPosition.futureValue, debt);
158:        } else {	// @audit-issue
159:            return 0;
160:        }

181:        if (debtPositionFutureValue != 0) {
182:            return Math.mulDivDown(debtPositionCollateral, creditPositionCredit, debtPositionFutureValue);
183:        } else {	// @audit-issue
184:            return 0;
185:        }
```
[83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L81-L85), [100](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L98-L102), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L135-L139), [158](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L156-L160), [183](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L181-L185), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

94:        } else if (
95:            params.variablePoolBorrowRateStaleRateInterval == 0
96:                || (
97:                    block.timestamp - params.variablePoolBorrowRateUpdatedAt
98:                        > params.variablePoolBorrowRateStaleRateInterval
99:                )
100:        ) {
101:            revert Errors.STALE_RATE(params.variablePoolBorrowRateUpdatedAt);
102:        } else {	// @audit-issue
103:            return SafeCast.toUint256(
104:                apr + SafeCast.toInt256(Math.mulDivDown(params.variablePoolBorrowRate, marketRateMultiplier, PERCENT))
105:            );
106:        }

121:        if (tenor < curveRelativeTime.tenors[0] || tenor > curveRelativeTime.tenors[length - 1]) {
122:            revert Errors.TENOR_OUT_OF_RANGE(tenor, curveRelativeTime.tenors[0], curveRelativeTime.tenors[length - 1]);
123:        } else {	// @audit-issue
124:            (uint256 low, uint256 high) = Math.binarySearch(curveRelativeTime.tenors, tenor);
125:            uint256 y0 =
126:                getAdjustedAPR(curveRelativeTime.aprs[low], curveRelativeTime.marketRateMultipliers[low], params);
127:
128:            if (low != high) {
129:                uint256 x0 = curveRelativeTime.tenors[low];
130:                uint256 x1 = curveRelativeTime.tenors[high];
131:                uint256 y1 =
132:                    getAdjustedAPR(curveRelativeTime.aprs[high], curveRelativeTime.marketRateMultipliers[high], params);
133:
134:                if (y1 >= y0) {
135:                    return y0 + Math.mulDivDown(y1 - y0, tenor - x0, x1 - x0);
136:                } else {
137:                    return y0 - Math.mulDivDown(y0 - y1, tenor - x0, x1 - x0);
138:                }
139:            } else {
140:                return y0;
141:            }
142:        }

134:                if (y1 >= y0) {
135:                    return y0 + Math.mulDivDown(y1 - y0, tenor - x0, x1 - x0);
136:                } else {	// @audit-issue
137:                    return y0 - Math.mulDivDown(y0 - y1, tenor - x0, x1 - x0);
138:                }
```
[102](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L94-L106), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L121-L142), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L134-L138), 


#### Recommendation

Consider simplifying code by removing unnecessary `else` blocks when the `if` block returns. You can achieve cleaner and more concise code by directly returning a value after the `if` block instead of using an `else` block for a subsequent return statement.

### Unused import
The identifier is imported but never used within the file.

```solidity
Path: ./src/Size.sol

42:import {State} from "@src/SizeStorage.sol";	// @audit-issue: State not used in the contract.
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L42-L42), 


```solidity
Path: ./src/SizeView.sol

4:import {SizeStorage, State, User} from "@src/SizeStorage.sol";	// @audit-issue: State not used in the contract.

4:import {SizeStorage, State, User} from "@src/SizeStorage.sol";	// @audit-issue: User not used in the contract.

7:import {	// @audit-issue: RESERVED_ID not used in the contract.
8:    CREDIT_POSITION_ID_START,
9:    CreditPosition,
10:    DEBT_POSITION_ID_START,
11:    DebtPosition,
12:    LoanLibrary,
13:    LoanStatus,
14:    RESERVED_ID
15:} from "@src/libraries/LoanLibrary.sol";

26:import {	// @audit-issue: InitializeDataParams not used in the contract.
27:    InitializeDataParams,
28:    InitializeFeeConfigParams,
29:    InitializeOracleParams,
30:    InitializeRiskConfigParams
31:} from "@src/libraries/actions/Initialize.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L4-L4), [4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L4-L4), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L7-L15), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L26-L31), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

14:import {	// @audit-issue: InitializeDataParams not used in the contract.
15:    InitializeDataParams,
16:    InitializeFeeConfigParams,
17:    InitializeOracleParams,
18:    InitializeRiskConfigParams
19:} from "@src/libraries/actions/Initialize.sol";
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L14-L19), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

8:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue: CreditPosition not used in the contract.
```
[8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L8-L8), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

8:import {Math} from "@src/libraries/Math.sol";	// @audit-issue: Math not used in the contract.
```
[8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L8-L8), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

4:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue: IERC20Metadata not used in the contract.

6:import {IWETH} from "@src/interfaces/IWETH.sol";	// @audit-issue: IWETH not used in the contract.
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L6-L6), 


#### Recommendation

Regularly review your Solidity code to remove unused imports. This practice declutters the codebase, making it easier to understand and maintain. In addition, consider using tools or IDE features that can automatically detect and highlight unused imports for cleanup. Keeping imports limited to only what is necessary helps maintain a clear understanding of the contract's dependencies and reduces potential confusion for developers working on or reviewing the code.

### Excessive Authorization in Contract Functions
A contract where a high percentage of functions require authorization (e.g., restricted to the contract owner or specific roles) may indicate over-centralization or excessive control. This could limit the contract's flexibility, reduce trust among users, and potentially create bottleneck points that could be exploited or become failure points. While some level of control is necessary for administrative purposes, overly restrictive access can detract from the decentralized nature of blockchain applications and concentrate too much power in the hands of a few.

```solidity
Path: ./src/token/NonTransferrableToken.sol

14:contract NonTransferrableToken is Ownable, ERC20 {	// @audit-issue: %80.0 amount of external/public and non-view/non-pure functions are required authorization to call.
	List of total functions: `burn`, `transferFrom`, `mint`, `transfer`, `approve`
	List of functions that require authorization: `transfer`, `mint`, `transferFrom`, `burn`
	List of functions that doesn't require authorization: `approve`
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L14-L14), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

19:contract NonTransferrableScaledToken is NonTransferrableToken {	// @audit-issue: %100.0 amount of external/public and non-view/non-pure functions are required authorization to call.
	List of total functions: `burnScaled`, `mintScaled`, `transferFrom`
	List of functions that require authorization: `burnScaled`, `mintScaled`, `transferFrom`
	List of functions that doesn't require authorization: None
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L19-L19), 


#### Recommendation

Make contract more decentralized.

### `public` functions not called by the contract should be declared `external` instead
In Solidity, function visibility is an important aspect that determines how and where a function can be called from. Two commonly used visibilities are `public` and `external`. A `public` function can be called both from other functions inside the same contract and from outside transactions, while an `external` function can only be called from outside the contract.
A potential pitfall in smart contract development is the misuse of the `public` keyword for functions that are only meant to be accessed externally. When a function is not used internally within a contract and is only intended for external calls, it should be labeled as `external` rather than `public`. Using `public` unnecessarily can introduce potential vulnerabilities and also make the contract consume more gas than required. This is because `public` functions have to add additional code to handle both internal and external calls, while `external` functions can be more optimized since they only handle external calls.


```solidity
Path: ./src/SizeView.sol

179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {	// @audit-issue
```
[179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L179-L179), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

21:    function validateMinimumCredit(State storage state, uint256 credit) public view {	// @audit-issue

31:    function validateMinimumCreditOpening(State storage state, uint256 credit) public view {	// @audit-issue

41:    function validateTenor(State storage state, uint256 tenor) public view {	// @audit-issue

71:    function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)	// @audit-issue

104:    function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {	// @audit-issue
```
[21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L21-L21), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L31-L31), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L41-L41), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L71-L71), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L104-L104), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

122:    function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {	// @audit-issue

170:    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)	// @audit-issue
```
[122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L122-L122), [170](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L170-L170), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

67:    function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {	// @audit-issue
```
[67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L67-L67), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

52:    function executeWithdraw(State storage state, WithdrawParams calldata params) public {	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

64:    function executeDeposit(State storage state, DepositParams calldata params) public {	// @audit-issue
```
[64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L64-L64), 


#### Recommendation

To optimize gas usage and improve code clarity, declare functions that are not called internally within the contract and are intended for external access as `external` rather than `public`. This ensures that these functions are only callable externally, reducing unnecessary gas consumption and potential security risks.

### `Internal` Functions Not Called by The Contract Should Be Removed
In Solidity, functions labeled as `internal` are meant to be used only within the contract in which they're defined or in derived contracts. They cannot be accessed from external transactions. While `internal` functions offer modularity and can be leveraged to break down complex logic, it's essential to monitor their relevance during the contract's lifecycle.

A common oversight during smart contract development is the presence of `internal` functions that remain within the contract code but are never called by any other function or part of the contract. These redundant functions occupy space, make the contract's bytecode longer, and, as a consequence, increase the contract deployment cost. Moreover, they can add unnecessary complexity to the code, making it harder to read, maintain, and audit.


```solidity
Path: ./src/libraries/RiskLibrary.sol

89:    function isCreditPositionTransferrable(State storage state, uint256 creditPositionId)	// @audit-issue
```
[89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L89-L89), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

32:    function isNull(LoanOffer memory self) internal pure returns (bool) {	// @audit-issue

39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {	// @audit-issue

62:    function getRatePerTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L32-L32), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L39-L39), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L62-L62), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L90), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

38:    function isNull(YieldCurve memory self) internal pure returns (bool) {	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L38-L38), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

26:    function debtTokenAmountToCollateralTokenAmount(State storage state, uint256 debtTokenAmount)	// @audit-issue

153:    function reduceDebtAndCredit(State storage state, uint256 debtPositionId, uint256 creditPositionId, uint256 amount)	// @audit-issue

185:    function getCashAmountOut(	// @audit-issue

228:    function getCreditAmountIn(	// @audit-issue

274:    function getCreditAmountOut(	// @audit-issue

311:    function getCashAmountIn(	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L26-L26), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L153-L153), [185](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L185-L185), [228](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L228-L228), [274](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L274-L274), [311](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L311-L311), 


```solidity
Path: ./src/libraries/Multicall.sol

26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L26-L26), 


#### Recommendation

To maintain clean and efficient code, review the contract's `internal` functions to identify and remove any that are no longer in use or serve no purpose. Removing unused `internal` functions can improve code readability, reduce deployment costs, and enhance the overall quality of the smart contract.

### `if`-statement can be converted to a ternary
The code can be made more compact while also increasing readability by converting the following `if`-statements to ternaries (e.g. `foo += (x > y) ? a : b`)

```solidity
Path: ./src/Size.sol

181:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue
182:            state.validateUserIsNotBelowOpeningLimitBorrowCR(params.borrower);
183:        }

191:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue
192:            state.validateUserIsNotBelowOpeningLimitBorrowCR(msg.sender);
193:        }
```
[181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L181-L183), [191](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L191-L193), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

59:        if (debt != 0) {	// @audit-issue
60:            return Math.mulDivDown(collateral, price, debtWad);
61:        } else {
62:            return type(uint256).max;
63:        }
```
[59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L59-L63), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

135:        } else if (block.timestamp > debtPosition.dueDate) {	// @audit-issue
136:            return LoanStatus.OVERDUE;
137:        } else {
138:            return LoanStatus.ACTIVE;
139:        }

156:        if (debt != 0) {	// @audit-issue
157:            return Math.mulDivDown(collateral, debtPosition.futureValue, debt);
158:        } else {
159:            return 0;
160:        }

181:        if (debtPositionFutureValue != 0) {	// @audit-issue
182:            return Math.mulDivDown(debtPositionCollateral, creditPositionCredit, debtPositionFutureValue);
183:        } else {
184:            return 0;
185:        }
```
[135](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L135-L139), [156](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L156-L160), [181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L181-L185), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

240:        if (maxCashAmountOut >= state.feeConfig.fragmentationFee) {	// @audit-issue
241:            maxCashAmountOutFragmentation = maxCashAmountOut - state.feeConfig.fragmentationFee;
242:        }
```
[240](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L240-L242), 


```solidity
Path: ./src/libraries/Math.sol

54:        if (value < array[low] || value > array[high]) {	// @audit-issue
55:            return (type(uint256).max, type(uint256).max);
56:        }

61:            } else if (array[mid] < value) {	// @audit-issue
62:                low = mid + 1;
63:            } else {
64:                high = mid - 1;
65:            }
```
[54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L54-L56), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L61-L65), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

184:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue
185:            // slither-disable-next-line unused-return
186:            state.createDebtAndCreditPositions({
187:                lender: msg.sender,
188:                borrower: msg.sender,
189:                futureValue: creditAmountIn,
190:                dueDate: block.timestamp + tenor
191:            });
192:        }
```
[184](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L184-L192), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

29:        if (!borrowOffer.isNull()) {	// @audit-issue
30:            // validate msg.sender
31:            // N/A
32:
33:            // validate curveRelativeTime
34:            YieldCurveLibrary.validateYieldCurve(
35:                params.curveRelativeTime, state.riskConfig.minTenor, state.riskConfig.maxTenor
36:            );
37:        }
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L29-L37), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

179:        if (params.creditPositionId == RESERVED_ID) {	// @audit-issue
180:            // slither-disable-next-line unused-return
181:            state.createDebtAndCreditPositions({
182:                lender: msg.sender,
183:                borrower: borrower,
184:                futureValue: creditAmountOut,
185:                dueDate: block.timestamp + tenor
186:            });
187:        } else {
188:            state.createCreditPosition({
189:                exitCreditPositionId: params.creditPositionId,
190:                lender: msg.sender,
191:                credit: creditAmountOut
192:            });
193:        }
```
[179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L179-L193), 


#### Recommendation

Consider using single line if statements as ternary.

### Custom error has no error details
Consider adding parameters to the error to indicate which user or values caused the failure

```solidity
Path: ./src/libraries/Errors.sol

11:    error NULL_ADDRESS();	// @audit-issue

12:    error NULL_AMOUNT();	// @audit-issue

13:    error NULL_TENOR();	// @audit-issue

14:    error NULL_MAX_DUE_DATE();	// @audit-issue

15:    error NULL_ARRAY();	// @audit-issue

16:    error NULL_OFFER();	// @audit-issue

18:    error TENORS_NOT_STRICTLY_INCREASING();	// @audit-issue

19:    error ARRAY_LENGTHS_MISMATCH();	// @audit-issue

73:    error NULL_STALE_PRICE();	// @audit-issue

74:    error NULL_STALE_RATE();	// @audit-issue

80:    error NOT_SUPPORTED();	// @audit-issue

82:    error SEQUENCER_DOWN();	// @audit-issue

83:    error GRACE_PERIOD_NOT_OVER();	// @audit-issue
```
[11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L12-L12), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L14-L14), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L15-L15), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L16-L16), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L18-L18), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L19-L19), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L73-L73), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L74-L74), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L80-L80), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L82-L82), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L83-L83), 


#### Recommendation

When defining custom errors, consider adding parameters or error details that provide information about the specific conditions or inputs that caused the error. Including error details can make debugging and troubleshooting easier by providing context on the cause of the failure.

### Consider moving `msg.sender` checks to `modifier`s
If some functions are only allowed to be called by some specific users, consider using a modifier instead of checking with a require statement, especially if this check is done in multiple functions.

```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

50:            if (creditPosition.lender != msg.sender) {
51:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionIds[i]);	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L50-L51), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

51:        if (msg.sender != creditPosition.lender) {
52:            revert Errors.LIQUIDATOR_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L51-L52), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

93:        if (msg.sender != debtPositionToRepay.borrower) {
94:            revert Errors.COMPENSATOR_IS_NOT_BORROWER(msg.sender, debtPositionToRepay.borrower);	// @audit-issue
```
[94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L93-L94), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

74:            if (msg.sender != creditPosition.lender) {
75:                revert Errors.BORROWER_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue
```
[75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L74-L75), 


#### Recommendation

Consider refactoring your code by moving `msg.sender` checks to modifiers when certain functions are only allowed to be called by specific users. This approach can enhance code readability, reduce redundancy, and make it easier to maintain access control logic.

### Constants in comparisons should appear on the left side
Doing so will prevent [typo bugs](https://www.moserware.com/2008/01/constants-on-left-are-better-but-this.html)

```solidity
Path: ./src/SizeView.sol

180:        if (tenor == 0) {	// @audit-issue
```
[180](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L180-L180), 


```solidity
Path: ./src/oracle/PriceFeed.sol

48:        if (_baseStalePriceInterval == 0 || _quoteStalePriceInterval == 0) {	// @audit-issue

68:            if (answer == 1) {	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L48-L48), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L68-L68), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

59:        if (debt != 0) {	// @audit-issue
```
[59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L59-L59), 


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
```
[98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L98-L98), 


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
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L42-L42), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

39:            if (params.maxDueDate == 0) {	// @audit-issue
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L39-L39), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

41:        if (msg.value != 0 && (msg.value != params.amount || params.token != address(state.data.weth))) {	// @audit-issue

54:        if (params.amount == 0) {	// @audit-issue
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L41-L41), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

22:        if (decimals_ == 0) {	// @audit-issue
```
[22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L22-L22), 


#### Recommendation

To prevent typo bugs and improve code readability, it's advisable to place constants on the left side of comparisons. This coding practice helps catch accidental assignment (=) instead of comparison (==) and enhances code quality.

### Convert duplicated codes to modifier/functions
Duplicated code in Solidity contracts, especially when repeated across multiple functions, can lead to inefficiencies and increased maintenance challenges. It not only bloats the contract but also makes it harder to implement changes consistently across the codebase. By identifying common patterns or checks that are repeated and abstracting them into modifiers or separate functions, the code can be made more concise, readable, and maintainable. This practice not only reduces the overall bytecode size, potentially lowering deployment and execution costs, but also enhances the contract's reliability by ensuring consistency in how these common checks or operations are executed.

```solidity
Path: ./src/libraries/actions/Compensate.sol

111:        CreditPosition storage creditPositionWithDebtToRepay =	// @audit-issue: Exactly same copy pasted functionality between lines: `{'43->48'}`
112:            state.getCreditPosition(params.creditPositionWithDebtToRepayId);
113:        DebtPosition storage debtPositionToRepay =
114:            state.getDebtPositionByCreditPositionId(params.creditPositionWithDebtToRepayId);
115:
116:        uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);
```
[111](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L111-L116), 


#### Recommendation

Identify and consolidate duplicated code blocks in Solidity contracts into reusable modifiers or functions. This approach streamlines the contract by eliminating redundancy, thereby improving clarity, maintainability, and potentially reducing gas costs. For common checks, consider using modifiers for concise and consistent enforcement of conditions. For reusable logic, encapsulate it in functions to avoid code duplication and simplify future updates or bug fixes.

### Style guide: Function ordering does not follow the Solidity style guide
According to the Solidity [style guide](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions), functions should be laid out in the following order :`constructor()`, `receive()`, `fallback()`, `external`, `public`, `internal`, `private`, but the cases below do not follow this pattern

```solidity
Path: ./src/Size.sol

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue
```
[110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L110), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

104:    function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {	// @audit-issue: isDebtPositionLiquidatable should come before than isCreditPositionTransferrable
```
[104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L104-L104), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

80:    function getDebtPosition(State storage state, uint256 debtPositionId) public view returns (DebtPosition storage) {	// @audit-issue
```
[80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L80-L80), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

115:    function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue
```
[115](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L115-L115), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

42:    function repayDebt(State storage state, uint256 debtPositionId, uint256 repayAmount) public {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L42-L42), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L79), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

175:    function validateInitialize(	// @audit-issue
```
[175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L175), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

105:    function balanceOf(address account) public view override returns (uint256) {	// @audit-issue: balanceOf should come before than _unscale
```
[105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L105-L105), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/latest/style-guide.html).

### Duplicated `require()`/`revert()` checks should be refactored to a modifier or function
In Solidity contracts, it's common to encounter duplicated `require()` or `revert()` statements across multiple functions. These statements are crucial for validating conditions and ensuring contract integrity. However, repeated checks can lead to code redundancy, making the contract larger and more difficult to maintain. This redundancy can be streamlined by encapsulating the repeated logic in a modifier or a dedicated function. By doing so, the code becomes more concise, easier to audit, and more gas-efficient. Furthermore, centralizing the validation logic in a single location makes the codebase more adaptable to changes and reduces the risk of inconsistencies or errors in future updates.

```solidity
Path: ./src/SizeView.sol

143:        if (offer.isNull()) {	// @audit-issue: Same if statement in line(s): ['159']
```
[143](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L143-L143), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

53:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue: Same if statement in line(s): ['81']
```
[53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L53-L53), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

199:            if (fees > maxCashAmountOut) {	// @audit-issue: Same if statement in line(s): ['209']
```
[199](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L199-L199), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

99:            if (	// @audit-issue: Same if statement in line(s): ['109']
```
[99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L99-L99), 


#### Recommendation

Consolidate repeated `require()` or `revert()` checks in Solidity by creating a modifier or a separate function. Apply this modifier to all functions requiring the same validation, or call the function at the beginning of each relevant function. This refactoring enhances contract efficiency, maintainability, and consistency, while potentially reducing gas costs associated with deploying and executing redundant code.

### Duplicate import statements
Contracts often depend on libraries and other contracts to modularize code and reuse functionalities. However, redundant imports occur when a contract imports a library or another contract that is already imported by one of its dependencies. This redundancy does not impact the compiled bytecode but can clutter the codebase, making it harder to understand the direct dependencies of each contract. Simplifying imports by removing these redundancies enhances code readability and maintainability.

```solidity
Path: ./src/Size.sol

9:import {RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue: Same library is also imported on: `['SizeView']`, at lines: `[7]`

11:import {	// @audit-issue: Same library is also imported on: `['ISize', 'SizeView']`, at lines: `[20, 26]`

18:import {UpdateConfig, UpdateConfigParams} from "@src/libraries/actions/UpdateConfig.sol";	// @audit-issue: Same library is also imported on: `['SizeView']`, at lines: `[16]`

20:import {SellCreditLimit, SellCreditLimitParams} from "@src/libraries/actions/SellCreditLimit.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[4]`

21:import {SellCreditMarket, SellCreditMarketParams} from "@src/libraries/actions/SellCreditMarket.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[5]`

23:import {Claim, ClaimParams} from "@src/libraries/actions/Claim.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[7]`

24:import {Deposit, DepositParams} from "@src/libraries/actions/Deposit.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[12]`

26:import {BuyCreditMarket, BuyCreditMarketParams} from "@src/libraries/actions/BuyCreditMarket.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[28]`

27:import {SetUserConfiguration, SetUserConfigurationParams} from "@src/libraries/actions/SetUserConfiguration.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[29]`

29:import {BuyCreditLimit, BuyCreditLimitParams} from "@src/libraries/actions/BuyCreditLimit.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[9]`

30:import {Liquidate, LiquidateParams} from "@src/libraries/actions/Liquidate.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[10]`

33:import {Compensate, CompensateParams} from "@src/libraries/actions/Compensate.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[19]`

34:import {	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[15]`

38:import {Repay, RepayParams} from "@src/libraries/actions/Repay.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[16]`

39:import {SelfLiquidate, SelfLiquidateParams} from "@src/libraries/actions/SelfLiquidate.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[17]`

40:import {Withdraw, WithdrawParams} from "@src/libraries/actions/Withdraw.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[13]`

42:import {State} from "@src/SizeStorage.sol";	// @audit-issue: Same library is also imported on: `['SizeView']`, at lines: `[4]`

45:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue: Same library is also imported on: `['SizeView']`, at lines: `[19]`

50:import {IMulticall} from "@src/interfaces/IMulticall.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[26]`

52:import {ISizeAdmin} from "@src/interfaces/ISizeAdmin.sol";	// @audit-issue: Same library is also imported on: `['ISize']`, at lines: `[31]`
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L11-L11), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L18-L18), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L21-L21), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L24-L24), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L27-L27), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L30-L30), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L33-L33), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L34-L34), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L39-L39), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L40-L40), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L42-L42), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L45-L45), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L50-L50), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L52-L52), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

11:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue: Same library is also imported on: `['NonTransferrableToken']`, at lines: `[7]`
```
[11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L11-L11), 


#### Recommendation

Review your Solidity contracts and eliminate duplicate import statements. Ensure each file is imported only once where it's needed. If you find the same import statement across multiple files, consider whether you can restructure your code to centralize common logic or dependencies, potentially through base contracts or libraries. Regularly conduct code audits or utilize linters and other static analysis tools to identify and resolve duplicate imports, thereby enhancing the clarity, structure, and security of your Solidity codebase.

### Adding a return statement when the function defines a named return variable, is redundant
Once the return variable has been assigned (or has its default value), there is no need to explicitly return it at the end of the function, since it's returned automatically.

```solidity
Path: ./src/libraries/Math.sol

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {
52:        low = 0;
53:        high = array.length - 1;
54:        if (value < array[low] || value > array[high]) {
55:            return (type(uint256).max, type(uint256).max);	// @audit-issue
56:        }
57:        while (low <= high) {
58:            uint256 mid = (low + high) / 2;
59:            if (array[mid] == value) {
60:                return (mid, mid);
61:            } else if (array[mid] < value) {
62:                low = mid + 1;
63:            } else {
64:                high = mid - 1;
65:            }
66:        }
67:        return (high, low);
68:    }

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {
52:        low = 0;
53:        high = array.length - 1;
54:        if (value < array[low] || value > array[high]) {
55:            return (type(uint256).max, type(uint256).max);
56:        }
57:        while (low <= high) {
58:            uint256 mid = (low + high) / 2;
59:            if (array[mid] == value) {
60:                return (mid, mid);	// @audit-issue
61:            } else if (array[mid] < value) {
62:                low = mid + 1;
63:            } else {
64:                high = mid - 1;
65:            }
66:        }
67:        return (high, low);
68:    }

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {
52:        low = 0;
53:        high = array.length - 1;
54:        if (value < array[low] || value > array[high]) {
55:            return (type(uint256).max, type(uint256).max);
56:        }
57:        while (low <= high) {
58:            uint256 mid = (low + high) / 2;
59:            if (array[mid] == value) {
60:                return (mid, mid);
61:            } else if (array[mid] < value) {
62:                low = mid + 1;
63:            } else {
64:                high = mid - 1;
65:            }
66:        }
67:        return (high, low);	// @audit-issue
68:    }
```
[55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L68), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L68), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L68), 


#### Recommendation

When a function defines a named return variable and assigns a value to it, there is no need to add an explicit return statement at the end of the function. The named return variable will be automatically returned with its assigned value. Removing the redundant return statement can lead to cleaner and more concise code, improving readability and reducing the risk of introducing unnecessary errors.

### Consider using `delete` rather than assigning values to `false`
The `delete` keyword more closely matches the semantics of what is being done, and draws more attention to the changing of state, which may lead to a more thorough audit of its associated logic.


```solidity
Path: ./src/libraries/Multicall.sol

44:        state.data.isMulticall = false;	// @audit-issue
```
[44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L44-L44), 


#### Recommendation

Prefer using the `delete` keyword to reset state variables to their initial values instead of assigning `false` or other values, as it makes the intent clearer and helps with auditing.

### Consider using `delete` rather than assigning zero to clear values
The `delete` keyword more closely matches the semantics of what is being done, and draws more attention to the changing of state, which may lead to a more thorough audit of its associated logic.

```solidity
Path: ./src/libraries/Math.sol

52:        low = 0;	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

151:        debtPosition.liquidityIndexAtRepayment = 0;	// @audit-issue
```
[151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L151-L151), 


#### Recommendation

When you need to clear or reset values in storage variables, consider using the `delete` keyword instead of manually assigning zero or other values. Using `delete` provides a more explicit and efficient way to clear storage variables. For example, instead of `myVariable = 0;`, you can use `delete myVariable;` to clear the value.

### Too long functions should be refactored
Functions with too many lines are difficult to understand. It is recommended to refactor complex functions into multiple shorter and easier to understand functions.


```solidity
Path: ./src/libraries/actions/Compensate.sol

42:    function validateCompensate(State storage state, CompensateParams calldata params) external view {	// @audit-issue 59 lines

106:    function executeCompensate(State storage state, CompensateParams calldata params) external {	// @audit-issue 50 lines
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L42-L42), [106](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L106-L106), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

51:    function validateSellCreditMarket(State storage state, SellCreditMarketParams calldata params) external view {	// @audit-issue 71 lines

127:    function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)	// @audit-issue 76 lines
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L51-L51), [127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L127-L127), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

86:    function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {	// @audit-issue 62 lines
```
[86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L86-L86), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

75:    function executeLiquidate(State storage state, LiquidateParams calldata params)	// @audit-issue 51 lines
```
[75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L75-L75), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

51:    function validateBuyCreditMarket(State storage state, BuyCreditMarketParams calldata params) external view {	// @audit-issue 64 lines

121:    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)	// @audit-issue 76 lines
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L51-L51), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L121-L121), 


#### Recommendation

To address this issue, refactor long and complex functions into multiple shorter and more manageable functions. This will improve code readability and maintainability, making it easier to understand and maintain your smart contract.

### Expressions for `constant` values should use `immutable` rather than `constant`
While it does not save gas for some simple binary expressions because the compiler knows that developers often make this mistake, it's still best to use the right tool for the task at hand. There is a difference between `constant` variables and `immutable` variables, and they should each be used in their appropriate contexts. `constant`s should be used for literal values written into the code, and `immutable` variables should be used for expressions, or values calculated in, or passed into the `constructor`.

```solidity
Path: ./src/Size.sol

54:bytes32 constant KEEPER_ROLE = keccak256("KEEPER_ROLE");	// @audit-issue

55:bytes32 constant PAUSER_ROLE = keccak256("PAUSER_ROLE");	// @audit-issue

56:bytes32 constant BORROW_RATE_UPDATER_ROLE = keccak256("BORROW_RATE_UPDATER_ROLE");	// @audit-issue
```
[54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L54-L54), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L55-L55), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L56-L56), 


#### Recommendation

To improve code readability and adhere to best practices, consider using `immutable` variables instead of `constant` for expressions or values calculated in, or passed into the constructor. While the compiler may handle this issue for simple binary expressions, it's essential to use the right tool for the task at hand. `constant` should be reserved for literal values written directly into the code, while `immutable` is better suited for dynamic values and expressions.

### Lines are too long
Usually lines in source code are limited to [80](https://softwareengineering.stackexchange.com/questions/148677/why-is-80-characters-the-standard-limit-for-code-width) characters. Today's screens are much larger so it's reasonable to stretch this in some cases. The solidity style guide recommends a maximumum line length of [120 characters](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#maximum-line-length), so the lines below should be split when they reach that length.        self.impact_details = 


```solidity
Path: ./src/oracle/PriceFeed.sol

14:/// @notice A contract that provides the price of a `base` asset in terms of a `quote` asset, using an intermediate asset, scaled to 18 decimals	// @audit-issue: Length of the line is: 145

18:///      _sequencerUptimeFeed: the sequencer uptime feed for supported L2s (https://docs.chain.link/data-feeds/l2-sequencer-feeds)	// @audit-issue: Length of the line is: 131
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L14-L14), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L18-L18), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

67:    /// @dev A credit position is self-liquidatable if the user is underwater and the loan is not REPAID (ie, ACTIVE or OVERDUE)	// @audit-issue: Length of the line is: 129

99:    /// @dev A debt position is liquidatable if the user is underwater and the loan is not REPAID (ie, ACTIVE or OVERDUE) or	// @audit-issue: Length of the line is: 125

138:    ///      If the user has not set a custom opening limit borrow CR, the default is the global opening limit borrow CR	// @audit-issue: Length of the line is: 121

143:            state.data.users[account].openingLimitBorrowCR // 0 by default, or user-defined if SetUserConfiguration has been used	// @audit-issue: Length of the line is: 130
```
[67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L67-L67), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L99-L99), [138](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L138-L138), [143](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L143-L143), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

132:                    getAdjustedAPR(curveRelativeTime.aprs[high], curveRelativeTime.marketRateMultipliers[high], params);	// @audit-issue: Length of the line is: 121
```
[132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L132-L132), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

70:            DebtPosition({borrower: borrower, futureValue: futureValue, dueDate: dueDate, liquidityIndexAtRepayment: 0});	// @audit-issue: Length of the line is: 122

96:    ///      If the credit amount is different, the existing credit position is reduced and a new credit position is created.	// @audit-issue: Length of the line is: 126

134:    ///        If the loan is in ACTIVE status, a debt reduction must be performed together with a credit reduction (See reduceDebtAndCredit).	// @audit-issue: Length of the line is: 143
```
[70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L70-L70), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L96-L96), [134](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L134-L134), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

12:    /// @notice Validate that the increase in borrow aToken supply is less than or equal to the decrease in debt token supply	// @audit-issue: Length of the line is: 126

50:    ///      Due to rounding, the borrow aToken supply may be slightly less than the actual AToken supply, which is acceptable.	// @audit-issue: Length of the line is: 128

62:    ///      This safety mechanism prevents takers from matching orders that could not be withdrawn from the Variable Pool.	// @audit-issue: Length of the line is: 124

63:    ///        Nevertheless, the Variable Pool may still fail to withdraw the cash due to other factors (such as a pause, etc),	// @audit-issue: Length of the line is: 128
```
[12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L12-L12), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L50-L50), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L62-L62), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L63-L63), 


```solidity
Path: ./src/libraries/Events.sol

90:        uint256 indexed debtPositionId, address indexed borrower, uint256 futureValue, uint256 liquidityIndexAtRepayment	// @audit-issue: Length of the line is: 121
```
[90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L90-L90), 


```solidity
Path: ./src/libraries/Multicall.sol

13:/// @author OpenZeppelin (https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v5.0.2/contracts/utils/Multicall.sol), Size	// @audit-issue: Length of the line is: 140
```
[13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L13-L13), 


```solidity
Path: ./src/libraries/Math.sol

45:    /// @dev If `value` is below the lowest value in `array` or above the highest value in `array`, the function returns (type(uint256).max, type(uint256).max)	// @audit-issue: Length of the line is: 160
```
[45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L45-L45), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

47:            revert Errors.LIQUIDATION_NOT_AT_LOSS(params.creditPositionId, state.collateralRatio(debtPosition.borrower));	// @audit-issue: Length of the line is: 122
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L47-L47), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

84:            tenor = debtPosition.dueDate - block.timestamp; // positive since the credit position is transferrable, so the loan must be ACTIVE	// @audit-issue: Length of the line is: 143

175:                    : Math.mulDivDown(creditPosition.credit, PERCENT - state.getSwapFeePercent(tenor), PERCENT + ratePerTenor),	// @audit-issue: Length of the line is: 128
```
[84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L84-L84), [175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L175-L175), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

34:///      A `key` string is used to identify the configuration parameter to update and a `value` uint256 is used to set the new value	// @audit-issue: Length of the line is: 133
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L34-L34), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

85:        // if the loan is both underwater and overdue, the protocol fee related to underwater liquidations takes precedence	// @audit-issue: Length of the line is: 124
```
[85](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L85-L85), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

58:/// @dev The collateralToken (e.g. szETH), borrowAToken (e.g. szaUSDC), and debtToken (e.g. szDebt) are created in the `executeInitialize` function	// @audit-issue: Length of the line is: 148
```
[58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L58-L58), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

80:            tenor = debtPosition.dueDate - block.timestamp; // positive since the credit position is transferrable, so the loan must be ACTIVE	// @audit-issue: Length of the line is: 143
```
[80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L80-L80), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

18:///      Enables the owner to mint and burn scaled amounts. Emits the TransferUnscaled event representing the actual unscaled amount	// @audit-issue: Length of the line is: 133
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L18-L18), 


#### Recommendation

To adhere to coding standards and enhance code readability, consider splitting long lines of code when they approach the recommended maximum line length. In Solidity, a common guideline is to limit lines to a maximum of 120 characters. Splitting long lines can improve code maintainability and make it easier to understand.

### Dependence on external protocols
External protocols should be monitored as such dependencies may introduce vulnerabilities if a vulnerability is found /introduced in the external protocol

```solidity
Path: ./src/SizeViewData.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

7:import {User} from "@src/SizeStorage.sol";	// @audit-issue

8:import {NonTransferrableScaledToken} from "@src/token/NonTransferrableScaledToken.sol";	// @audit-issue

9:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L5-L5), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L9-L9), 


```solidity
Path: ./src/Size.sol

4:import {AccessControlUpgradeable} from "@openzeppelin/contracts-upgradeable/access/AccessControlUpgradeable.sol";	// @audit-issue

6:import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";	// @audit-issue

7:import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";	// @audit-issue

8:import {PausableUpgradeable} from "@openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol";	// @audit-issue

9:import {RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

11:import {	// @audit-issue

18:import {UpdateConfig, UpdateConfigParams} from "@src/libraries/actions/UpdateConfig.sol";	// @audit-issue

20:import {SellCreditLimit, SellCreditLimitParams} from "@src/libraries/actions/SellCreditLimit.sol";	// @audit-issue

21:import {SellCreditMarket, SellCreditMarketParams} from "@src/libraries/actions/SellCreditMarket.sol";	// @audit-issue

23:import {Claim, ClaimParams} from "@src/libraries/actions/Claim.sol";	// @audit-issue

24:import {Deposit, DepositParams} from "@src/libraries/actions/Deposit.sol";	// @audit-issue

26:import {BuyCreditMarket, BuyCreditMarketParams} from "@src/libraries/actions/BuyCreditMarket.sol";	// @audit-issue

27:import {SetUserConfiguration, SetUserConfigurationParams} from "@src/libraries/actions/SetUserConfiguration.sol";	// @audit-issue

29:import {BuyCreditLimit, BuyCreditLimitParams} from "@src/libraries/actions/BuyCreditLimit.sol";	// @audit-issue

30:import {Liquidate, LiquidateParams} from "@src/libraries/actions/Liquidate.sol";	// @audit-issue

32:import {Multicall} from "@src/libraries/Multicall.sol";	// @audit-issue

33:import {Compensate, CompensateParams} from "@src/libraries/actions/Compensate.sol";	// @audit-issue

34:import {	// @audit-issue

38:import {Repay, RepayParams} from "@src/libraries/actions/Repay.sol";	// @audit-issue

39:import {SelfLiquidate, SelfLiquidateParams} from "@src/libraries/actions/SelfLiquidate.sol";	// @audit-issue

40:import {Withdraw, WithdrawParams} from "@src/libraries/actions/Withdraw.sol";	// @audit-issue

42:import {State} from "@src/SizeStorage.sol";	// @audit-issue

44:import {CapsLibrary} from "@src/libraries/CapsLibrary.sol";	// @audit-issue

45:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

47:import {SizeView} from "@src/SizeView.sol";	// @audit-issue

48:import {Events} from "@src/libraries/Events.sol";	// @audit-issue

50:import {IMulticall} from "@src/interfaces/IMulticall.sol";	// @audit-issue

51:import {ISize} from "@src/interfaces/ISize.sol";	// @audit-issue

52:import {ISizeAdmin} from "@src/interfaces/ISizeAdmin.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L11-L11), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L18-L18), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L21-L21), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L24-L24), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L27-L27), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L30-L30), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L33-L33), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L34-L34), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L39-L39), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L40-L40), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L42-L42), [44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L44-L44), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L45-L45), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L47-L47), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L48-L48), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L50-L50), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L52-L52), 


```solidity
Path: ./src/SizeStorage.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

6:import {IWETH} from "@src/interfaces/IWETH.sol";	// @audit-issue

8:import {CreditPosition, DebtPosition} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

9:import {BorrowOffer, LoanOffer} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

11:import {IPriceFeed} from "@src/oracle/IPriceFeed.sol";	// @audit-issue

13:import {NonTransferrableScaledToken} from "@src/token/NonTransferrableScaledToken.sol";	// @audit-issue

14:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L11-L11), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L14-L14), 


```solidity
Path: ./src/SizeView.sol

4:import {SizeStorage, State, User} from "@src/SizeStorage.sol";	// @audit-issue

5:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue

7:import {	// @audit-issue

16:import {UpdateConfig} from "@src/libraries/actions/UpdateConfig.sol";	// @audit-issue

18:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

19:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

21:import {DataView, UserView} from "@src/SizeViewData.sol";	// @audit-issue

23:import {ISizeView} from "@src/interfaces/ISizeView.sol";	// @audit-issue

24:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

25:import {BorrowOffer, LoanOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

26:import {	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L5-L5), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L7-L7), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L16-L16), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L18-L18), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L19-L19), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L21-L21), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L25-L25), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L26-L26), 


```solidity
Path: ./src/oracle/PriceFeed.sol

4:import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";	// @audit-issue

5:import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";	// @audit-issue

6:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

9:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L6-L6), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L9-L9), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

8:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

9:import {Math} from "@src/libraries/Math.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L9-L9), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

4:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

5:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

6:import {VariablePoolBorrowRateParams, YieldCurve, YieldCurveLibrary} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L6-L6), 


```solidity
Path: ./src/libraries/Errors.sol

4:import {LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L4-L4), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

7:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

8:import {Math} from "@src/libraries/Math.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L8-L8), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

4:import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";	// @audit-issue

5:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

6:import {Math, PERCENT} from "@src/libraries/Math.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L6-L6), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

7:import {Events} from "@src/libraries/Events.sol";	// @audit-issue

8:import {Math, PERCENT, YEAR} from "@src/libraries/Math.sol";	// @audit-issue

10:import {CreditPosition, DebtPosition, LoanLibrary, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

11:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L8-L8), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L11-L11), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

5:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L5-L5), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

4:import {IAToken} from "@aave/interfaces/IAToken.sol";	// @audit-issue

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

6:import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";	// @audit-issue

8:import {State} from "@src/SizeStorage.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L8-L8), 


```solidity
Path: ./src/libraries/Events.sol

4:import {LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

5:import {	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L5-L5), 


```solidity
Path: ./src/libraries/Multicall.sol

4:import {Address} from "@openzeppelin/contracts/utils/Address.sol";	// @audit-issue

6:import {State} from "@src/SizeStorage.sol";	// @audit-issue

7:import {CapsLibrary} from "@src/libraries/CapsLibrary.sol";	// @audit-issue

8:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L8-L8), 


```solidity
Path: ./src/libraries/Math.sol

4:import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L4-L4), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

4:import {State, User} from "@src/SizeStorage.sol";	// @audit-issue

6:import {CreditPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

8:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

9:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L9-L9), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

4:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

6:import {CreditPosition, DebtPosition, LoanLibrary} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

7:import {PERCENT} from "@src/libraries/Math.sol";	// @audit-issue

8:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

10:import {State} from "@src/SizeStorage.sol";	// @audit-issue

12:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

13:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L8-L8), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L10-L10), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L12-L12), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L13-L13), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

8:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

10:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

11:import {Events} from "@src/libraries/Events.sol";	// @audit-issue

12:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

14:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L8-L8), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L12-L12), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L14-L14), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

4:import {CreditPosition, DebtPosition, LoanLibrary, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

5:import {Math, PERCENT} from "@src/libraries/Math.sol";	// @audit-issue

6:import {LoanOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

7:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue

9:import {State} from "@src/SizeStorage.sol";	// @audit-issue

11:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

12:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

14:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

15:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L12-L12), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L14-L14), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L15-L15), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

4:import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";	// @audit-issue

5:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

7:import {Events} from "@src/libraries/Events.sol";	// @audit-issue

9:import {Math, PERCENT, YEAR} from "@src/libraries/Math.sol";	// @audit-issue

10:import {Initialize} from "@src/libraries/actions/Initialize.sol";	// @audit-issue

12:import {IPriceFeed} from "@src/oracle/IPriceFeed.sol";	// @audit-issue

14:import {	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L9-L9), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L10-L10), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L12-L12), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L14-L14), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

4:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

6:import {PERCENT} from "@src/libraries/Math.sol";	// @audit-issue

8:import {DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

10:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

11:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

13:import {State} from "@src/SizeStorage.sol";	// @audit-issue

15:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

16:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L8-L8), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L11-L11), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L13-L13), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L15-L15), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L16-L16), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

4:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

6:import {PERCENT} from "@src/libraries/Math.sol";	// @audit-issue

8:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

9:import {BorrowOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

10:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue

12:import {State} from "@src/SizeStorage.sol";	// @audit-issue

14:import {Liquidate, LiquidateParams} from "@src/libraries/actions/Liquidate.sol";	// @audit-issue

16:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

17:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L9-L9), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L10-L10), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L12-L12), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L14-L14), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L16-L16), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L17-L17), 


```solidity
Path: ./src/libraries/actions/Claim.sol

4:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

5:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

7:import {State} from "@src/SizeStorage.sol";	// @audit-issue

9:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

11:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

12:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L5-L5), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L12-L12), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue

5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

6:import {IWETH} from "@src/interfaces/IWETH.sol";	// @audit-issue

8:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

10:import {CREDIT_POSITION_ID_START, DEBT_POSITION_ID_START} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

11:import {PERCENT} from "@src/libraries/Math.sol";	// @audit-issue

13:import {IPriceFeed} from "@src/oracle/IPriceFeed.sol";	// @audit-issue

15:import {NonTransferrableScaledToken} from "@src/token/NonTransferrableScaledToken.sol";	// @audit-issue

16:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";	// @audit-issue

18:import {State} from "@src/SizeStorage.sol";	// @audit-issue

20:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

21:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L8-L8), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L11-L11), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L13-L13), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L15-L15), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L16-L16), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L18-L18), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L21-L21), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {DepositTokenLibrary} from "@src/libraries/DepositTokenLibrary.sol";	// @audit-issue

7:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

9:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

10:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L9-L9), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L10-L10), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

5:import {BorrowOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

6:import {YieldCurve, YieldCurveLibrary} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue

8:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L8-L8), 


```solidity
Path: ./src/libraries/actions/Repay.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue

6:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

7:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

9:import {DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

11:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

12:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L12-L12), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

4:import {LoanOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

5:import {YieldCurve, YieldCurveLibrary} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue

7:import {State} from "@src/SizeStorage.sol";	// @audit-issue

9:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

10:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L5-L5), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L9-L9), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L10-L10), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

4:import {State, User} from "@src/SizeStorage.sol";	// @audit-issue

6:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue

7:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

8:import {Events} from "@src/libraries/Events.sol";	// @audit-issue

9:import {CreditPosition, DebtPosition, LoanLibrary, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue

10:import {Math, PERCENT} from "@src/libraries/Math.sol";	// @audit-issue

11:import {BorrowOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue

13:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";	// @audit-issue

14:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L4-L4), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L7-L7), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L9-L9), [10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L11-L11), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L14-L14), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

4:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue

5:import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";	// @audit-issue

6:import {IWETH} from "@src/interfaces/IWETH.sol";	// @audit-issue

7:import {CapsLibrary} from "@src/libraries/CapsLibrary.sol";	// @audit-issue

9:import {State} from "@src/SizeStorage.sol";	// @audit-issue

11:import {DepositTokenLibrary} from "@src/libraries/DepositTokenLibrary.sol";	// @audit-issue

13:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue

14:import {Events} from "@src/libraries/Events.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L6-L6), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L7-L7), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L11-L11), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L14-L14), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

4:import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";	// @audit-issue

5:import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";	// @audit-issue

7:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L5-L5), [7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L7-L7), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue

5:import {WadRayMath} from "@aave/protocol/libraries/math/WadRayMath.sol";	// @audit-issue

6:import {IERC20Metadata} from "@openzeppelin/contracts/interfaces/IERC20Metadata.sol";	// @audit-issue

8:import {Math} from "@src/libraries/Math.sol";	// @audit-issue

9:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";	// @audit-issue

11:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L4-L4), [5](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L5-L5), [6](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L6-L6), [8](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L8-L8), [9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L11-L11), 


#### Recommendation

Regularly monitor and review the external protocols your Solidity contracts depend on for any updates or identified vulnerabilities. Consider implementing fallback mechanisms or contingency plans in your contracts to handle potential failures or compromises in these external protocols. Additionally, thoroughly audit and test the integration points with external protocols to ensure they adhere to your contract's security standards. Where possible, design your contract architecture to minimize reliance on external protocols or allow for upgradability in response to changes in these dependencies. Stay informed about developments in the protocols you depend on and actively participate in their community for early awareness of potential issues.

### Incorrect withdraw declaration
In Solidity, it's essential for clarity and interoperability to correctly specify return types in function declarations. If the `withdraw` function is expected to return a `bool` to indicate success or failure, its omission could lead to ambiguity or unexpected behavior when interacting with or calling this function from other contracts or off-chain systems. Missing return values can mislead developers and potentially lead to contract integrations built on incorrect assumptions. To resolve this, the function declaration for `withdraw` should be modified to explicitly include the `bool` return type, ensuring clarity and correctness in contract interactions.

```solidity
Path: ./src/Size.sol

159:    function withdraw(WithdrawParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue
```
[159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L159-L159), 


#### Recommendation

Review the function declarations in your Solidity contracts, particularly for critical operations like `withdraw`, to ensure that they correctly specify return types. If a function is intended to indicate success or failure, it should explicitly declare a `bool` return type in its signature. For example, modify the `withdraw` function declaration from:
```solidity
function withdraw(uint256 amount) public {
    // Implementation
}

// to

function withdraw(uint256 amount) public returns (bool) {
    // Implementation
    return true; // Indicate success
}
```


### Style Guide: Surround top level declarations in Solidity source with two blank lines.
1- Surround top level declarations in Solidity source with two blank lines.
2- Within a contract surround function declarations with a single blank line.


```solidity
Path: ./src/Size.sol

80:    using Multicall for State;
81:
82:    /// @custom:oz-upgrades-unsafe-allow constructor	// @audit-issue: There should be single blank line between function declarations.
83:    constructor() {

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}
108:
109:    /// @inheritdoc ISizeAdmin	// @audit-issue: There should be single blank line between function declarations.
110:    function updateConfig(UpdateConfigParams calldata params)

117:    }
118:
119:    /// @inheritdoc ISizeAdmin	// @audit-issue: There should be single blank line between function declarations.
120:    function setVariablePoolBorrowRate(uint128 borrowRate)

129:    }
130:
131:    /// @inheritdoc ISizeAdmin	// @audit-issue: There should be single blank line between function declarations.
132:    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {

134:    }
135:
136:    /// @inheritdoc ISizeAdmin	// @audit-issue: There should be single blank line between function declarations.
137:    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {

139:    }
140:
141:    /// @inheritdoc IMulticall	// @audit-issue: There should be single blank line between function declarations.
142:    function multicall(bytes[] calldata _data)

150:    }
151:
152:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
153:    function deposit(DepositParams calldata params) public payable override(ISize) whenNotPaused {

156:    }
157:
158:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
159:    function withdraw(WithdrawParams calldata params) external payable override(ISize) whenNotPaused {

163:    }
164:
165:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
166:    function buyCreditLimit(BuyCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {

169:    }
170:
171:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
172:    function sellCreditLimit(SellCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {

175:    }
176:
177:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
178:    function buyCreditMarket(BuyCreditMarketParams calldata params) external payable override(ISize) whenNotPaused {

185:    }
186:
187:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
188:    function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {

195:    }
196:
197:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
198:    function repay(RepayParams calldata params) external payable override(ISize) whenNotPaused {

201:    }
202:
203:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
204:    function claim(ClaimParams calldata params) external payable override(ISize) whenNotPaused {

207:    }
208:
209:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
210:    function liquidate(LiquidateParams calldata params)

220:    }
221:
222:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
223:    function selfLiquidate(SelfLiquidateParams calldata params) external payable override(ISize) whenNotPaused {

226:    }
227:
228:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)

244:    }
245:
246:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
247:    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {

251:    }
252:
253:    /// @inheritdoc ISize	// @audit-issue: There should be single blank line between function declarations.
254:    function setUserConfiguration(SetUserConfigurationParams calldata params)
```
[82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L80-L83), [109](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L110), [119](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L117-L120), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L129-L132), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L134-L137), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L139-L142), [152](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L150-L153), [158](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L156-L159), [165](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L163-L166), [171](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L169-L172), [177](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L175-L178), [187](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L185-L188), [197](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L195-L198), [203](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L201-L204), [209](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L207-L210), [222](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L220-L223), [228](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L226-L229), [246](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L244-L247), [253](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L251-L254), 


```solidity
Path: ./src/SizeView.sol

45:    using UpdateConfig for State;
46:
47:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
48:    function collateralRatio(address user) external view returns (uint256) {

50:    }
51:
52:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
53:    function isUserUnderwater(address user) external view returns (bool) {

55:    }
56:
57:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
58:    function isDebtPositionLiquidatable(uint256 debtPositionId) external view returns (bool) {

60:    }
61:
62:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
63:    function debtTokenAmountToCollateralTokenAmount(uint256 borrowATokenAmount) external view returns (uint256) {

65:    }
66:
67:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
68:    function feeConfig() external view returns (InitializeFeeConfigParams memory) {

70:    }
71:
72:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
73:    function riskConfig() external view returns (InitializeRiskConfigParams memory) {

75:    }
76:
77:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
78:    function oracle() external view returns (InitializeOracleParams memory) {

80:    }
81:
82:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
83:    function data() external view returns (DataView memory) {

94:    }
95:
96:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
97:    function getUserView(address user) external view returns (UserView memory) {

105:    }
106:
107:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
108:    function isDebtPositionId(uint256 debtPositionId) external view returns (bool) {

110:    }
111:
112:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
113:    function isCreditPositionId(uint256 creditPositionId) external view returns (bool) {

115:    }
116:
117:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
118:    function getDebtPosition(uint256 debtPositionId) external view returns (DebtPosition memory) {

120:    }
121:
122:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
123:    function getCreditPosition(uint256 creditPositionId) external view returns (CreditPosition memory) {

125:    }
126:
127:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
128:    function getLoanStatus(uint256 positionId) external view returns (LoanStatus) {

130:    }
131:
132:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
133:    function getPositionsCount() external view returns (uint256, uint256) {

138:    }
139:
140:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
141:    function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {

154:    }
155:
156:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
157:    function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {

170:    }
171:
172:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
173:    function getDebtPositionAssignedCollateral(uint256 debtPositionId) external view returns (uint256) {

176:    }
177:
178:    /// @inheritdoc ISizeView	// @audit-issue: There should be single blank line between function declarations.
179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L45-L48), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L50-L53), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L55-L58), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L60-L63), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L65-L68), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L70-L73), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L75-L78), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L80-L83), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L94-L97), [107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L105-L108), [112](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L110-L113), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L115-L118), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L120-L123), [127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L125-L128), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L130-L133), [140](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L138-L141), [156](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L154-L157), [172](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L170-L173), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L176-L179), 


```solidity
Path: ./src/oracle/PriceFeed.sol

33:    uint256 public immutable quoteStalePriceInterval;
34:    /* solhint-enable */
35:	// @audit-issue: There should be single blank line between function declarations.
36:    constructor(
```
[35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L33-L36), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

15:    using LoanLibrary for State;
16:
17:    /// @notice Validate the credit amount during an exit
18:    /// @dev Reverts if the remaining credit is lower than the minimum credit
19:    /// @param state The state
20:    /// @param credit The remaining credit	// @audit-issue: There should be single blank line between function declarations.
21:    function validateMinimumCredit(State storage state, uint256 credit) public view {

25:    }
26:
27:    /// @notice Validate the credit amount during an opening
28:    /// @dev Reverts if the credit is lower than the minimum credit
29:    /// @param state The state
30:    /// @param credit The credit	// @audit-issue: There should be single blank line between function declarations.
31:    function validateMinimumCreditOpening(State storage state, uint256 credit) public view {

35:    }
36:
37:    /// @notice Validate the tenor of a loan
38:    /// @dev Reverts if the tenor is out of range defined by minTenor and maxTenor
39:    /// @param state The state
40:    /// @param tenor The tenor	// @audit-issue: There should be single blank line between function declarations.
41:    function validateTenor(State storage state, uint256 tenor) public view {

45:    }
46:
47:    /// @notice Calculate the collateral ratio of an account
48:    /// @dev The collateral ratio is the ratio of the collateral to the debt
49:    ///      If the debt is 0, the collateral ratio is type(uint256).max
50:    /// @param state The state
51:    /// @param account The account
52:    /// @return The collateral ratio	// @audit-issue: There should be single blank line between function declarations.
53:    function collateralRatio(State storage state, address account) public view returns (uint256) {

64:    }
65:
66:    /// @notice Check if a credit position is self-liquidatable
67:    /// @dev A credit position is self-liquidatable if the user is underwater and the loan is not REPAID (ie, ACTIVE or OVERDUE)
68:    /// @param state The state
69:    /// @param creditPositionId The credit position ID
70:    /// @return True if the credit position is self-liquidatable, false otherwise	// @audit-issue: There should be single blank line between function declarations.
71:    function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)

82:    }
83:
84:    /// @notice Check if a credit position is transferrable
85:    /// @dev A credit position is transferrable if the loan is ACTIVE and the related borrower is not underwater
86:    /// @param state The state
87:    /// @param creditPositionId The credit position ID
88:    /// @return True if the credit position is transferrable, false otherwise	// @audit-issue: There should be single blank line between function declarations.
89:    function isCreditPositionTransferrable(State storage state, uint256 creditPositionId)

96:    }
97:
98:    /// @notice Check if a debt position is liquidatable
99:    /// @dev A debt position is liquidatable if the user is underwater and the loan is not REPAID (ie, ACTIVE or OVERDUE) or
100:    ///        if the loan is OVERDUE.
101:    /// @param state The state
102:    /// @param debtPositionId The debt position ID
103:    /// @return True if the debt position is liquidatable, false otherwise	// @audit-issue: There should be single blank line between function declarations.
104:    function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {

115:    }
116:
117:    /// @notice Check if the user is underwater
118:    /// @dev A user is underwater if the collateral ratio is below the liquidation threshold
119:    /// @param state The state
120:    /// @param account The account	// @audit-issue: There should be single blank line between function declarations.
121:    function isUserUnderwater(State storage state, address account) public view returns (bool) {

123:    }
124:
125:    /// @notice Validate that the user is not underwater
126:    /// @dev Reverts if the user is underwater
127:    /// @param state The state
128:    /// @param account The account	// @audit-issue: There should be single blank line between function declarations.
129:    function validateUserIsNotUnderwater(State storage state, address account) external view {

133:    }
134:
135:    /// @notice Validate that the user is not below the opening limit borrow CR
136:    /// @dev Reverts if the user is below the opening limit borrow CR
137:    ///      The user can set a custom opening limit borrow CR using SetUserConfiguration
138:    ///      If the user has not set a custom opening limit borrow CR, the default is the global opening limit borrow CR
139:    /// @param state The state	// @audit-issue: There should be single blank line between function declarations.
140:    function validateUserIsNotBelowOpeningLimitBorrowCR(State storage state, address account) external view {
```
[20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L15-L21), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L25-L31), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L35-L41), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L45-L53), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L64-L71), [88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L82-L89), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L96-L104), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L115-L121), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L123-L129), [139](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L133-L140), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

27:    using YieldCurveLibrary for YieldCurve;
28:
29:    /// @notice Check if the loan offer is null
30:    /// @param self The loan offer
31:    /// @return True if the loan offer is null, false otherwise	// @audit-issue: There should be single blank line between function declarations.
32:    function isNull(LoanOffer memory self) internal pure returns (bool) {

34:    }
35:
36:    /// @notice Check if the borrow offer is null
37:    /// @param self The borrow offer
38:    /// @return True if the borrow offer is null, false otherwise	// @audit-issue: There should be single blank line between function declarations.
39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {

41:    }
42:
43:    /// @notice Get the APR by tenor of a loan offer
44:    /// @param self The loan offer
45:    /// @param params The variable pool borrow rate params
46:    /// @param tenor The tenor
47:    /// @return The APR	// @audit-issue: There should be single blank line between function declarations.
48:    function getAPRByTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)

55:    }
56:
57:    /// @notice Get the absolute rate per tenor of a loan offer
58:    /// @param self The loan offer
59:    /// @param params The variable pool borrow rate params
60:    /// @param tenor The tenor
61:    /// @return The absolute rate	// @audit-issue: There should be single blank line between function declarations.
62:    function getRatePerTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)

69:    }
70:
71:    /// @notice Get the APR by tenor of a borrow offer
72:    /// @param self The borrow offer
73:    /// @param params The variable pool borrow rate params
74:    /// @param tenor The tenor
75:    /// @return The APR	// @audit-issue: There should be single blank line between function declarations.
76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)

83:    }
84:
85:    /// @notice Get the absolute rate per tenor of a borrow offer
86:    /// @param self The borrow offer
87:    /// @param params The variable pool borrow rate params
88:    /// @param tenor The tenor
89:    /// @return The absolute rate	// @audit-issue: There should be single blank line between function declarations.
90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L27-L32), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L34-L39), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L41-L48), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L55-L62), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L69-L76), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L83-L90), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

57:    using AccountingLibrary for State;
58:
59:    /// @notice Check if a positionId is a DebtPosition id
60:    /// @param state The state struct
61:    /// @param positionId The positionId
62:    /// @return True if the positionId is a DebtPosition id, false otherwise	// @audit-issue: There should be single blank line between function declarations.
63:    function isDebtPositionId(State storage state, uint256 positionId) internal view returns (bool) {

65:    }
66:
67:    /// @notice Check if a positionId is a CreditPosition id
68:    /// @param state The state struct
69:    /// @param positionId The positionId
70:    /// @return True if the positionId is a CreditPosition id, false otherwise	// @audit-issue: There should be single blank line between function declarations.
71:    function isCreditPositionId(State storage state, uint256 positionId) internal view returns (bool) {

73:    }
74:
75:    /// @notice Get a DebtPosition from a debtPositionId
76:    /// @dev Reverts if the debtPositionId is invalid
77:    /// @param state The state struct
78:    /// @param debtPositionId The debtPositionId
79:    /// @return The DebtPosition	// @audit-issue: There should be single blank line between function declarations.
80:    function getDebtPosition(State storage state, uint256 debtPositionId) public view returns (DebtPosition storage) {

86:    }
87:
88:    /// @notice Get a CreditPosition from a creditPositionId
89:    /// @dev Reverts if the creditPositionId is invalid
90:    /// @param state The state struct
91:    /// @param creditPositionId The creditPositionId
92:    /// @return The CreditPosition	// @audit-issue: There should be single blank line between function declarations.
93:    function getCreditPosition(State storage state, uint256 creditPositionId)

103:    }
104:
105:    /// @notice Get a DebtPosition from a CreditPosition id
106:    /// @param state The state struct
107:    /// @param creditPositionId The creditPositionId
108:    /// @return The DebtPosition	// @audit-issue: There should be single blank line between function declarations.
109:    function getDebtPositionByCreditPositionId(State storage state, uint256 creditPositionId)

116:    }
117:
118:    /// @notice Get the status of a loan
119:    /// @param state The state struct
120:    /// @param positionId The positionId (can be either a DebtPosition or a CreditPosition)
121:    /// @return The status of the loan	// @audit-issue: There should be single blank line between function declarations.
122:    function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {

140:    }
141:
142:    /// @notice Get the amount of collateral assigned to a DebtPosition
143:    ///         The amount of collateral assigned to a DebtPosition is the borrower's
144:    ///         collateral pro-rata to the DebtPosition's futureValue and the borrower's debt
145:    /// @param state The state struct
146:    /// @param debtPosition The DebtPosition
147:    /// @return The amount of collateral assigned to the DebtPosition	// @audit-issue: There should be single blank line between function declarations.
148:    function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)

161:    }
162:
163:    /// @notice Get the pro-rata collateral assigned to a CreditPosition
164:    ///         The amount of collateral assigned to a CreditPosition is the amount of collateral assigned to the
165:    ///         DebtPosition pro-rata to the CreditPosition's credit and the DebtPosition's futureValue
166:    /// @dev If the DebtPosition's futureValue is 0, the amount of collateral assigned to the CreditPosition is 0
167:    /// @param state The state struct
168:    /// @param creditPosition The CreditPosition
169:    /// @return The amount of collateral assigned to the CreditPosition	// @audit-issue: There should be single blank line between function declarations.
170:    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L57-L63), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L65-L71), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L73-L80), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L86-L93), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L103-L109), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L116-L122), [147](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L140-L148), [169](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L161-L170), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

40:    }
41:
42:    /// @notice Validate the yield curve
43:    /// @dev Reverts if the yield curve is invalid:
44:    ///      - The arrays are empty or have different lengths
45:    ///      - The tenors are not strictly increasing
46:    ///      - The tenors are out of range defined by minTenor and maxTenor
47:    /// @param self The yield curve
48:    /// @param minTenor The minimum tenor
49:    /// @param maxTenor The maximum tenor	// @audit-issue: There should be single blank line between function declarations.
50:    function validateYieldCurve(YieldCurve memory self, uint256 minTenor, uint256 maxTenor) internal pure {

78:    }
79:
80:    /// @notice Get the APR from the yield curve adjusted by the variable pool borrow rate
81:    /// @dev Reverts if the final result is negative
82:    ///      Only query the market borrow rate if the rate multiplier is not 0
83:    /// @param apr The annual percentage rate from the yield curve
84:    /// @param marketRateMultiplier The market rate multiplier
85:    /// @param params The variable pool borrow rate feed params
86:    /// @return Returns ratePerTenor + marketRate * marketRateMultiplier	// @audit-issue: There should be single blank line between function declarations.
87:    function getAdjustedAPR(int256 apr, uint256 marketRateMultiplier, VariablePoolBorrowRateParams memory params)

107:    }
108:
109:    /// @notice Get the rate from the yield curve by performing a linear interpolation between two time buckets
110:    /// @dev Reverts if the tenor is out of range
111:    /// @param curveRelativeTime The yield curve
112:    /// @param params The variable pool borrow rate feed params
113:    /// @param tenor The tenor
114:    /// @return The rate from the yield curve per given tenor	// @audit-issue: There should be single blank line between function declarations.
115:    function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)
```
[49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L40-L50), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L78-L87), [114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L107-L115), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

19:    using LoanLibrary for State;
20:
21:    /// @notice Converts debt token amount to a value in collateral tokens
22:    /// @dev Rounds up the debt token amount
23:    /// @param state The state object
24:    /// @param debtTokenAmount The amount of debt tokens
25:    /// @return collateralTokenAmount The amount of collateral tokens	// @audit-issue: There should be single blank line between function declarations.
26:    function debtTokenAmountToCollateralTokenAmount(State storage state, uint256 debtTokenAmount)

35:    }
36:
37:    /// @notice Repays a debt position
38:    /// @dev Upon repayment, the debt position future value and the borrower's total debt tracker are updated
39:    /// @param state The state object
40:    /// @param debtPositionId The debt position id
41:    /// @param repayAmount The amount to repay	// @audit-issue: There should be single blank line between function declarations.
42:    function repayDebt(State storage state, uint256 debtPositionId, uint256 repayAmount) public {

51:    }
52:
53:    /// @notice Creates a debt and credit position
54:    /// @dev Updates the borrower's total debt tracker.
55:    ///      The debt position future value and the credit position amount are created with the same value.
56:    /// @param state The state object
57:    /// @param lender The lender address
58:    /// @param borrower The borrower address
59:    /// @param futureValue The future value of the debt
60:    /// @param dueDate The due date of the debt
61:    /// @return creditPosition The created credit position	// @audit-issue: There should be single blank line between function declarations.
62:    function createDebtAndCreditPositions(

92:    }
93:
94:    /// @notice Creates a credit position by exiting an existing credit position
95:    /// @dev If the credit amount is the same, the existing credit position is updated with the new lender.
96:    ///      If the credit amount is different, the existing credit position is reduced and a new credit position is created.
97:    ///      The exit process can only be done with loans in the ACTIVE status.
98:    ///        It guarantees that the sum of credit positions keeps equal to the debt position future value.
99:    /// @param state The state object
100:    /// @param exitCreditPositionId The credit position id to exit
101:    /// @param lender The lender address
102:    /// @param credit The credit amount	// @audit-issue: There should be single blank line between function declarations.
103:    function createCreditPosition(State storage state, uint256 exitCreditPositionId, address lender, uint256 credit)

127:    }
128:
129:    /// @notice Reduces the credit amount of a credit position
130:    /// @dev The credit position is updated with the new credit amount.
131:    ///      The credit amount cannot be reduced below the minimum credit.
132:    ///      This operation breaks the initial sum of credit equal to the debt position future value.
133:    ///        If the loan is in REPAID status, this is expected, as lenders grdually claim their credit.
134:    ///        If the loan is in ACTIVE status, a debt reduction must be performed together with a credit reduction (See reduceDebtAndCredit).
135:    /// @param state The state object
136:    /// @param creditPositionId The credit position id	// @audit-issue: There should be single blank line between function declarations.
137:    function reduceCredit(State storage state, uint256 creditPositionId, uint256 amount) public {

145:    }
146:
147:    /// @notice Reduces the debt and credit amounts of a debt and credit position
148:    /// @dev The debt and credit positions are reduced with the same amount.
149:    /// @param state The state object
150:    /// @param debtPositionId The debt position id
151:    /// @param creditPositionId The credit position id
152:    /// @param amount The amount to reduce	// @audit-issue: There should be single blank line between function declarations.
153:    function reduceDebtAndCredit(State storage state, uint256 debtPositionId, uint256 creditPositionId, uint256 amount)

158:    }
159:
160:    /// @notice Get the swap fee percent for a given tenor
161:    /// @param state The state object
162:    /// @param tenor The tenor
163:    /// @return swapFeePercent The swap fee percent	// @audit-issue: There should be single blank line between function declarations.
164:    function getSwapFeePercent(State storage state, uint256 tenor) internal view returns (uint256) {

166:    }
167:
168:    /// @notice Get the swap fee for a given cash amount and tenor
169:    /// @param state The state object
170:    /// @param cash The cash amount
171:    /// @param tenor The tenor
172:    /// @return swapFee The swap fee	// @audit-issue: There should be single blank line between function declarations.
173:    function getSwapFee(State storage state, uint256 cash, uint256 tenor) internal view returns (uint256) {

175:    }
176:
177:    /// @notice Get the cash amount out for a given credit amount in
178:    /// @param state The state object
179:    /// @param creditAmountIn The credit amount in
180:    /// @param maxCredit The maximum credit
181:    /// @param ratePerTenor The rate per tenor
182:    /// @param tenor The tenor
183:    /// @return cashAmountOut The cash amount out
184:    /// @return fees The fees	// @audit-issue: There should be single blank line between function declarations.
185:    function getCashAmountOut(

217:    }
218:
219:    /// @notice Get the credit amount in for a given cash amount out
220:    /// @param state The state object
221:    /// @param cashAmountOut The cash amount out
222:    /// @param maxCredit The maximum credit
223:    /// @param maxCredit The maximum cash amount out
224:    /// @param ratePerTenor The rate per tenor
225:    /// @param tenor The tenor
226:    /// @return creditAmountIn The credit amount in
227:    /// @return fees The fees	// @audit-issue: There should be single blank line between function declarations.
228:    function getCreditAmountIn(

263:    }
264:
265:    /// @notice Get the credit amount out for a given cash amount in
266:    /// @param state The state object
267:    /// @param cashAmountIn The cash amount in
268:    /// @param maxCashAmountIn The maximum cash amount in
269:    /// @param maxCredit The maximum credit
270:    /// @param ratePerTenor The rate per tenor
271:    /// @param tenor The tenor
272:    /// @return creditAmountOut The credit amount out
273:    /// @return fees The fees	// @audit-issue: There should be single blank line between function declarations.
274:    function getCreditAmountOut(

301:    }
302:
303:    /// @notice Get the cash amount in for a given credit amount out
304:    /// @param state The state object
305:    /// @param creditAmountOut The credit amount out
306:    /// @param maxCredit The maximum credit
307:    /// @param ratePerTenor The rate per tenor
308:    /// @param tenor The tenor
309:    /// @return cashAmountIn The cash amount in
310:    /// @return fees The fees	// @audit-issue: There should be single blank line between function declarations.
311:    function getCashAmountIn(
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L19-L26), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L35-L42), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L51-L62), [102](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L92-L103), [136](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L127-L137), [152](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L145-L153), [163](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L158-L164), [172](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L166-L173), [184](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L175-L185), [227](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L217-L228), [273](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L263-L274), [310](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L301-L311), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

44:    }
45:
46:    /// @notice Validate that the borrow aToken supply is less than or equal to the borrow aToken cap
47:    ///         The cap is set in AToken amounts, which are rebasing by construction.
48:    ///         The admin should monitor the automatic supply increase and adjust the cap accordingly if necessary.
49:    /// @dev Reverts if the borrow aToken supply is greater than the borrow aToken cap
50:    ///      Due to rounding, the borrow aToken supply may be slightly less than the actual AToken supply, which is acceptable.
51:    /// @param state The state struct	// @audit-issue: There should be single blank line between function declarations.
52:    function validateBorrowATokenCap(State storage state) external view {

58:    }
59:
60:    /// @notice Validate that the Variable Pool has enough liquidity to withdraw the amount of cash
61:    /// @dev Reverts if the Variable Pool does not have enough liquidity
62:    ///      This safety mechanism prevents takers from matching orders that could not be withdrawn from the Variable Pool.
63:    ///        Nevertheless, the Variable Pool may still fail to withdraw the cash due to other factors (such as a pause, etc),
64:    ///        which is understood as an acceptable risk.
65:    /// @param state The state struct
66:    /// @param amount The amount of cash to withdraw	// @audit-issue: There should be single blank line between function declarations.
67:    function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L44-L52), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L58-L67), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

16:    using SafeERC20 for IERC20Metadata;
17:
18:    /// @notice Deposit underlying collateral token to the Size protocol
19:    /// @param state The state struct
20:    /// @param from The address from which the underlying collateral token is transferred
21:    /// @param to The address to which the Size deposit token is minted
22:    /// @param amount The amount of underlying collateral token to deposit	// @audit-issue: There should be single blank line between function declarations.
23:    function depositUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount) external {

27:    }
28:
29:    /// @notice Withdraw underlying collateral token from the Size protocol
30:    /// @param state The state struct
31:    /// @param from The address from which the Size deposit token is burned
32:    /// @param to The address to which the underlying collateral token is transferred
33:    /// @param amount The amount of underlying collateral token to withdraw	// @audit-issue: There should be single blank line between function declarations.
34:    function withdrawUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount)

40:    }
41:
42:    /// @notice Deposit underlying borrow token to the Size protocol
43:    /// @dev The underlying borrow token is deposited to the Variable Pool,
44:    ///        and the corresponding Size borrow token is minted in scaled amounts.
45:    /// @param state The state struct
46:    /// @param from The address from which the underlying borrow token is transferred
47:    /// @param to The address to which the Size borrow token is minted
48:    /// @param amount The amount of underlying borrow token to deposit	// @audit-issue: There should be single blank line between function declarations.
49:    function depositUnderlyingBorrowTokenToVariablePool(State storage state, address from, address to, uint256 amount)

65:    }
66:
67:    /// @notice Withdraw underlying borrow token from the Size protocol
68:    /// @dev The underlying borrow token is withdrawn from the Variable Pool,
69:    ///        and the corresponding Size borrow token is burned in scaled amounts.
70:    /// @param state The state struct
71:    /// @param from The address from which the Size borrow token is burned
72:    /// @param to The address to which the underlying borrow token is transferred
73:    /// @param amount The amount of underlying borrow token to withdraw	// @audit-issue: There should be single blank line between function declarations.
74:    function withdrawUnderlyingTokenFromVariablePool(State storage state, address from, address to, uint256 amount)
```
[22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L16-L23), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L27-L34), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L40-L49), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L65-L74), 


```solidity
Path: ./src/libraries/Multicall.sol

22:    using RiskLibrary for State;
23:
24:    /// @dev Receives and executes a batch of function calls on this contract.
25:    /// @custom:oz-upgrades-unsafe-allow-reachable delegatecall	// @audit-issue: There should be single blank line between function declarations.
26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L22-L26), 


```solidity
Path: ./src/libraries/Math.sol

33:    }
34:
35:    /// @notice Convert an APR to an absolute rate for a given tenor
36:    /// @dev The formula is `apr * tenor / YEAR`
37:    /// @param apr The APR to convert
38:    /// @param tenor The tenor
39:    /// @return The absolute rate	// @audit-issue: There should be single blank line between function declarations.
40:    function aprToRatePerTenor(uint256 apr, uint256 tenor) internal pure returns (uint256) {

42:    }
43:
44:    /// @notice Find the index of `value` in the sorted list `array`
45:    /// @dev If `value` is below the lowest value in `array` or above the highest value in `array`, the function returns (type(uint256).max, type(uint256).max)
46:    ///      Formally verified with Halmos (check_Math_binarySearch)
47:    /// @param array The sorted list to search
48:    /// @param value The value to search for
49:    /// @return low The index of the largest element in `array` that is less than or equal to `value`
50:    /// @return high The index of the smallest element in `array` that is greater than or equal to `value`	// @audit-issue: There should be single blank line between function declarations.
51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L33-L40), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L42-L51), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

26:    using LoanLibrary for State;
27:
28:    /// @notice Validates the input parameters for setting user configuration
29:    /// @param state The state
30:    /// @param params The input parameters for setting user configuration	// @audit-issue: There should be single blank line between function declarations.
31:    function validateSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params)

58:    }
59:
60:    /// @notice Executes the setting of user configuration
61:    /// @param state The state
62:    /// @param params The input parameters for setting user configuration	// @audit-issue: There should be single blank line between function declarations.
63:    function executeSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params) external {
```
[30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L26-L31), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L58-L63), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

29:    using RiskLibrary for State;
30:
31:    /// @notice Validates the input parameters for self-liquidating a credit position
32:    /// @param state The state
33:    /// @param params The input parameters for self-liquidating a credit position	// @audit-issue: There should be single blank line between function declarations.
34:    function validateSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external view {

54:    }
55:
56:    /// @notice Executes the self-liquidation of a credit position
57:    /// @param state The state
58:    /// @param params The input parameters for self-liquidating a credit position	// @audit-issue: There should be single blank line between function declarations.
59:    function executeSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external {
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L29-L34), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L54-L59), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

37:    using RiskLibrary for State;
38:
39:    /// @notice Validates the input parameters for compensating a credit position
40:    /// @param state The state
41:    /// @param params The input parameters for compensating a credit position	// @audit-issue: There should be single blank line between function declarations.
42:    function validateCompensate(State storage state, CompensateParams calldata params) external view {

101:    }
102:
103:    /// @notice Executes the compensating of a credit position
104:    /// @param state The state
105:    /// @param params The input parameters for compensating a credit position	// @audit-issue: There should be single blank line between function declarations.
106:    function executeCompensate(State storage state, CompensateParams calldata params) external {
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L37-L42), [105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L101-L106), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

46:    using AccountingLibrary for State;
47:
48:    /// @notice Validates the input parameters for selling credit as a market order
49:    /// @param state The state
50:    /// @param params The input parameters for selling credit as a market order	// @audit-issue: There should be single blank line between function declarations.
51:    function validateSellCreditMarket(State storage state, SellCreditMarketParams calldata params) external view {

122:    }
123:
124:    /// @notice Executes the selling of credit as a market order
125:    /// @param state The state
126:    /// @param params The input parameters for selling credit as a market order	// @audit-issue: There should be single blank line between function declarations.
127:    function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)
```
[50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L46-L51), [126](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L122-L127), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

37:    using Initialize for State;
38:
39:    /// @notice Returns the current fee configuration parameters
40:    /// @param state The state of the protocol
41:    /// @return The current fee configuration parameters	// @audit-issue: There should be single blank line between function declarations.
42:    function feeConfigParams(State storage state) public view returns (InitializeFeeConfigParams memory) {

51:    }
52:
53:    /// @notice Returns the current risk configuration parameters
54:    /// @param state The state of the protocol
55:    /// @return The current risk configuration parameters	// @audit-issue: There should be single blank line between function declarations.
56:    function riskConfigParams(State storage state) public view returns (InitializeRiskConfigParams memory) {

65:    }
66:
67:    /// @notice Returns the current oracle configuration parameters
68:    /// @param state The state of the protocol
69:    /// @return The current oracle configuration parameters	// @audit-issue: There should be single blank line between function declarations.
70:    function oracleParams(State storage state) public view returns (InitializeOracleParams memory) {

75:    }
76:
77:    /// @dev Validation is done at execution
78:    ///      We purposefuly leave this function empty for documentation purposes	// @audit-issue: There should be single blank line between function declarations.
79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {

81:    }
82:
83:    /// @notice Updates the configuration of the protocol
84:    /// @param state The state of the protocol
85:    /// @param params The parameters to update the configuration	// @audit-issue: There should be single blank line between function declarations.
86:    function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L37-L42), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L51-L56), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L65-L70), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L75-L79), [85](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L81-L86), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

33:    using AccountingLibrary for State;
34:
35:    /// @notice Validates the input parameters for liquidating a debt position
36:    /// @param state The state	// @audit-issue: There should be single blank line between function declarations.
37:    function validateLiquidate(State storage state, LiquidateParams calldata params) external view {

54:    }
55:
56:    /// @notice Validates the minimum profit in collateral tokens expected by the liquidator
57:    /// @param params The input parameters for liquidating a debt position
58:    /// @param liquidatorProfitCollateralToken The profit in collateral tokens expected by the liquidator	// @audit-issue: There should be single blank line between function declarations.
59:    function validateMinimumCollateralProfit(

69:    }
70:
71:    /// @notice Executes the liquidation of a debt position
72:    /// @param state The state
73:    /// @param params The input parameters for liquidating a debt position
74:    /// @return liquidatorProfitCollateralToken The profit in collateral tokens expected by the liquidator	// @audit-issue: There should be single blank line between function declarations.
75:    function executeLiquidate(State storage state, LiquidateParams calldata params)
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L33-L37), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L54-L59), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L69-L75), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

42:    using LoanLibrary for DebtPosition;
43:
44:    /// @notice Validates the input parameters for liquidating a debt position with a replacement borrower
45:    /// @param state The state
46:    /// @param params The input parameters for liquidating a debt position with a replacement borrower	// @audit-issue: There should be single blank line between function declarations.
47:    function validateLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)

93:    }
94:
95:    /// @notice Validates the minimum profit in collateral tokens expected by the liquidator
96:    /// @param state The state
97:    /// @param params The input parameters for liquidating a debt position with a replacement borrower
98:    /// @param liquidatorProfitCollateralToken The profit in collateral tokens expected by the liquidator	// @audit-issue: There should be single blank line between function declarations.
99:    function validateMinimumCollateralProfit(

112:    }
113:
114:    /// @notice Executes the liquidation of a debt position with a replacement borrower
115:    /// @param state The state
116:    /// @param params The input parameters for liquidating a debt position with a replacement borrower
117:    /// @return issuanceValue The issuance value
118:    /// @return liquidatorProfitCollateralToken The profit in collateral tokens expected by the liquidator
119:    /// @return liquidatorProfitBorrowToken The profit in borrow tokens expected by the liquidator	// @audit-issue: There should be single blank line between function declarations.
120:    function executeLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)
```
[46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L42-L47), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L93-L99), [119](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L112-L120), 


```solidity
Path: ./src/libraries/actions/Claim.sol

26:    using AccountingLibrary for State;
27:
28:    /// @notice Validates the input parameters for claiming a credit position
29:    /// @param state The state
30:    /// @param params The input parameters for claiming a credit position	// @audit-issue: There should be single blank line between function declarations.
31:    function validateClaim(State storage state, ClaimParams calldata params) external view {

43:    }
44:
45:    /// @notice Executes the claiming of a credit position
46:    /// @param state The state
47:    /// @param params The input parameters for claiming a credit position	// @audit-issue: There should be single blank line between function declarations.
48:    function executeClaim(State storage state, ClaimParams calldata params) external {
```
[30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L26-L31), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L43-L48), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

66:    }
67:
68:    /// @notice Validates the parameters for the fee configuration
69:    /// @param f The fee configuration parameters	// @audit-issue: There should be single blank line between function declarations.
70:    function validateInitializeFeeConfigParams(InitializeFeeConfigParams memory f) internal pure {

94:    }
95:
96:    /// @notice Validates the parameters for the risk configuration
97:    /// @param r The risk configuration parameters	// @audit-issue: There should be single blank line between function declarations.
98:    function validateInitializeRiskConfigParams(InitializeRiskConfigParams memory r) internal pure {

128:    }
129:
130:    /// @notice Validates the parameters for the oracle configuration
131:    /// @param o The oracle configuration parameters	// @audit-issue: There should be single blank line between function declarations.
132:    function validateInitializeOracleParams(InitializeOracleParams memory o) internal view {

142:    }
143:
144:    /// @notice Validates the parameters for the data configuration
145:    /// @param d The data configuration parameters	// @audit-issue: There should be single blank line between function declarations.
146:    function validateInitializeDataParams(InitializeDataParams memory d) internal view {

167:    }
168:
169:    /// @notice Validates the parameters for the initialization
170:    /// @param owner The owner address
171:    /// @param f The fee configuration parameters
172:    /// @param r The risk configuration parameters
173:    /// @param o The oracle configuration parameters
174:    /// @param d The data configuration parameters	// @audit-issue: There should be single blank line between function declarations.
175:    function validateInitialize(

188:    }
189:
190:    /// @notice Executes the initialization of the fee configuration
191:    /// @param state The state
192:    /// @param f The fee configuration parameters	// @audit-issue: There should be single blank line between function declarations.
193:    function executeInitializeFeeConfig(State storage state, InitializeFeeConfigParams memory f) internal {

202:    }
203:
204:    /// @notice Executes the initialization of the risk configuration
205:    /// @param state The state
206:    /// @param r The risk configuration parameters	// @audit-issue: There should be single blank line between function declarations.
207:    function executeInitializeRiskConfig(State storage state, InitializeRiskConfigParams memory r) internal {

217:    }
218:
219:    /// @notice Executes the initialization of the oracle configuration
220:    /// @param state The state
221:    /// @param o The oracle configuration parameters	// @audit-issue: There should be single blank line between function declarations.
222:    function executeInitializeOracle(State storage state, InitializeOracleParams memory o) internal {

225:    }
226:
227:    /// @notice Executes the initialization of the data configuration
228:    /// @param state The state
229:    /// @param d The data configuration parameters	// @audit-issue: There should be single blank line between function declarations.
230:    function executeInitializeData(State storage state, InitializeDataParams memory d) internal {

259:    }
260:
261:    /// @notice Executes the initialization of the protocol
262:    /// @param state The state
263:    /// @param f The fee configuration parameters
264:    /// @param r The risk configuration parameters
265:    /// @param o The oracle configuration parameters
266:    /// @param d The data configuration parameters	// @audit-issue: There should be single blank line between function declarations.
267:    function executeInitialize(
```
[69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L66-L70), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L94-L98), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L128-L132), [145](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L142-L146), [174](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L167-L175), [192](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L188-L193), [206](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L202-L207), [221](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L217-L222), [229](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L225-L230), [266](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L259-L267), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

20:    using OfferLibrary for BorrowOffer;
21:
22:    /// @notice Validates the input parameters for selling credit as a limit order
23:    /// @param state The state
24:    /// @param params The input parameters for selling credit as a limit order	// @audit-issue: There should be single blank line between function declarations.
25:    function validateSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external view {

38:    }
39:
40:    /// @notice Executes the selling of credit as a limit order
41:    /// @param state The state
42:    /// @param params The input parameters for selling credit as a limit order
43:    /// @dev A null offer means clearing a user's borrow limit order	// @audit-issue: There should be single blank line between function declarations.
44:    function executeSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external {
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L20-L25), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L38-L44), 


```solidity
Path: ./src/libraries/actions/Repay.sol

28:    using RiskLibrary for State;
29:
30:    /// @notice Validates the input parameters for repaying a debt position
31:    /// @param state The state
32:    /// @param params The input parameters for repaying a debt position	// @audit-issue: There should be single blank line between function declarations.
33:    function validateRepay(State storage state, RepayParams calldata params) external view {

41:    }
42:
43:    /// @notice Executes the repayment of a debt position
44:    /// @param state The state
45:    /// @param params The input parameters for repaying a debt position	// @audit-issue: There should be single blank line between function declarations.
46:    function executeRepay(State storage state, RepayParams calldata params) external {
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L28-L33), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L41-L46), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

24:    using OfferLibrary for LoanOffer;
25:
26:    /// @notice Validates the input parameters for buying credit as a limit order
27:    /// @param state The state
28:    /// @param params The input parameters for buying credit as a limit order	// @audit-issue: There should be single blank line between function declarations.
29:    function validateBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external view {

51:    }
52:
53:    /// @notice Executes the buying of credit as a limit order
54:    /// @param state The state
55:    /// @param params The input parameters for buying credit as a limit order
56:    /// @dev A null offer means clearing a user's loan limit order	// @audit-issue: There should be single blank line between function declarations.
57:    function executeBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external {
```
[28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L24-L29), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L51-L57), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

46:    using RiskLibrary for State;
47:
48:    /// @notice Validates the input parameters for buying credit as a market order
49:    /// @param state The state
50:    /// @param params The input parameters for buying credit as a market order	// @audit-issue: There should be single blank line between function declarations.
51:    function validateBuyCreditMarket(State storage state, BuyCreditMarketParams calldata params) external view {

115:    }
116:
117:    /// @notice Executes the buying of credit as a market order
118:    /// @param state The state
119:    /// @param params The input parameters for buying credit as a market order
120:    /// @return cashAmountIn The amount of cash paid for the credit	// @audit-issue: There should be single blank line between function declarations.
121:    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
```
[50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L46-L51), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L115-L121), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

15:    uint8 internal immutable _decimals;
16:
17:    // solhint-disable-next-line no-empty-blocks	// @audit-issue: There should be single blank line between function declarations.
18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)
```
[17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L15-L18), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

39:    }
40:
41:    /// @dev Reverts with NOT_SUPPORTED	// @audit-issue: There should be single blank line between function declarations.
42:    function mint(address, uint256) external view override onlyOwner {

44:    }
45:
46:    /// @notice Mint scaled tokens to an account
47:    /// @param to The account to mint the tokens to
48:    /// @param scaledAmount The scaled amount of tokens to mint
49:    /// @dev Emits a TransferUnscaled event representing the actual unscaled amount	// @audit-issue: There should be single blank line between function declarations.
50:    function mintScaled(address to, uint256 scaledAmount) external onlyOwner {

53:    }
54:
55:    /// @dev Reverts with NOT_SUPPORTED	// @audit-issue: There should be single blank line between function declarations.
56:    function burn(address, uint256) external view override onlyOwner {

58:    }
59:
60:    /// @notice Burn scaled tokens from an account
61:    /// @param from The account to burn the tokens from
62:    /// @param scaledAmount The scaled amount of tokens to burn
63:    /// @dev Emits a TransferUnscaled event representing the actual unscaled amount	// @audit-issue: There should be single blank line between function declarations.
64:    function burnScaled(address from, uint256 scaledAmount) external onlyOwner {

67:    }
68:
69:    /// @notice Transfer tokens from one account to another
70:    /// @param from The account to transfer the tokens from
71:    /// @param to The account to transfer the tokens to
72:    /// @param value The unscaled amount of tokens to transfer
73:    /// @dev Emits TransferUnscaled events representing the actual unscaled amount
74:    ///      Scales the amount by the current liquidity index before transferring scaled tokens
75:    /// @return True if the transfer was successful	// @audit-issue: There should be single blank line between function declarations.
76:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {

85:    }
86:
87:    /// @notice Returns the scaled balance of an account
88:    /// @param account The account to get the balance of
89:    /// @return The scaled balance of the account	// @audit-issue: There should be single blank line between function declarations.
90:    function scaledBalanceOf(address account) public view returns (uint256) {

92:    }
93:
94:    /// @notice Unscales a scaled amount
95:    /// @param scaledAmount The scaled amount to unscale
96:    /// @return The unscaled amount
97:    /// @dev The unscaled amount is the scaled amount divided by the current liquidity index	// @audit-issue: There should be single blank line between function declarations.
98:    function _unscale(uint256 scaledAmount) internal view returns (uint256) {

100:    }
101:
102:    /// @notice Returns the unscaled balance of an account
103:    /// @param account The account to get the balance of
104:    /// @return The unscaled balance of the account	// @audit-issue: There should be single blank line between function declarations.
105:    function balanceOf(address account) public view override returns (uint256) {

107:    }
108:
109:    /// @notice Returns the scaled total supply of the token
110:    /// @return The scaled total supply of the token	// @audit-issue: There should be single blank line between function declarations.
111:    function scaledTotalSupply() public view returns (uint256) {

113:    }
114:
115:    /// @notice Returns the unscaled total supply of the token
116:    /// @return The unscaled total supply of the token	// @audit-issue: There should be single blank line between function declarations.
117:    function totalSupply() public view override returns (uint256) {

119:    }
120:
121:    /// @notice Returns the current liquidity index of the variable pool
122:    /// @return The current liquidity index of the variable pool	// @audit-issue: There should be single blank line between function declarations.
123:    function liquidityIndex() public view returns (uint256) {
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L39-L42), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L44-L50), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L53-L56), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L58-L64), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L67-L76), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L85-L90), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L92-L98), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L100-L105), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L107-L111), [116](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L113-L117), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L119-L123), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/latest/style-guide.html#blank-lines).

### `constants` should be defined rather than using magic numbers
Even [assembly](https://github.com/code-423n4/2022-05-opensea-seaport/blob/9d7ce4d08bf3c3010304a0476a785c70c0e90ae7/contracts/lib/TokenTransferrer.sol#L35-L39) can benefit from using readable constants instead of hex/numeric literals

```solidity
Path: ./src/oracle/PriceFeed.sol

80:            _getPrice(base, baseStalePriceInterval), 10 ** decimals, _getPrice(quote, quoteStalePriceInterval)	// @audit-issue
```
[80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L80-L80), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

33:            debtTokenAmountWad, 10 ** state.oracle.priceFeed.decimals(), state.oracle.priceFeed.getPrice()	// @audit-issue
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L33-L33), 


```solidity
Path: ./src/libraries/Math.sol

32:        return amount * 10 ** (18 - decimals);	// @audit-issue
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L32-L32), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

151:        if (IERC20Metadata(d.underlyingCollateralToken).decimals() > 18) {	// @audit-issue

159:        if (IERC20Metadata(d.underlyingBorrowToken).decimals() > 18) {	// @audit-issue
```
[151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L151-L151), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L159-L159), 


#### Recommendation

Consider defining constants with meaningful names for magic numbers and hexadecimal literals to improve code readability and maintainability.

### Lack of index element validation in function
There's no validation to check whether the index element provided as an argument actually exists in the call. This omission could lead to unintended behavior if an element that does not exist in the call is passed to the function. The function should validate that the provided index element exists in the call before proceeding.

```solidity
Path: ./src/libraries/Multicall.sol

34:            results[i] = Address.functionDelegateCall(address(this), data[i]);	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L34-L34), 


```solidity
Path: ./src/libraries/Math.sol

54:        if (value < array[low] || value > array[high]) {	// @audit-issue

61:            } else if (array[mid] < value) {	// @audit-issue
```
[54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L54-L54), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L61-L61), 


#### Recommendation

Integrate explicit index validation checks at the beginning of functions that operate based on index elements. Use conditional statements to verify that the provided index falls within the valid range of existing elements. For array operations, ensure the index is less than the array's length. For mappings, consider additional logic to confirm the presence of a key. For example, in an array-based function:
```solidity
function getElementByIndex(uint256 index) public view returns (ElementType) {
    require(index < array.length, "Index out of bounds");
    return array[index];
}
```


### Contract should expose an `interface`
All `external`/`public` functions should extend an `interface`. This is useful to make sure that the whole API is extracted.


```solidity
Path: ./src/Size.sol

62:contract Size is ISize, SizeView, Initializable, AccessControlUpgradeable, PausableUpgradeable, UUPSUpgradeable {	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L62-L62), 


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

Consider defining an `interface` that includes all `external`/`public` functions of the contract. Exposing a well-defined interface helps ensure that the entire API is extracted and provides a clear and standardized way for other contracts or users to interact with your contract.

### Polymorphic functions make security audits more time-consuming and error-prone
The instances below point to one of two functions with the same name. Consider naming each function differently, in order to make code navigation and analysis easier.

```solidity
Path: ./src/libraries/OfferLibrary.sol

39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {	// @audit-issue same function also on line(s): 32

76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue same function also on line(s): 48

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue same function also on line(s): 62
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L39-L39), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L76-L76), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L90), 


#### Recommendation

Rename one or both of the polymorphic functions to have distinct names, improving code readability and making security audits more efficient and less error-prone. Clear and unique function names help prevent confusion and ensure that the intended function is called.

### Consider using named returns
Using named returns makes the code more self-documenting, makes it easier to fill out NatSpec, and in some cases can save gas. The cases below are where there currently is at most one return statement, which is ideal for named returns.

```solidity
Path: ./src/SizeView.sol

48:    function collateralRatio(address user) external view returns (uint256) {	// @audit-issue

53:    function isUserUnderwater(address user) external view returns (bool) {	// @audit-issue

58:    function isDebtPositionLiquidatable(uint256 debtPositionId) external view returns (bool) {	// @audit-issue

63:    function debtTokenAmountToCollateralTokenAmount(uint256 borrowATokenAmount) external view returns (uint256) {	// @audit-issue

68:    function feeConfig() external view returns (InitializeFeeConfigParams memory) {	// @audit-issue

73:    function riskConfig() external view returns (InitializeRiskConfigParams memory) {	// @audit-issue

78:    function oracle() external view returns (InitializeOracleParams memory) {	// @audit-issue

83:    function data() external view returns (DataView memory) {	// @audit-issue

97:    function getUserView(address user) external view returns (UserView memory) {	// @audit-issue

108:    function isDebtPositionId(uint256 debtPositionId) external view returns (bool) {	// @audit-issue

113:    function isCreditPositionId(uint256 creditPositionId) external view returns (bool) {	// @audit-issue

118:    function getDebtPosition(uint256 debtPositionId) external view returns (DebtPosition memory) {	// @audit-issue

123:    function getCreditPosition(uint256 creditPositionId) external view returns (CreditPosition memory) {	// @audit-issue

128:    function getLoanStatus(uint256 positionId) external view returns (LoanStatus) {	// @audit-issue

133:    function getPositionsCount() external view returns (uint256, uint256) {	// @audit-issue

141:    function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {	// @audit-issue

157:    function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {	// @audit-issue

173:    function getDebtPositionAssignedCollateral(uint256 debtPositionId) external view returns (uint256) {	// @audit-issue

179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L48-L48), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L53-L53), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L58-L58), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L63-L63), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L68-L68), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L73-L73), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L78-L78), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L83-L83), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L97-L97), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L108-L108), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L113-L113), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L118-L118), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L123-L123), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L128-L128), [133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L133-L133), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L141-L141), [157](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L157-L157), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L173-L173), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L179-L179), 


```solidity
Path: ./src/oracle/IPriceFeed.sol

9:    function getPrice() external view returns (uint256);	// @audit-issue

11:    function decimals() external view returns (uint256);	// @audit-issue
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L11-L11), 


```solidity
Path: ./src/oracle/PriceFeed.sol

63:    function getPrice() external view returns (uint256) {	// @audit-issue

84:    function _getPrice(AggregatorV3Interface aggregator, uint256 stalePriceInterval) internal view returns (uint256) {	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L63-L63), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L84-L84), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

53:    function collateralRatio(State storage state, address account) public view returns (uint256) {	// @audit-issue

71:    function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)
72:        public
73:        view
74:        returns (bool)	// @audit-issue

89:    function isCreditPositionTransferrable(State storage state, uint256 creditPositionId)
90:        internal
91:        view
92:        returns (bool)	// @audit-issue

104:    function isDebtPositionLiquidatable(State storage state, uint256 debtPositionId) public view returns (bool) {	// @audit-issue

121:    function isUserUnderwater(State storage state, address account) public view returns (bool) {	// @audit-issue
```
[53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L53-L53), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L71-L74), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L89-L92), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L104-L104), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L121-L121), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

32:    function isNull(LoanOffer memory self) internal pure returns (bool) {	// @audit-issue

39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {	// @audit-issue

48:    function getAPRByTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
49:        internal
50:        view
51:        returns (uint256)	// @audit-issue

62:    function getRatePerTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
63:        internal
64:        view
65:        returns (uint256)	// @audit-issue

76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
77:        internal
78:        view
79:        returns (uint256)	// @audit-issue

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
91:        internal
92:        view
93:        returns (uint256)	// @audit-issue
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L32-L32), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L39-L39), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L48-L51), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L62-L65), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L76-L79), [93](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L93), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

63:    function isDebtPositionId(State storage state, uint256 positionId) internal view returns (bool) {	// @audit-issue

71:    function isCreditPositionId(State storage state, uint256 positionId) internal view returns (bool) {	// @audit-issue

80:    function getDebtPosition(State storage state, uint256 debtPositionId) public view returns (DebtPosition storage) {	// @audit-issue

93:    function getCreditPosition(State storage state, uint256 creditPositionId)
94:        public
95:        view
96:        returns (CreditPosition storage)	// @audit-issue

109:    function getDebtPositionByCreditPositionId(State storage state, uint256 creditPositionId)
110:        public
111:        view
112:        returns (DebtPosition storage)	// @audit-issue

122:    function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {	// @audit-issue

148:    function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)
149:        public
150:        view
151:        returns (uint256)	// @audit-issue

170:    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)
171:        public
172:        view
173:        returns (uint256)	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L63-L63), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L71-L71), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L80-L80), [96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L93-L96), [112](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L109-L112), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L122-L122), [151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L148-L151), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L170-L173), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

38:    function isNull(YieldCurve memory self) internal pure returns (bool) {	// @audit-issue

87:    function getAdjustedAPR(int256 apr, uint256 marketRateMultiplier, VariablePoolBorrowRateParams memory params)
88:        internal
89:        view
90:        returns (uint256)	// @audit-issue

115:    function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)
116:        external
117:        view
118:        returns (uint256)	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L38-L38), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L87-L90), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L115-L118), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

164:    function getSwapFeePercent(State storage state, uint256 tenor) internal view returns (uint256) {	// @audit-issue

173:    function getSwapFee(State storage state, uint256 cash, uint256 tenor) internal view returns (uint256) {	// @audit-issue
```
[164](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L164-L164), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L173-L173), 


```solidity
Path: ./src/libraries/Math.sol

15:    function min(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue

23:    function mulDivUp(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue

27:    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue

40:    function aprToRatePerTenor(uint256 apr, uint256 tenor) internal pure returns (uint256) {	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L15-L15), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L23-L23), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L27-L27), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

42:    function feeConfigParams(State storage state) public view returns (InitializeFeeConfigParams memory) {	// @audit-issue

56:    function riskConfigParams(State storage state) public view returns (InitializeRiskConfigParams memory) {	// @audit-issue

70:    function oracleParams(State storage state) public view returns (InitializeOracleParams memory) {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L56-L56), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L70-L70), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

76:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

90:    function scaledBalanceOf(address account) public view returns (uint256) {	// @audit-issue

98:    function _unscale(uint256 scaledAmount) internal view returns (uint256) {	// @audit-issue

105:    function balanceOf(address account) public view override returns (uint256) {	// @audit-issue

111:    function scaledTotalSupply() public view returns (uint256) {	// @audit-issue

117:    function totalSupply() public view override returns (uint256) {	// @audit-issue

123:    function liquidityIndex() public view returns (uint256) {	// @audit-issue
```
[76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L76-L76), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L90-L90), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L98-L98), [105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L105-L105), [111](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L111-L111), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L117-L117), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L123-L123), 


#### Recommendation

Adopt named returns in your Solidity functions, especially in cases where functions contain a single return statement. This approach enhances code readability and documentation by making the return values clear and explicit. When defining your function, specify the return types with names, and manipulate these named variables directly within your function logic. Additionally, leverage named returns to streamline your NatSpec documentation, providing clear descriptions for each return variable. Evaluate your current contracts for opportunities to refactor functions to use named returns, prioritizing those with simple return patterns for immediate benefits in gas efficiency and code clarity.

### Missing events in initializers/deploys
As a best practice, consider emitting an event when the contract is initialized. In this way, it's easy for the user to track the exact point in time when the contract was initialized, by filtering the emitted events.

```solidity
Path: ./src/Size.sol

87:    function initialize(	// @audit-issue
```
[87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L87), 


#### Recommendation

To provide transparency and enable users to track the initialization of the contract, consider emitting an event within the contract's initializer function. Emitting an event during initialization can help users pinpoint the exact moment the contract was initialized by filtering and monitoring the emitted events.

### Inefficient Array Usage
Use mappings instead of arrays for managing lists of data in order to conserve gas. Mappings are less expensive and more efficient for accessing any value without having to iterate through an array.

```solidity
Path: ./src/Size.sol

142:    function multicall(bytes[] calldata _data)	// @audit-issue
```
[142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L142), 


```solidity
Path: ./src/libraries/Multicall.sol

26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L26-L26), 


```solidity
Path: ./src/libraries/Math.sol

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L51), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

10:    uint256[] tenors;	// @audit-issue

12:    int256[] aprs;	// @audit-issue

14:    uint256[] marketRateMultipliers;	// @audit-issue
```
[10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L10-L10), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L12-L12), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L14-L14), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

19:    uint256[] creditPositionIds;	// @audit-issue
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L19-L19), 


#### Recommendation

In scenarios where data access efficiency is critical, prefer using mappings over arrays in Solidity contracts. Mappings offer more efficient and gas-effective data retrieval and updates, especially when dealing with large or frequently accessed datasets. Ensure to structure your data and choose keys thoughtfully to maximize the efficiency gains offered by mappings. While arrays might be suitable for ordered data or when the entire dataset needs to be iterated, for most other use cases, mappings are likely to be the more gas-efficient choice.

### Enum values should be used in place of constant array indexes
Consider using an `enum` instead of hardcoding an index access to make the code easier to understand.

```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

69:        if (self.tenors[0] < minTenor) {	// @audit-issue

70:            revert Errors.TENOR_OUT_OF_RANGE(self.tenors[0], minTenor, maxTenor);	// @audit-issue

121:        if (tenor < curveRelativeTime.tenors[0] || tenor > curveRelativeTime.tenors[length - 1]) {	// @audit-issue

122:            revert Errors.TENOR_OUT_OF_RANGE(tenor, curveRelativeTime.tenors[0], curveRelativeTime.tenors[length - 1]);	// @audit-issue
```
[69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L69-L69), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L70-L70), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L121-L121), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L122-L122), 


#### Recommendation

To improve code readability and maintainability, replace hardcoded array indexes with corresponding enum values. Enum values provide descriptive names for array elements, making your code more self-explanatory and reducing the risk of errors when working with arrays. This enhances the overall clarity and robustness of your smart contract code.

### Returning a struct instead of returning many variables is better
If a function returns [too many variables](https://docs.soliditylang.org/en/latest/contracts.html#returning-multiple-values), replacing them with a struct can improve code readability, maintainability and reusability.

```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

120:    function executeLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)
121:        external
122:        returns (uint256 issuanceValue, uint256 liquidatorProfitCollateralToken, uint256 liquidatorProfitBorrowToken)	// @audit-issue
```
[122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L120-L122), 


#### Recommendation

Consider returning a struct instead of multiple variables when a function returns many variables. This can enhance code readability, maintainability, and reusability, as well as make the contract interface more user-friendly.

### Consider using a `struct` rather than having many function input parameters
In Solidity, functions with a large number of input parameters can become cumbersome to manage and call. This can lead to readability issues and increased likelihood of errors, especially if the order of parameters is complex or not intuitive. To streamline this, consider consolidating multiple parameters into a single `struct`. This approach not only simplifies function signatures but also enhances code clarity. Structs allow for grouping related data together, making it easier to understand the relationship between parameters and manage them as a single entity.

```solidity
Path: ./src/Size.sol

87:    function initialize(
88:        address owner,
89:        InitializeFeeConfigParams calldata f,
90:        InitializeRiskConfigParams calldata r,
91:        InitializeOracleParams calldata o,
92:        InitializeDataParams calldata d
93:    ) external initializer {
```
[105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L93), 


```solidity
Path: ./src/oracle/PriceFeed.sol

36:    constructor(
37:        address _base,
38:        address _quote,
39:        address _sequencerUptimeFeed,
40:        uint256 _baseStalePriceInterval,
41:        uint256 _quoteStalePriceInterval
42:    ) {
```
[61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L36-L42), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

62:    function createDebtAndCreditPositions(
63:        State storage state,
64:        address lender,
65:        address borrower,
66:        uint256 futureValue,
67:        uint256 dueDate
68:    ) external returns (CreditPosition memory creditPosition) {

185:    function getCashAmountOut(
186:        State storage state,
187:        uint256 creditAmountIn,
188:        uint256 maxCredit,
189:        uint256 ratePerTenor,
190:        uint256 tenor
191:    ) internal view returns (uint256 cashAmountOut, uint256 fees) {

228:    function getCreditAmountIn(
229:        State storage state,
230:        uint256 cashAmountOut,
231:        uint256 maxCashAmountOut,
232:        uint256 maxCredit,
233:        uint256 ratePerTenor,
234:        uint256 tenor
235:    ) internal view returns (uint256 creditAmountIn, uint256 fees) {

274:    function getCreditAmountOut(
275:        State storage state,
276:        uint256 cashAmountIn,
277:        uint256 maxCashAmountIn,
278:        uint256 maxCredit,
279:        uint256 ratePerTenor,
280:        uint256 tenor
281:    ) internal view returns (uint256 creditAmountOut, uint256 fees) {

311:    function getCashAmountIn(
312:        State storage state,
313:        uint256 creditAmountOut,
314:        uint256 maxCredit,
315:        uint256 ratePerTenor,
316:        uint256 tenor
317:    ) internal view returns (uint256 cashAmountIn, uint256 fees) {
```
[92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L62-L68), [217](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L185-L191), [263](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L228-L235), [301](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L274-L281), [337](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L311-L317), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

19:    function validateBorrowATokenIncreaseLteDebtTokenDecrease(
20:        State storage state,
21:        uint256 borrowATokenSupplyBefore,
22:        uint256 debtTokenSupplyBefore,
23:        uint256 borrowATokenSupplyAfter,
24:        uint256 debtTokenSupplyAfter
25:    ) external view {
```
[44](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L19-L25), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

175:    function validateInitialize(
176:        State storage,
177:        address owner,
178:        InitializeFeeConfigParams memory f,
179:        InitializeRiskConfigParams memory r,
180:        InitializeOracleParams memory o,
181:        InitializeDataParams memory d
182:    ) external view {

267:    function executeInitialize(
268:        State storage state,
269:        InitializeFeeConfigParams memory f,
270:        InitializeRiskConfigParams memory r,
271:        InitializeOracleParams memory o,
272:        InitializeDataParams memory d
273:    ) external {
```
[188](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L182), [279](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L273), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

25:    constructor(
26:        IPool variablePool_,
27:        IERC20Metadata underlyingToken_,
28:        address owner_,
29:        string memory name_,
30:        string memory symbol_,
31:        uint8 decimals_
32:    ) NonTransferrableToken(owner_, name_, symbol_, decimals_) {
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L32), 


#### Recommendation

When faced with functions having numerous input parameters, refactor them to accept a `struct` instead. Define a `struct` that encapsulates all these parameters, thereby simplifying the function signature and improving code readability and maintainability. This method is particularly effective in complex functions or those with parameters that are logically related, making the code more intuitive and less error-prone.

### Unnecessary Use of override Keyword
In Solidity version 0.8.8 and later, the use of the override keyword becomes superfluous when a function is overriding solely from an interface and the function isn't present in multiple base contracts. Previously, the override keyword was required as an explicit indication to the compiler. However, this is no longer the case, and the extraneous use of the keyword can make the code less clean and more verbose.
Solidity documentation on [Function Overriding](https://docs.soliditylang.org/en/v0.8.20/contracts.html#function-overriding).


```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue
111:        external
112:        override(ISizeAdmin)
113:        onlyRole(DEFAULT_ADMIN_ROLE)
114:    {

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue
121:        external
122:        override(ISizeAdmin)
123:        onlyRole(BORROW_RATE_UPDATER_ROLE)
124:    {

132:    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue

137:    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue

142:    function multicall(bytes[] calldata _data)	// @audit-issue
143:        public
144:        payable
145:        override(IMulticall)
146:        whenNotPaused
147:        returns (bytes[] memory results)
148:    {

153:    function deposit(DepositParams calldata params) public payable override(ISize) whenNotPaused {	// @audit-issue

159:    function withdraw(WithdrawParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

166:    function buyCreditLimit(BuyCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

172:    function sellCreditLimit(SellCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

178:    function buyCreditMarket(BuyCreditMarketParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

188:    function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {	// @audit-issue

198:    function repay(RepayParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

204:    function claim(ClaimParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

210:    function liquidate(LiquidateParams calldata params)	// @audit-issue
211:        external
212:        payable
213:        override(ISize)
214:        whenNotPaused
215:        returns (uint256 liquidatorProfitCollateralToken)
216:    {

223:    function selfLiquidate(SelfLiquidateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)	// @audit-issue
230:        external
231:        payable
232:        override(ISize)
233:        whenNotPaused
234:        onlyRole(KEEPER_ROLE)
235:        returns (uint256 liquidatorProfitCollateralToken, uint256 liquidatorProfitBorrowToken)
236:    {

247:    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue

254:    function setUserConfiguration(SetUserConfigurationParams calldata params)	// @audit-issue
255:        external
256:        payable
257:        override(ISize)
258:        whenNotPaused
259:    {
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L114), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L124), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L132-L132), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L137-L137), [142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L148), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L153-L153), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L159-L159), [166](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L166-L166), [172](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L172-L172), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L178-L178), [188](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L188-L188), [198](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L198-L198), [204](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L204-L204), [210](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L210-L216), [223](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L223-L223), [229](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L236), [247](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L247-L247), [254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L254-L259), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue

76:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue

105:    function balanceOf(address account) public view override returns (uint256) {	// @audit-issue

117:    function totalSupply() public view override returns (uint256) {	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L76-L76), [105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L105-L105), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L117-L117), 


#### Recommendation

In Solidity versions 0.8.8 and later, the `override` keyword is no longer required for functions that are solely overriding from an interface and not present in multiple base contracts. Removing the unnecessary `override` keyword can make the code cleaner and less verbose.

### Include sender information in events
When an action is triggered based on a user's action, not being able to filter based on who triggered the action makes event processing a lot more cumbersome. Including the `msg.sender` the events of these types of action will make events much more useful to end users, especially when `msg.sender` is not `tx.origin`.

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

To improve the usability and analysis of your smart contract events, consider including the `msg.sender` address as part of the event data. This enables easier filtering and identification of the sender's actions within your contract, providing valuable insights for users and external tools.

### Unnecessary Constant Variable in Function Parameters
Passing a constant variable as a function parameter is redundant because the function can access the constant directly.

```solidity
Path: ./src/libraries/Math.sol

41:        return mulDivDown(apr, tenor, YEAR);	// @audit-issue
```
[41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L41-L41), 


#### Recommendation


        Reference constant variables directly within the function body instead of passing them as parameters. This simplifies the function signature and conserves gas.


### Control structures do not follow the Solidity Style Guide
Refer to the [Solidity style guide - Control Structures](https://docs.soliditylang.org/en/v0.8.20/style-guide.html#control-structures).

```solidity
Path: ./src/Size.sol

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue

142:    function multicall(bytes[] calldata _data)
143:        public
144:        payable
145:        override(IMulticall)
146:        whenNotPaused
147:        returns (bytes[] memory results)	// @audit-issue

210:    function liquidate(LiquidateParams calldata params)
211:        external
212:        payable
213:        override(ISize)
214:        whenNotPaused
215:        returns (uint256 liquidatorProfitCollateralToken)	// @audit-issue

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)
230:        external
231:        payable
232:        override(ISize)
233:        whenNotPaused
234:        onlyRole(KEEPER_ROLE)
235:        returns (uint256 liquidatorProfitCollateralToken, uint256 liquidatorProfitBorrowToken)	// @audit-issue

254:    function setUserConfiguration(SetUserConfigurationParams calldata params)	// @audit-issue
```
[110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L110), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L120), [147](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L147), [215](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L210-L215), [235](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L235), [254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L254-L254), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

71:    function isCreditPositionSelfLiquidatable(State storage state, uint256 creditPositionId)
72:        public
73:        view
74:        returns (bool)	// @audit-issue

89:    function isCreditPositionTransferrable(State storage state, uint256 creditPositionId)
90:        internal
91:        view
92:        returns (bool)	// @audit-issue
```
[74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L71-L74), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L89-L92), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

48:    function getAPRByTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
49:        internal
50:        view
51:        returns (uint256)	// @audit-issue

62:    function getRatePerTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
63:        internal
64:        view
65:        returns (uint256)	// @audit-issue

76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
77:        internal
78:        view
79:        returns (uint256)	// @audit-issue

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)
91:        internal
92:        view
93:        returns (uint256)	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L48-L51), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L62-L65), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L76-L79), [93](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L93), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

93:    function getCreditPosition(State storage state, uint256 creditPositionId)
94:        public
95:        view
96:        returns (CreditPosition storage)	// @audit-issue

109:    function getDebtPositionByCreditPositionId(State storage state, uint256 creditPositionId)
110:        public
111:        view
112:        returns (DebtPosition storage)	// @audit-issue

148:    function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)
149:        public
150:        view
151:        returns (uint256)	// @audit-issue

170:    function getCreditPositionProRataAssignedCollateral(State storage state, CreditPosition memory creditPosition)
171:        public
172:        view
173:        returns (uint256)	// @audit-issue
```
[96](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L93-L96), [112](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L109-L112), [151](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L148-L151), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L170-L173), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

87:    function getAdjustedAPR(int256 apr, uint256 marketRateMultiplier, VariablePoolBorrowRateParams memory params)
88:        internal
89:        view
90:        returns (uint256)	// @audit-issue

94:        } else if (
95:            params.variablePoolBorrowRateStaleRateInterval == 0
96:                || (
97:                    block.timestamp - params.variablePoolBorrowRateUpdatedAt
98:                        > params.variablePoolBorrowRateStaleRateInterval
99:                )	// @audit-issue

115:    function getAPR(YieldCurve memory curveRelativeTime, VariablePoolBorrowRateParams memory params, uint256 tenor)
116:        external
117:        view
118:        returns (uint256)	// @audit-issue
```
[90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L87-L90), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L94-L99), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L115-L118), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

26:    function debtTokenAmountToCollateralTokenAmount(State storage state, uint256 debtTokenAmount)
27:        internal
28:        view
29:        returns (uint256 collateralTokenAmount)	// @audit-issue

103:    function createCreditPosition(State storage state, uint256 exitCreditPositionId, address lender, uint256 credit)	// @audit-issue

153:    function reduceDebtAndCredit(State storage state, uint256 debtPositionId, uint256 creditPositionId, uint256 amount)	// @audit-issue
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L26-L29), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L103-L103), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L153-L153), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

34:    function withdrawUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount)	// @audit-issue

49:    function depositUnderlyingBorrowTokenToVariablePool(State storage state, address from, address to, uint256 amount)	// @audit-issue

74:    function withdrawUnderlyingTokenFromVariablePool(State storage state, address from, address to, uint256 amount)	// @audit-issue
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L34-L34), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L49-L49), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L74-L74), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

31:    function validateSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params)	// @audit-issue
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

75:            if (
76:                debtPositionToRepay.dueDate
77:                    < state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId).dueDate	// @audit-issue
```
[77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L75-L77), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

127:    function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)
128:        external
129:        returns (uint256 cashAmountOut)	// @audit-issue
```
[129](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L127-L129), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

99:            if (
100:                state.feeConfig.swapFeeAPR != 0
101:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)	// @audit-issue

109:            if (
110:                state.feeConfig.swapFeeAPR != 0
111:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)	// @audit-issue
```
[101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L99-L101), [111](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L109-L111), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

75:    function executeLiquidate(State storage state, LiquidateParams calldata params)
76:        external
77:        returns (uint256 liquidatorProfitCollateralToken)	// @audit-issue
```
[77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L75-L77), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

47:    function validateLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)	// @audit-issue

120:    function executeLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)
121:        external
122:        returns (uint256 issuanceValue, uint256 liquidatorProfitCollateralToken, uint256 liquidatorProfitBorrowToken)	// @audit-issue
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L47-L47), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L120-L122), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

34:        if (
35:            params.token != address(state.data.underlyingCollateralToken)
36:                && params.token != address(state.data.underlyingBorrowToken)	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L34-L36), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

121:    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
122:        external
123:        returns (uint256 cashAmountIn)	// @audit-issue
```
[123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L121-L123), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

46:        if (
47:            params.token != address(state.data.underlyingCollateralToken)
48:                && params.token != address(state.data.underlyingBorrowToken)	// @audit-issue
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L46-L48), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), 


#### Recommendation

Adhere to the Solidity style guide regarding control structures by avoiding the definition of multiple functions with identical names in a contract. Unique and descriptive function names improve code clarity and prevent potential confusion or errors. Consult [Solidity style guide - Control Structures](https://docs.soliditylang.org/en/v0.8.20/style-guide.html#control-structures) for best practices.

### Add inline comments for unnamed variables
In Solidity, it's not uncommon to encounter functions with unnamed parameters, especially when certain arguments are not used within the function body. While this is syntactically valid, it can lead to confusion and a lack of clarity about the function's intent and design. For better readability and maintainability, it's beneficial to include inline comments for these unnamed parameters. This practice provides context to other developers or auditors, clarifying the purpose or the reason for the exclusion of these parameters. For instance, transforming `function foo(address x, address)` to `function foo(address x, address /* unused */)`` or `function foo(address x, address /* y */)``. This small change can significantly enhance the understandability of the contract.

```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue: Need inline comments for unnamed parameter.
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L79), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

59:    function validateMinimumCollateralProfit(
60:        State storage,	// @audit-issue: Need inline comments for unnamed parameter.
61:        LiquidateParams calldata params,
62:        uint256 liquidatorProfitCollateralToken
63:    ) external pure {
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L59-L63), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

175:    function validateInitialize(
176:        State storage,	// @audit-issue: Need inline comments for unnamed parameter.
177:        address owner,
178:        InitializeFeeConfigParams memory f,
179:        InitializeRiskConfigParams memory r,
180:        InitializeOracleParams memory o,
181:        InitializeDataParams memory d
182:    ) external view {
```
[176](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L182), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue: Need inline comments for unnamed parameter.

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue: Need inline comments for unnamed parameter.
```
[46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue: Need inline comments for unnamed parameter.

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue: Need inline comments for unnamed parameter.
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), 


#### Recommendation

Incorporate inline comments for unnamed parameters in Solidity function definitions. This enhances readability and provides clarity on the function's design. For example, change `function foo(address x, address)` to `function foo(address x, address /* unused */)` or `function foo(address x, address /* y */)` to clearly indicate the purpose or the intentional omission of these parameters.

### Consider making contracts `Upgradeable`
This allows for bugs to be fixed in production, at the expense of significantly increasing centralization.

```solidity
Path: ./src/oracle/PriceFeed.sol

24:contract PriceFeed is IPriceFeed {	// @audit-issue
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L24-L24), 


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

Assess the need for upgradeability in your Solidity contracts based on the project's requirements and lifecycle. If chosen, implement a well-known proxy pattern ensuring rigorous security and governance mechanisms are in place. Be aware of the increased centralization and plan accordingly to mitigate potential risks, such as through decentralized governance models or multi-sig control for upgrade decisions.

### Events that mark critical parameter changes should contain both the old and the new value
This should especially be done if the new value is not required to be different from the old value

```solidity
Path: ./src/Size.sol

128:        emit Events.VariablePoolBorrowRateUpdated(oldBorrowRate, borrowRate);	// @audit-issue
```
[128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L128-L128), 


#### Recommendation

To enhance transparency and auditability, ensure that events are emitted when sensitive changes are made to the contracts. Review and update functions that lack event emissions, especially in cases where sensitive operations or state changes occur, to provide a clear record of such actions.

### Consider using descriptive `constants` when passing zero as a function argument
Passing zero as a function argument can sometimes result in a security issue (e.g. passing zero as the slippage parameter). Consider using a `constant` variable with a descriptive name, so it's clear that the argument is intentionally being used, and for the right reasons.

```solidity
Path: ./src/libraries/AccountingLibrary.sol

70:            DebtPosition({borrower: borrower, futureValue: futureValue, dueDate: dueDate, liquidityIndexAtRepayment: 0});	// @audit-issue
```
[70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L70-L70), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

60:        state.data.variablePool.supply(address(state.data.underlyingBorrowToken), amount, address(this), 0);	// @audit-issue
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L60-L60), 


#### Recommendation

Replace direct usage of zero as a function argument with a well-named `constant`. For example, use something like `uint256 constant NO_SLIPPAGE = 0;` when zero is intended to signify 'no slippage'. This approach enhances code readability, reduces ambiguity, and helps ensure that the function is used correctly and for its intended purpose.

### Don't define functions with the same name in a contract
In Solidity, while function overriding allows for functions with the same name to coexist, it is advisable to avoid this practice to enhance code readability and maintainability. Having multiple functions with the same name, even with different parameters or in inherited contracts, can cause confusion and increase the likelihood of errors during development, testing, and debugging. Using distinct and descriptive function names not only clarifies the purpose and behavior of each function, but also helps prevent unintended function calls or incorrect overriding. By adopting a clear and consistent naming convention, developers can create more comprehensible and maintainable smart contracts.

```solidity
Path: ./src/libraries/OfferLibrary.sol

39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {	// @audit-issue: Different function with name name found on line: 32

76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue: Different function with name name found on line: 48

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue: Different function with name name found on line: 62
```
[39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L39-L39), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L76-L76), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L90), 


#### Recommendation

Avoid defining multiple functions with the same name within a Solidity contract, including inherited contracts. Use distinct and descriptive names for each function to enhance readability, prevent confusion, and reduce the risk of errors in development and usage.

### Non-`external`/`public` function names should begin with an underscore
According to the Solidity Style Guide, Non-external/public function names should begin with an [underscore](https://docs.soliditylang.org/en/latest/style-guide.html#underscore-prefix-for-non-external-functions-and-variables)


```solidity
Path: ./src/libraries/RiskLibrary.sol

89:    function isCreditPositionTransferrable(State storage state, uint256 creditPositionId)	// @audit-issue
```
[89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L89-L89), 


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

63:    function isDebtPositionId(State storage state, uint256 positionId) internal view returns (bool) {	// @audit-issue

71:    function isCreditPositionId(State storage state, uint256 positionId) internal view returns (bool) {	// @audit-issue
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L63-L63), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L71-L71), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

38:    function isNull(YieldCurve memory self) internal pure returns (bool) {	// @audit-issue

50:    function validateYieldCurve(YieldCurve memory self, uint256 minTenor, uint256 maxTenor) internal pure {	// @audit-issue

87:    function getAdjustedAPR(int256 apr, uint256 marketRateMultiplier, VariablePoolBorrowRateParams memory params)	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L38-L38), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L50-L50), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L87-L87), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

26:    function debtTokenAmountToCollateralTokenAmount(State storage state, uint256 debtTokenAmount)	// @audit-issue

153:    function reduceDebtAndCredit(State storage state, uint256 debtPositionId, uint256 creditPositionId, uint256 amount)	// @audit-issue

164:    function getSwapFeePercent(State storage state, uint256 tenor) internal view returns (uint256) {	// @audit-issue

173:    function getSwapFee(State storage state, uint256 cash, uint256 tenor) internal view returns (uint256) {	// @audit-issue

185:    function getCashAmountOut(	// @audit-issue
186:        State storage state,
187:        uint256 creditAmountIn,
188:        uint256 maxCredit,
189:        uint256 ratePerTenor,
190:        uint256 tenor
191:    ) internal view returns (uint256 cashAmountOut, uint256 fees) {

228:    function getCreditAmountIn(	// @audit-issue
229:        State storage state,
230:        uint256 cashAmountOut,
231:        uint256 maxCashAmountOut,
232:        uint256 maxCredit,
233:        uint256 ratePerTenor,
234:        uint256 tenor
235:    ) internal view returns (uint256 creditAmountIn, uint256 fees) {

274:    function getCreditAmountOut(	// @audit-issue
275:        State storage state,
276:        uint256 cashAmountIn,
277:        uint256 maxCashAmountIn,
278:        uint256 maxCredit,
279:        uint256 ratePerTenor,
280:        uint256 tenor
281:    ) internal view returns (uint256 creditAmountOut, uint256 fees) {

311:    function getCashAmountIn(	// @audit-issue
312:        State storage state,
313:        uint256 creditAmountOut,
314:        uint256 maxCredit,
315:        uint256 ratePerTenor,
316:        uint256 tenor
317:    ) internal view returns (uint256 cashAmountIn, uint256 fees) {
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L26-L26), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L153-L153), [164](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L164-L164), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L173-L173), [185](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L185-L191), [228](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L228-L235), [274](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L274-L281), [311](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L311-L317), 


```solidity
Path: ./src/libraries/Multicall.sol

26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L26-L26), 


```solidity
Path: ./src/libraries/Math.sol

15:    function min(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue

23:    function mulDivUp(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue

27:    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue

40:    function aprToRatePerTenor(uint256 apr, uint256 tenor) internal pure returns (uint256) {	// @audit-issue

51:    function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L15-L15), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L23-L23), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L27-L27), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), [40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L40-L40), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L51-L51), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

62:    function validateOwner(address owner) internal pure {	// @audit-issue

70:    function validateInitializeFeeConfigParams(InitializeFeeConfigParams memory f) internal pure {	// @audit-issue

98:    function validateInitializeRiskConfigParams(InitializeRiskConfigParams memory r) internal pure {	// @audit-issue

132:    function validateInitializeOracleParams(InitializeOracleParams memory o) internal view {	// @audit-issue

146:    function validateInitializeDataParams(InitializeDataParams memory d) internal view {	// @audit-issue

193:    function executeInitializeFeeConfig(State storage state, InitializeFeeConfigParams memory f) internal {	// @audit-issue

207:    function executeInitializeRiskConfig(State storage state, InitializeRiskConfigParams memory r) internal {	// @audit-issue

222:    function executeInitializeOracle(State storage state, InitializeOracleParams memory o) internal {	// @audit-issue

230:    function executeInitializeData(State storage state, InitializeDataParams memory d) internal {	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L62-L62), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L70-L70), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L98-L98), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L132-L132), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L146-L146), [193](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L193-L193), [207](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L207-L207), [222](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L222-L222), [230](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L230-L230), 


#### Recommendation

To adhere to the Solidity Style Guide, consider prefixing the names of non-`external`/`public` functions with an underscore (_). This naming convention enhances code readability and helps distinguish the visibility of functions.

### Non-`external`/`public` variable names should begin with an underscore
According to the Solidity Style Guide, non-external/public variable names should begin with an underscore


```solidity
Path: ./src/SizeStorage.sol

114:    State internal state;	// @audit-issue
```
[114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L114-L114), 


#### Recommendation

To adhere to the Solidity Style Guide, consider prefixing the names of non-`external`/`public` variables with an underscore (_). This naming convention enhances code readability and helps distinguish the visibility of  variables.

### Names of constants should use the UPPER_CASE_WITH_UNDERSCORES style
It is recommended by the [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html)

```solidity
Path: ./src/oracle/PriceFeed.sol

28:    uint256 public constant decimals = 18;	// @audit-issue
```
[28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L28-L28), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### Variable names for `immutable`s should use CONSTANT_CASE
For `immutable` variable names, each word should use all capital letters, with underscores separating each word (CONSTANT_CASE)

```solidity
Path: ./src/oracle/PriceFeed.sol

29:    AggregatorV3Interface public immutable base;	// @audit-issue name should be: BASE

30:    AggregatorV3Interface public immutable quote;	// @audit-issue name should be: QUOTE

31:    AggregatorV3Interface internal immutable sequencerUptimeFeed;	// @audit-issue name should be: SEQUENCER_UPTIME_FEED

32:    uint256 public immutable baseStalePriceInterval;	// @audit-issue name should be: BASE_STALE_PRICE_INTERVAL

33:    uint256 public immutable quoteStalePriceInterval;	// @audit-issue name should be: QUOTE_STALE_PRICE_INTERVAL
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L30-L30), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L31-L31), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L33-L33), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

15:    uint8 internal immutable _decimals;	// @audit-issue name should be: _DECIMALS
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L15-L15), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

20:    IPool private immutable variablePool;	// @audit-issue name should be: VARIABLE_POOL

21:    IERC20Metadata private immutable underlyingToken;	// @audit-issue name should be: UNDERLYING_TOKEN
```
[20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L21-L21), 


#### Recommendation

When naming `immutable` variables, follow the CONSTANT_CASE convention, which means using all capital letters with underscores to separate words. This naming convention improves code readability and aligns with best practices for naming constants.

### Names of `private`/`internal` state variables should be prefixed with an underscore
It is recommended by the [Solidity Style Guide](https://docs.soliditylang.org/en/v0.8.20/style-guide.html#underscore-prefix-for-non-external-functions-and-variables).

```solidity
Path: ./src/SizeStorage.sol

114:    State internal state;	// @audit-issue name should be: _tate
```
[114](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L114-L114), 


```solidity
Path: ./src/oracle/PriceFeed.sol

25:    uint256 private constant GRACE_PERIOD_TIME = 3600;	// @audit-issue name should be: _RACE_PERIOD_TIME

31:    AggregatorV3Interface internal immutable sequencerUptimeFeed;	// @audit-issue name should be: _equencerUptimeFeed
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L25-L25), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L31-L31), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

20:    IPool private immutable variablePool;	// @audit-issue name should be: _ariablePool

21:    IERC20Metadata private immutable underlyingToken;	// @audit-issue name should be: _nderlyingToken
```
[20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L21-L21), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### Using underscore at the end of variable name
The use of the underscore at the end of the variable name is unusual, consider refactoring it.

```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

26:        IPool variablePool_,	// @audit-issue

27:        IERC20Metadata underlyingToken_,	// @audit-issue

28:        address owner_,	// @audit-issue

29:        string memory name_,	// @audit-issue

30:        string memory symbol_,	// @audit-issue

31:        uint8 decimals_	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L27-L27), [28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L28-L28), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L30-L30), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L31-L31), 


#### Recommendation

Refrain from using an underscore at the end of variable names in Solidity. Opt for more conventional naming conventions to enhance code readability and maintainability.

### Avoid using `owner` or `_owner` as a parameter name
Using 'owner' or '_owner' as a parameter name in functions, especially in contracts inheriting from or interacting with OpenZeppelin's `Ownable` contract, can lead to confusion and potential bugs. These contracts often have a state variable named `owner`, managed by the `Ownable` pattern for access control. Using the same name for function parameters may obscure the contract's `owner` state variable, complicating code readability and maintenance. Prefer alternative descriptive names for parameters to maintain clarity and prevent conflicts with established ownership patterns.

```solidity
Path: ./src/Size.sol

88:        address owner,	// @audit-issue
```
[88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L88-L88), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

62:    function validateOwner(address owner) internal pure {	// @audit-issue

177:        address owner,	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L62-L62), [177](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L177-L177), 


#### Recommendation

Refactor your Solidity contracts to use descriptive and unambiguous names for function parameters, avoiding the use of `owner` or `_owner` if your contract inherits from or interacts with the `Ownable` contract or follows a similar ownership pattern. Opt for alternative names that clearly indicate the parameter's purpose without conflicting with the `owner` state variable.

### Revert statements within external and public functions can be used to perform DOS attacks
In Solidity, `revert` statements are used to undo changes and throw an exception when certain conditions are not met. However, in public and external functions, improper use of `revert` can be exploited for Denial of Service (DoS) attacks. An attacker can intentionally trigger these `revert' conditions, causing legitimate transactions to consistently fail. For example, if a function relies on specific conditions from user input or contract state, an attacker could manipulate these to continually force `revert`s, blocking the function's execution. Therefore, it's crucial to design contract logic to handle exceptions properly and avoid scenarios where `revert` can be predictably triggered by malicious actors. This includes careful input validation and considering alternative design patterns that are less susceptible to such abuses.

```solidity
Path: ./src/SizeView.sol

144:            revert Errors.NULL_OFFER();	// @audit-issue

160:            revert Errors.NULL_OFFER();	// @audit-issue

181:            revert Errors.NULL_TENOR();	// @audit-issue
```
[144](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L144-L144), [160](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L160-L160), [181](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L181-L181), 


```solidity
Path: ./src/oracle/PriceFeed.sol

70:                revert Errors.SEQUENCER_DOWN();	// @audit-issue

75:                revert Errors.GRACE_PERIOD_NOT_OVER();	// @audit-issue
```
[70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L70-L70), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L75-L75), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

23:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(credit, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

33:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT_OPENING(credit, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

43:            revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);	// @audit-issue

131:            revert Errors.USER_IS_UNDERWATER(account, collateralRatio(state, account));	// @audit-issue

146:            revert Errors.CR_BELOW_OPENING_LIMIT_BORROW_CR(	// @audit-issue
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L23-L23), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L33-L33), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L43-L43), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L131-L131), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L146-L146), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

84:            revert Errors.INVALID_DEBT_POSITION_ID(debtPositionId);	// @audit-issue

101:            revert Errors.INVALID_CREDIT_POSITION_ID(creditPositionId);	// @audit-issue

130:            revert Errors.INVALID_POSITION_ID(positionId);	// @audit-issue
```
[84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L84-L84), [101](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L101-L101), [130](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L130-L130), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

122:            revert Errors.TENOR_OUT_OF_RANGE(tenor, curveRelativeTime.tenors[0], curveRelativeTime.tenors[length - 1]);	// @audit-issue
```
[122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L122-L122), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

37:                revert Errors.BORROW_ATOKEN_INCREASE_EXCEEDS_DEBT_TOKEN_DECREASE(	// @audit-issue

54:            revert Errors.BORROW_ATOKEN_CAP_EXCEEDED(	// @audit-issue

70:            revert Errors.NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(liquidity, amount);	// @audit-issue
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L37-L37), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L54-L54), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L70-L70), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

51:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionIds[i]);	// @audit-issue

55:                revert Errors.LOAN_NOT_ACTIVE(params.creditPositionIds[i]);	// @audit-issue
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L51-L51), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L55-L55), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

40:            revert Errors.LOAN_NOT_SELF_LIQUIDATABLE(	// @audit-issue

47:            revert Errors.LIQUIDATION_NOT_AT_LOSS(params.creditPositionId, state.collateralRatio(debtPosition.borrower));	// @audit-issue

52:            revert Errors.LIQUIDATOR_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L40-L40), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L47-L47), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

52:            revert Errors.LOAN_NOT_ACTIVE(params.creditPositionWithDebtToRepayId);	// @audit-issue

61:                revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);	// @audit-issue

69:                revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(	// @audit-issue

79:                revert Errors.DUE_DATE_NOT_COMPATIBLE(	// @audit-issue

84:                revert Errors.INVALID_LENDER(creditPositionToCompensate.lender);	// @audit-issue

87:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionToCompensateId);	// @audit-issue

94:            revert Errors.COMPENSATOR_IS_NOT_BORROWER(msg.sender, debtPositionToRepay.borrower);	// @audit-issue

99:            revert Errors.NULL_AMOUNT();	// @audit-issue
```
[52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L52-L52), [61](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L61-L61), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L69-L69), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L79-L79), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L84-L84), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L87-L87), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L94-L94), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L99-L99), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

60:            revert Errors.INVALID_LOAN_OFFER(params.lender);	// @audit-issue

69:                revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);	// @audit-issue

75:                revert Errors.BORROWER_IS_NOT_LENDER(msg.sender, creditPosition.lender);	// @audit-issue

78:                revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(	// @audit-issue

88:                revert Errors.NOT_ENOUGH_CREDIT(params.amount, creditPosition.credit);	// @audit-issue

94:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

99:            revert Errors.DUE_DATE_GREATER_THAN_MAX_DUE_DATE(block.timestamp + tenor, loanOffer.maxDueDate);	// @audit-issue

104:            revert Errors.PAST_DEADLINE(params.deadline);	// @audit-issue

117:            revert Errors.APR_GREATER_THAN_MAX_APR(apr, params.maxAPR);	// @audit-issue
```
[60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L60-L60), [69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L69-L69), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L75-L75), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L78-L78), [88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L88-L88), [94](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L94-L94), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L99-L99), [104](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L104-L104), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L117-L117), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

91:                revert Errors.INVALID_COLLATERAL_RATIO(params.value);	// @audit-issue

103:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue

113:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue

120:                revert Errors.VALUE_GREATER_THAN_MAX(	// @audit-issue

140:            revert Errors.INVALID_KEY(params.key);	// @audit-issue
```
[91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L91-L91), [103](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L103-L103), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L113-L113), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L120-L120), [140](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L140-L140), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

45:            revert Errors.LOAN_NOT_LIQUIDATABLE(	// @audit-issue

65:            revert Errors.LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(	// @audit-issue
```
[45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L45-L45), [65](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L65-L65), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

64:            revert Errors.LOAN_NOT_ACTIVE(params.debtPositionId);	// @audit-issue

68:            revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);	// @audit-issue

73:            revert Errors.INVALID_BORROW_OFFER(params.borrower);	// @audit-issue

78:            revert Errors.PAST_DEADLINE(params.deadline);	// @audit-issue

91:            revert Errors.APR_LOWER_THAN_MIN_APR(apr, params.minAPR);	// @audit-issue
```
[64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L64-L64), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L68-L68), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L73-L73), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L78-L78), [91](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L91-L91), 


```solidity
Path: ./src/libraries/actions/Claim.sol

38:            revert Errors.LOAN_NOT_REPAID(params.creditPositionId);	// @audit-issue

41:            revert Errors.CREDIT_POSITION_ALREADY_CLAIMED(params.creditPositionId);	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L38-L38), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L41-L41), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

38:            revert Errors.INVALID_TOKEN(params.token);	// @audit-issue

43:            revert Errors.NULL_AMOUNT();	// @audit-issue

48:            revert Errors.NULL_ADDRESS();	// @audit-issue
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L38-L38), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L43-L43), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L48-L48), 


```solidity
Path: ./src/libraries/actions/Repay.sol

36:            revert Errors.LOAN_ALREADY_REPAID(params.debtPositionId);	// @audit-issue
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L36-L36), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

40:                revert Errors.NULL_MAX_DUE_DATE();	// @audit-issue

43:                revert Errors.PAST_MAX_DUE_DATE(params.maxDueDate);	// @audit-issue
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L40-L40), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L43-L43), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

62:                revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);	// @audit-issue

68:                revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(	// @audit-issue

76:                revert Errors.CREDIT_NOT_FOR_SALE(params.creditPositionId);	// @audit-issue

87:            revert Errors.INVALID_BORROW_OFFER(borrower);	// @audit-issue

92:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);	// @audit-issue

97:            revert Errors.PAST_DEADLINE(params.deadline);	// @audit-issue

110:            revert Errors.APR_LOWER_THAN_MIN_APR(apr, params.minAPR);	// @audit-issue
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L62-L62), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L68-L68), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L76-L76), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L87-L87), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L92-L92), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L97-L97), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L110-L110), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

42:            revert Errors.INVALID_MSG_VALUE(msg.value);	// @audit-issue

50:            revert Errors.INVALID_TOKEN(params.token);	// @audit-issue

55:            revert Errors.NULL_AMOUNT();	// @audit-issue

60:            revert Errors.NULL_ADDRESS();	// @audit-issue
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L42-L42), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L50-L50), [55](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L55-L55), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L60-L60), 


#### Recommendation

Design your Solidity contract's public and external functions with care to mitigate the risk of DoS attacks via `revert` statements. Implement robust input validation to ensure inputs are within expected bounds and conditions. Consider alternative logic or design patterns that reduce the reliance on `revert` for critical operations, particularly those that can be influenced externally. Evaluate the use of modifiers, try-catch blocks, or state checks that allow for safer handling of conditions and exceptions. Ensure that your contract's critical functionality remains accessible and resilient against potential abuse of `revert` behavior by malicious actors.

### Consider adding emergency-stop functionality
In the event of a security breach or any unforeseen emergency, swiftly suspending all protocol operations becomes crucial. Having a mechanism in place to halt all functions collectively, instead of pausing individual contracts separately, substantially enhances the efficiency of mitigating ongoing attacks or vulnerabilities. This not only quickens the response time to potential threats but also reduces operational stress during these critical periods. Therefore, consider integrating a 'circuit breaker' or 'emergency stop' function into the smart contract system architecture. Such a feature would provide the capability to suspend the entire protocol instantly, which could prove invaluable during a time-sensitive crisis management situation.

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
Path: ./src/libraries/OfferLibrary.sol

26:library OfferLibrary {	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L26-L26), 


```solidity
Path: ./src/libraries/Errors.sol

9:library Errors {	// @audit-issue
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L9-L9), 


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
Path: ./src/libraries/Events.sol

15:library Events {	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L15-L15), 


```solidity
Path: ./src/libraries/Multicall.sol

20:library Multicall {	// @audit-issue
```
[20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L20-L20), 


```solidity
Path: ./src/libraries/Math.sol

14:library Math {	// @audit-issue
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L14-L14), 


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

Implement an emergency-stop feature in your Solidity contract system to enhance security and crisis response capabilities. This can be achieved through a 'circuit breaker' pattern, where a central switch or set of conditions can instantly suspend critical operations across the contract ecosystem. Ensure that this mechanism is accessible to authorized parties only, such as contract administrators or a decentralized governance system. Design the emergency-stop functionality to be transparent and auditable, with clear conditions and processes for activation and deactivation. Regularly test and audit this feature to ensure its reliability and effectiveness in potential emergency situations.

### For extended 'using-for' usage, use the latest pragma version
Solidity versions of 0.8.13 or above can make use of enhanced using-for notation within contracts.

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
Path: ./src/libraries/RiskLibrary.sol

14:library RiskLibrary {	// @audit-issue
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L14-L14), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

26:library OfferLibrary {	// @audit-issue
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L26-L26), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

56:library LoanLibrary {	// @audit-issue
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L56-L56), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

16:library AccountingLibrary {	// @audit-issue
```
[16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L16-L16), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

15:library DepositTokenLibrary {	// @audit-issue
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L15-L15), 


```solidity
Path: ./src/libraries/Multicall.sol

20:library Multicall {	// @audit-issue
```
[20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L20-L20), 


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


#### Recommendation

Update your Solidity contracts to use the latest pragma version, ideally 0.8.13 or higher, to leverage the enhanced 'using-for' notation. This upgrade will enable you to apply library functions to user-defined types more effectively, enhancing the functionality and readability of your contracts. Make sure to thoroughly test your contracts after upgrading to ensure compatibility and to take full advantage of the latest Solidity features and improvements. Regularly keep your contracts updated with the latest Solidity versions to stay aligned with the evolving capabilities and best practices of the language.

### Cyclomatic complexity in functions
Cyclomatic complexity is a software metric used to measure the complexity of a program. It quantifies the number of linearly independent paths through a program's source code, giving an idea of how complex the control flow is. High cyclomatic complexity may indicate a higher risk of defects and can make the code harder to understand, test, and maintain. It often suggests that a function or method is trying to do too much, and a refactor might be needed. By breaking down complex functions into smaller, more focused pieces, you can improve readability, ease of testing, and overall maintainability.

```solidity
Path: ./src/libraries/actions/Compensate.sol

42:    function validateCompensate(State storage state, CompensateParams calldata params) external view {	// @audit-issue
43:        CreditPosition storage creditPositionWithDebtToRepay =
44:            state.getCreditPosition(params.creditPositionWithDebtToRepayId);
45:        DebtPosition storage debtPositionToRepay =
46:            state.getDebtPositionByCreditPositionId(params.creditPositionWithDebtToRepayId);
47:
48:        uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);
49:
50:        // validate creditPositionWithDebtToRepayId
51:        if (state.getLoanStatus(params.creditPositionWithDebtToRepayId) != LoanStatus.ACTIVE) {
52:            revert Errors.LOAN_NOT_ACTIVE(params.creditPositionWithDebtToRepayId);
53:        }
54:
55:        // validate creditPositionToCompensateId
56:        if (params.creditPositionToCompensateId == RESERVED_ID) {
57:            uint256 tenor = debtPositionToRepay.dueDate - block.timestamp;
58:
59:            // validate tenor
60:            if (tenor < state.riskConfig.minTenor || tenor > state.riskConfig.maxTenor) {
61:                revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);
62:            }
63:        } else {
64:            CreditPosition storage creditPositionToCompensate =
65:                state.getCreditPosition(params.creditPositionToCompensateId);
66:            DebtPosition storage debtPositionToCompensate =
67:                state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId);
68:            if (!state.isCreditPositionTransferrable(params.creditPositionToCompensateId)) {
69:                revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(
70:                    params.creditPositionToCompensateId,
71:                    state.getLoanStatus(params.creditPositionToCompensateId),
72:                    state.collateralRatio(debtPositionToCompensate.borrower)
73:                );
74:            }
75:            if (
76:                debtPositionToRepay.dueDate
77:                    < state.getDebtPositionByCreditPositionId(params.creditPositionToCompensateId).dueDate
78:            ) {
79:                revert Errors.DUE_DATE_NOT_COMPATIBLE(
80:                    params.creditPositionWithDebtToRepayId, params.creditPositionToCompensateId
81:                );
82:            }
83:            if (creditPositionToCompensate.lender != debtPositionToRepay.borrower) {
84:                revert Errors.INVALID_LENDER(creditPositionToCompensate.lender);
85:            }
86:            if (params.creditPositionToCompensateId == params.creditPositionWithDebtToRepayId) {
87:                revert Errors.INVALID_CREDIT_POSITION_ID(params.creditPositionToCompensateId);
88:            }
89:            amountToCompensate = Math.min(amountToCompensate, creditPositionToCompensate.credit);
90:        }
91:
92:        // validate msg.sender
93:        if (msg.sender != debtPositionToRepay.borrower) {
94:            revert Errors.COMPENSATOR_IS_NOT_BORROWER(msg.sender, debtPositionToRepay.borrower);
95:        }
96:
97:        // validate amount
98:        if (amountToCompensate == 0) {
99:            revert Errors.NULL_AMOUNT();
100:        }
101:    }
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L42-L101), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

51:    function validateSellCreditMarket(State storage state, SellCreditMarketParams calldata params) external view {	// @audit-issue
52:        LoanOffer memory loanOffer = state.data.users[params.lender].loanOffer;
53:        uint256 tenor;
54:
55:        // validate msg.sender
56:        // N/A
57:
58:        // validate lender
59:        if (loanOffer.isNull()) {
60:            revert Errors.INVALID_LOAN_OFFER(params.lender);
61:        }
62:
63:        // validate creditPositionId
64:        if (params.creditPositionId == RESERVED_ID) {
65:            tenor = params.tenor;
66:
67:            // validate tenor
68:            if (tenor < state.riskConfig.minTenor || tenor > state.riskConfig.maxTenor) {
69:                revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);
70:            }
71:        } else {
72:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);
73:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);
74:            if (msg.sender != creditPosition.lender) {
75:                revert Errors.BORROWER_IS_NOT_LENDER(msg.sender, creditPosition.lender);
76:            }
77:            if (!state.isCreditPositionTransferrable(params.creditPositionId)) {
78:                revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(
79:                    params.creditPositionId,
80:                    state.getLoanStatus(params.creditPositionId),
81:                    state.collateralRatio(debtPosition.borrower)
82:                );
83:            }
84:            tenor = debtPosition.dueDate - block.timestamp; // positive since the credit position is transferrable, so the loan must be ACTIVE
85:
86:            // validate amount
87:            if (params.amount > creditPosition.credit) {
88:                revert Errors.NOT_ENOUGH_CREDIT(params.amount, creditPosition.credit);
89:            }
90:        }
91:
92:        // validate amount
93:        if (params.amount < state.riskConfig.minimumCreditBorrowAToken) {
94:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);
95:        }
96:
97:        // validate tenor
98:        if (block.timestamp + tenor > loanOffer.maxDueDate) {
99:            revert Errors.DUE_DATE_GREATER_THAN_MAX_DUE_DATE(block.timestamp + tenor, loanOffer.maxDueDate);
100:        }
101:
102:        // validate deadline
103:        if (params.deadline < block.timestamp) {
104:            revert Errors.PAST_DEADLINE(params.deadline);
105:        }
106:
107:        // validate maxAPR
108:        uint256 apr = loanOffer.getAPRByTenor(
109:            VariablePoolBorrowRateParams({
110:                variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
111:                variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
112:                variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
113:            }),
114:            tenor
115:        );
116:        if (apr > params.maxAPR) {
117:            revert Errors.APR_GREATER_THAN_MAX_APR(apr, params.maxAPR);
118:        }
119:
120:        // validate exactAmountIn
121:        // N/A
122:    }
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L51-L122), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

86:    function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {	// @audit-issue
87:        if (Strings.equal(params.key, "crOpening")) {
88:            state.riskConfig.crOpening = params.value;
89:        } else if (Strings.equal(params.key, "crLiquidation")) {
90:            if (params.value >= state.riskConfig.crLiquidation) {
91:                revert Errors.INVALID_COLLATERAL_RATIO(params.value);
92:            }
93:            state.riskConfig.crLiquidation = params.value;
94:        } else if (Strings.equal(params.key, "minimumCreditBorrowAToken")) {
95:            state.riskConfig.minimumCreditBorrowAToken = params.value;
96:        } else if (Strings.equal(params.key, "borrowATokenCap")) {
97:            state.riskConfig.borrowATokenCap = params.value;
98:        } else if (Strings.equal(params.key, "minTenor")) {
99:            if (
100:                state.feeConfig.swapFeeAPR != 0
101:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
102:            ) {
103:                revert Errors.VALUE_GREATER_THAN_MAX(
104:                    params.value, Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
105:                );
106:            }
107:            state.riskConfig.minTenor = params.value;
108:        } else if (Strings.equal(params.key, "maxTenor")) {
109:            if (
110:                state.feeConfig.swapFeeAPR != 0
111:                    && params.value >= Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
112:            ) {
113:                revert Errors.VALUE_GREATER_THAN_MAX(
114:                    params.value, Math.mulDivDown(YEAR, PERCENT, state.feeConfig.swapFeeAPR)
115:                );
116:            }
117:            state.riskConfig.maxTenor = params.value;
118:        } else if (Strings.equal(params.key, "swapFeeAPR")) {
119:            if (params.value >= Math.mulDivDown(PERCENT, YEAR, state.riskConfig.maxTenor)) {
120:                revert Errors.VALUE_GREATER_THAN_MAX(
121:                    params.value, Math.mulDivDown(PERCENT, YEAR, state.riskConfig.maxTenor)
122:                );
123:            }
124:            state.feeConfig.swapFeeAPR = params.value;
125:        } else if (Strings.equal(params.key, "fragmentationFee")) {
126:            state.feeConfig.fragmentationFee = params.value;
127:        } else if (Strings.equal(params.key, "liquidationRewardPercent")) {
128:            state.feeConfig.liquidationRewardPercent = params.value;
129:        } else if (Strings.equal(params.key, "overdueCollateralProtocolPercent")) {
130:            state.feeConfig.overdueCollateralProtocolPercent = params.value;
131:        } else if (Strings.equal(params.key, "collateralProtocolPercent")) {
132:            state.feeConfig.collateralProtocolPercent = params.value;
133:        } else if (Strings.equal(params.key, "feeRecipient")) {
134:            state.feeConfig.feeRecipient = address(uint160(params.value));
135:        } else if (Strings.equal(params.key, "priceFeed")) {
136:            state.oracle.priceFeed = IPriceFeed(address(uint160(params.value)));
137:        } else if (Strings.equal(params.key, "variablePoolBorrowRateStaleRateInterval")) {
138:            state.oracle.variablePoolBorrowRateStaleRateInterval = uint64(params.value);
139:        } else {
140:            revert Errors.INVALID_KEY(params.key);
141:        }
142:
143:        Initialize.validateInitializeFeeConfigParams(feeConfigParams(state));
144:        Initialize.validateInitializeRiskConfigParams(riskConfigParams(state));
145:        Initialize.validateInitializeOracleParams(oracleParams(state));
146:
147:        emit Events.UpdateConfig(params.key, params.value);
148:    }
```
[86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L86-L148), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

98:    function validateInitializeRiskConfigParams(InitializeRiskConfigParams memory r) internal pure {	// @audit-issue
99:        // validate crOpening
100:        if (r.crOpening < PERCENT) {
101:            revert Errors.INVALID_COLLATERAL_RATIO(r.crOpening);
102:        }
103:
104:        // validate crLiquidation
105:        if (r.crLiquidation < PERCENT) {
106:            revert Errors.INVALID_COLLATERAL_RATIO(r.crLiquidation);
107:        }
108:        if (r.crOpening <= r.crLiquidation) {
109:            revert Errors.INVALID_LIQUIDATION_COLLATERAL_RATIO(r.crOpening, r.crLiquidation);
110:        }
111:
112:        // validate minimumCreditBorrowAToken
113:        if (r.minimumCreditBorrowAToken == 0) {
114:            revert Errors.NULL_AMOUNT();
115:        }
116:
117:        // validate underlyingBorrowTokenCap
118:        // N/A
119:
120:        // validate minTenor
121:        if (r.minTenor == 0) {
122:            revert Errors.NULL_AMOUNT();
123:        }
124:
125:        if (r.maxTenor <= r.minTenor) {
126:            revert Errors.INVALID_MAXIMUM_TENOR(r.maxTenor);
127:        }
128:    }
```
[98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L98-L128), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

51:    function validateBuyCreditMarket(State storage state, BuyCreditMarketParams calldata params) external view {	// @audit-issue
52:        address borrower;
53:        uint256 tenor;
54:
55:        // validate creditPositionId
56:        if (params.creditPositionId == RESERVED_ID) {
57:            borrower = params.borrower;
58:            tenor = params.tenor;
59:
60:            // validate tenor
61:            if (tenor < state.riskConfig.minTenor || tenor > state.riskConfig.maxTenor) {
62:                revert Errors.TENOR_OUT_OF_RANGE(tenor, state.riskConfig.minTenor, state.riskConfig.maxTenor);
63:            }
64:        } else {
65:            CreditPosition storage creditPosition = state.getCreditPosition(params.creditPositionId);
66:            DebtPosition storage debtPosition = state.getDebtPositionByCreditPositionId(params.creditPositionId);
67:            if (!state.isCreditPositionTransferrable(params.creditPositionId)) {
68:                revert Errors.CREDIT_POSITION_NOT_TRANSFERRABLE(
69:                    params.creditPositionId,
70:                    state.getLoanStatus(params.creditPositionId),
71:                    state.collateralRatio(debtPosition.borrower)
72:                );
73:            }
74:            User storage user = state.data.users[creditPosition.lender];
75:            if (user.allCreditPositionsForSaleDisabled || !creditPosition.forSale) {
76:                revert Errors.CREDIT_NOT_FOR_SALE(params.creditPositionId);
77:            }
78:
79:            borrower = creditPosition.lender;
80:            tenor = debtPosition.dueDate - block.timestamp; // positive since the credit position is transferrable, so the loan must be ACTIVE
81:        }
82:
83:        BorrowOffer memory borrowOffer = state.data.users[borrower].borrowOffer;
84:
85:        // validate borrower
86:        if (borrowOffer.isNull()) {
87:            revert Errors.INVALID_BORROW_OFFER(borrower);
88:        }
89:
90:        // validate amount
91:        if (params.amount < state.riskConfig.minimumCreditBorrowAToken) {
92:            revert Errors.CREDIT_LOWER_THAN_MINIMUM_CREDIT(params.amount, state.riskConfig.minimumCreditBorrowAToken);
93:        }
94:
95:        // validate deadline
96:        if (params.deadline < block.timestamp) {
97:            revert Errors.PAST_DEADLINE(params.deadline);
98:        }
99:
100:        // validate minAPR
101:        uint256 apr = borrowOffer.getAPRByTenor(
102:            VariablePoolBorrowRateParams({
103:                variablePoolBorrowRate: state.oracle.variablePoolBorrowRate,
104:                variablePoolBorrowRateUpdatedAt: state.oracle.variablePoolBorrowRateUpdatedAt,
105:                variablePoolBorrowRateStaleRateInterval: state.oracle.variablePoolBorrowRateStaleRateInterval
106:            }),
107:            tenor
108:        );
109:        if (apr < params.minAPR) {
110:            revert Errors.APR_LOWER_THAN_MIN_APR(apr, params.minAPR);
111:        }
112:
113:        // validate exactAmountIn
114:        // N/A
115:    }
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L51-L115), 


#### Recommendation

Regularly analyze your Solidity contracts for functions with high cyclomatic complexity. Consider refactoring these functions into smaller, more focused units. This could involve breaking down complex functions into multiple simpler functions, reducing the number of conditional branches, or simplifying logic where possible. Such refactoring improves the readability and testability of your code and reduces the likelihood of defects. Additionally, simpler functions are easier to audit and maintain, enhancing the overall security and robustness of your contract. Employ tools or metrics to periodically assess the complexity of your functions as part of your development and maintenance processes.

### Consider only defining one library/interface/contract per sol file
Combining multiple libraries, interfaces, or contracts in a single .sol file can lead to clutter, reduced readability, and versioning issues. Resolution: Adopt the best practice of defining only one library, interface, or contract per Solidity file. This modular approach enhances clarity, simplifies unit testing, and streamlines code review. Furthermore, segregating components makes version management easier, as updates to one component won't necessitate changes to a file housing multiple unrelated components. Structured file management can further assist in avoiding naming collisions and ensure smoother integration into larger systems or DApps.

```solidity
Path: ./src/SizeViewData.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue
5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
6:
7:import {User} from "@src/SizeStorage.sol";
8:import {NonTransferrableScaledToken} from "@src/token/NonTransferrableScaledToken.sol";
9:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeViewData.sol#L4-L9), 


```solidity
Path: ./src/Size.sol

4:import {AccessControlUpgradeable} from "@openzeppelin/contracts-upgradeable/access/AccessControlUpgradeable.sol";	// @audit-issue
5:
6:import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
7:import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
8:import {PausableUpgradeable} from "@openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol";
9:import {RESERVED_ID} from "@src/libraries/LoanLibrary.sol";
10:
11:import {
12:    Initialize,
13:    InitializeDataParams,
14:    InitializeFeeConfigParams,
15:    InitializeOracleParams,
16:    InitializeRiskConfigParams
17:} from "@src/libraries/actions/Initialize.sol";
18:import {UpdateConfig, UpdateConfigParams} from "@src/libraries/actions/UpdateConfig.sol";
19:
20:import {SellCreditLimit, SellCreditLimitParams} from "@src/libraries/actions/SellCreditLimit.sol";
21:import {SellCreditMarket, SellCreditMarketParams} from "@src/libraries/actions/SellCreditMarket.sol";
22:
23:import {Claim, ClaimParams} from "@src/libraries/actions/Claim.sol";
24:import {Deposit, DepositParams} from "@src/libraries/actions/Deposit.sol";
25:
26:import {BuyCreditMarket, BuyCreditMarketParams} from "@src/libraries/actions/BuyCreditMarket.sol";
27:import {SetUserConfiguration, SetUserConfigurationParams} from "@src/libraries/actions/SetUserConfiguration.sol";
28:
29:import {BuyCreditLimit, BuyCreditLimitParams} from "@src/libraries/actions/BuyCreditLimit.sol";
30:import {Liquidate, LiquidateParams} from "@src/libraries/actions/Liquidate.sol";
31:
32:import {Multicall} from "@src/libraries/Multicall.sol";
33:import {Compensate, CompensateParams} from "@src/libraries/actions/Compensate.sol";
34:import {
35:    LiquidateWithReplacement,
36:    LiquidateWithReplacementParams
37:} from "@src/libraries/actions/LiquidateWithReplacement.sol";
38:import {Repay, RepayParams} from "@src/libraries/actions/Repay.sol";
39:import {SelfLiquidate, SelfLiquidateParams} from "@src/libraries/actions/SelfLiquidate.sol";
40:import {Withdraw, WithdrawParams} from "@src/libraries/actions/Withdraw.sol";
41:
42:import {State} from "@src/SizeStorage.sol";
43:
44:import {CapsLibrary} from "@src/libraries/CapsLibrary.sol";
45:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
46:
47:import {SizeView} from "@src/SizeView.sol";
48:import {Events} from "@src/libraries/Events.sol";
49:
50:import {IMulticall} from "@src/interfaces/IMulticall.sol";
51:import {ISize} from "@src/interfaces/ISize.sol";
52:import {ISizeAdmin} from "@src/interfaces/ISizeAdmin.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L4-L52), 


```solidity
Path: ./src/SizeStorage.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue
5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
6:import {IWETH} from "@src/interfaces/IWETH.sol";
7:
8:import {CreditPosition, DebtPosition} from "@src/libraries/LoanLibrary.sol";
9:import {BorrowOffer, LoanOffer} from "@src/libraries/OfferLibrary.sol";
10:
11:import {IPriceFeed} from "@src/oracle/IPriceFeed.sol";
12:
13:import {NonTransferrableScaledToken} from "@src/token/NonTransferrableScaledToken.sol";
14:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeStorage.sol#L4-L14), 


```solidity
Path: ./src/SizeView.sol

4:import {SizeStorage, State, User} from "@src/SizeStorage.sol";	// @audit-issue
5:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";
6:
7:import {
8:    CREDIT_POSITION_ID_START,
9:    CreditPosition,
10:    DEBT_POSITION_ID_START,
11:    DebtPosition,
12:    LoanLibrary,
13:    LoanStatus,
14:    RESERVED_ID
15:} from "@src/libraries/LoanLibrary.sol";
16:import {UpdateConfig} from "@src/libraries/actions/UpdateConfig.sol";
17:
18:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
19:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
20:
21:import {DataView, UserView} from "@src/SizeViewData.sol";
22:
23:import {ISizeView} from "@src/interfaces/ISizeView.sol";
24:import {Errors} from "@src/libraries/Errors.sol";
25:import {BorrowOffer, LoanOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";
26:import {
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L4-L26), 


```solidity
Path: ./src/oracle/IPriceFeed.sol

4:/// @title IPriceFeed	// @audit-issue
5:/// @custom:security-contact security@size.credit
6:/// @author Size (https://size.credit/)
7:interface IPriceFeed {
8:    /// @notice Returns the price of the asset
9:    function getPrice() external view returns (uint256);
10:    /// @notice Returns the number of decimals of the price feed
11:    function decimals() external view returns (uint256);
12:}
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L4-L26), 


```solidity
Path: ./src/oracle/PriceFeed.sol

4:import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";	// @audit-issue
5:import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
6:import {Math} from "@src/libraries/Math.sol";
7:
8:import {IPriceFeed} from "./IPriceFeed.sol";
9:import {Errors} from "@src/libraries/Errors.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L4-L9), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {Errors} from "@src/libraries/Errors.sol";
7:
8:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";
9:import {Math} from "@src/libraries/Math.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L4-L9), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

4:import {Errors} from "@src/libraries/Errors.sol";	// @audit-issue
5:import {Math} from "@src/libraries/Math.sol";
6:import {VariablePoolBorrowRateParams, YieldCurve, YieldCurveLibrary} from "@src/libraries/YieldCurveLibrary.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L4-L6), 


```solidity
Path: ./src/libraries/Errors.sol

4:import {LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L4-L4), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
7:import {Errors} from "@src/libraries/Errors.sol";
8:import {Math} from "@src/libraries/Math.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L4-L8), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

4:import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";	// @audit-issue
5:import {Errors} from "@src/libraries/Errors.sol";
6:import {Math, PERCENT} from "@src/libraries/Math.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L4-L6), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {Errors} from "@src/libraries/Errors.sol";
7:import {Events} from "@src/libraries/Events.sol";
8:import {Math, PERCENT, YEAR} from "@src/libraries/Math.sol";
9:
10:import {CreditPosition, DebtPosition, LoanLibrary, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";
11:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L4-L11), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:import {Errors} from "@src/libraries/Errors.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L4-L5), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

4:import {IAToken} from "@aave/interfaces/IAToken.sol";	// @audit-issue
5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
6:import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
7:
8:import {State} from "@src/SizeStorage.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L4-L8), 


```solidity
Path: ./src/libraries/Events.sol

4:import {LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue
5:import {
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L4-L5), 


```solidity
Path: ./src/libraries/Multicall.sol

4:import {Address} from "@openzeppelin/contracts/utils/Address.sol";	// @audit-issue
5:
6:import {State} from "@src/SizeStorage.sol";
7:import {CapsLibrary} from "@src/libraries/CapsLibrary.sol";
8:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L4-L8), 


```solidity
Path: ./src/libraries/Math.sol

4:import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";	// @audit-issue
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L4-L4), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

4:import {State, User} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {CreditPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";
7:
8:import {Errors} from "@src/libraries/Errors.sol";
9:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L4-L9), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

4:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";	// @audit-issue
5:
6:import {CreditPosition, DebtPosition, LoanLibrary} from "@src/libraries/LoanLibrary.sol";
7:import {PERCENT} from "@src/libraries/Math.sol";
8:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
9:
10:import {State} from "@src/SizeStorage.sol";
11:
12:import {Errors} from "@src/libraries/Errors.sol";
13:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L4-L13), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {Math} from "@src/libraries/Math.sol";
7:
8:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
9:
10:import {Errors} from "@src/libraries/Errors.sol";
11:import {Events} from "@src/libraries/Events.sol";
12:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";
13:
14:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L4-L14), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

4:import {CreditPosition, DebtPosition, LoanLibrary, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";	// @audit-issue
5:import {Math, PERCENT} from "@src/libraries/Math.sol";
6:import {LoanOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";
7:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";
8:
9:import {State} from "@src/SizeStorage.sol";
10:
11:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
12:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
13:
14:import {Errors} from "@src/libraries/Errors.sol";
15:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L4-L15), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

4:import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";	// @audit-issue
5:import {State} from "@src/SizeStorage.sol";
6:import {Errors} from "@src/libraries/Errors.sol";
7:import {Events} from "@src/libraries/Events.sol";
8:
9:import {Math, PERCENT, YEAR} from "@src/libraries/Math.sol";
10:import {Initialize} from "@src/libraries/actions/Initialize.sol";
11:
12:import {IPriceFeed} from "@src/oracle/IPriceFeed.sol";
13:
14:import {
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L4-L14), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

4:import {Math} from "@src/libraries/Math.sol";	// @audit-issue
5:
6:import {PERCENT} from "@src/libraries/Math.sol";
7:
8:import {DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";
9:
10:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
11:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
12:
13:import {State} from "@src/SizeStorage.sol";
14:
15:import {Errors} from "@src/libraries/Errors.sol";
16:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L4-L16), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

4:import {Math} from "@src/libraries/Math.sol";	// @audit-issue
5:
6:import {PERCENT} from "@src/libraries/Math.sol";
7:
8:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";
9:import {BorrowOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";
10:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";
11:
12:import {State} from "@src/SizeStorage.sol";
13:
14:import {Liquidate, LiquidateParams} from "@src/libraries/actions/Liquidate.sol";
15:
16:import {Errors} from "@src/libraries/Errors.sol";
17:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L4-L17), 


```solidity
Path: ./src/libraries/actions/Claim.sol

4:import {CreditPosition, DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";	// @audit-issue
5:import {Math} from "@src/libraries/Math.sol";
6:
7:import {State} from "@src/SizeStorage.sol";
8:
9:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
10:
11:import {Errors} from "@src/libraries/Errors.sol";
12:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L4-L12), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue
5:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
6:import {IWETH} from "@src/interfaces/IWETH.sol";
7:
8:import {Math} from "@src/libraries/Math.sol";
9:
10:import {CREDIT_POSITION_ID_START, DEBT_POSITION_ID_START} from "@src/libraries/LoanLibrary.sol";
11:import {PERCENT} from "@src/libraries/Math.sol";
12:
13:import {IPriceFeed} from "@src/oracle/IPriceFeed.sol";
14:
15:import {NonTransferrableScaledToken} from "@src/token/NonTransferrableScaledToken.sol";
16:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";
17:
18:import {State} from "@src/SizeStorage.sol";
19:
20:import {Errors} from "@src/libraries/Errors.sol";
21:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L4-L21), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {DepositTokenLibrary} from "@src/libraries/DepositTokenLibrary.sol";
7:import {Math} from "@src/libraries/Math.sol";
8:
9:import {Errors} from "@src/libraries/Errors.sol";
10:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L4-L10), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:import {BorrowOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";
6:import {YieldCurve, YieldCurveLibrary} from "@src/libraries/YieldCurveLibrary.sol";
7:
8:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L4-L8), 


```solidity
Path: ./src/libraries/actions/Repay.sol

4:import {State} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
7:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
8:
9:import {DebtPosition, LoanLibrary, LoanStatus} from "@src/libraries/LoanLibrary.sol";
10:
11:import {Errors} from "@src/libraries/Errors.sol";
12:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L4-L12), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

4:import {LoanOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";	// @audit-issue
5:import {YieldCurve, YieldCurveLibrary} from "@src/libraries/YieldCurveLibrary.sol";
6:
7:import {State} from "@src/SizeStorage.sol";
8:
9:import {Errors} from "@src/libraries/Errors.sol";
10:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L4-L10), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

4:import {State, User} from "@src/SizeStorage.sol";	// @audit-issue
5:
6:import {AccountingLibrary} from "@src/libraries/AccountingLibrary.sol";
7:import {Errors} from "@src/libraries/Errors.sol";
8:import {Events} from "@src/libraries/Events.sol";
9:import {CreditPosition, DebtPosition, LoanLibrary, RESERVED_ID} from "@src/libraries/LoanLibrary.sol";
10:import {Math, PERCENT} from "@src/libraries/Math.sol";
11:import {BorrowOffer, OfferLibrary} from "@src/libraries/OfferLibrary.sol";
12:
13:import {RiskLibrary} from "@src/libraries/RiskLibrary.sol";
14:import {VariablePoolBorrowRateParams} from "@src/libraries/YieldCurveLibrary.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L4-L14), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

4:import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";	// @audit-issue
5:import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
6:import {IWETH} from "@src/interfaces/IWETH.sol";
7:import {CapsLibrary} from "@src/libraries/CapsLibrary.sol";
8:
9:import {State} from "@src/SizeStorage.sol";
10:
11:import {DepositTokenLibrary} from "@src/libraries/DepositTokenLibrary.sol";
12:
13:import {Errors} from "@src/libraries/Errors.sol";
14:import {Events} from "@src/libraries/Events.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L4-L14), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

4:import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";	// @audit-issue
5:import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
6:
7:import {Errors} from "@src/libraries/Errors.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L4-L7), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

4:import {IPool} from "@aave/interfaces/IPool.sol";	// @audit-issue
5:import {WadRayMath} from "@aave/protocol/libraries/math/WadRayMath.sol";
6:import {IERC20Metadata} from "@openzeppelin/contracts/interfaces/IERC20Metadata.sol";
7:
8:import {Math} from "@src/libraries/Math.sol";
9:import {NonTransferrableToken} from "@src/token/NonTransferrableToken.sol";
10:
11:import {Errors} from "@src/libraries/Errors.sol";
```
[4](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L4-L11), 


#### Recommendation

Adopt a modular file structure in your Solidity projects by defining only one library, interface, or contract per file. This approach significantly enhances the clarity and readability of your code, making it easier to manage, test, and review. It also simplifies version control, as updates to individual components are isolated to their respective files, reducing the risk of unintended side effects. Organize your project directory to reflect this structure, with a clear naming convention that matches file names to their contained contracts, libraries, or interfaces. This organization not only prevents naming collisions but also facilitates smoother integration into larger systems or decentralized applications (DApps).

### Reduce deployment costs by tweaking contracts' metadata
When solidity generates the bytecode for the smart contract to be deployed, it appends metadata about the compilation at the end of the bytecode.
By default, the solidity compiler appends metadata at the end of the “actual” initcode, which gets stored to the blockchain when the constructor finishes executing.
Consider tweaking the metadata to avoid this unnecessary allocation. A full guide can be found [here](https://www.rareskills.io/post/solidity-metadata).
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

See [this](https://www.rareskills.io/post/solidity-metadata) link, at its bottom, for full details

### All verbatim blocks are considered identical by deduplicator and can incorrectly be unified
The Solidity Team reported a bug on October 24, 2023, affecting Yul code using the verbatim builtin, specifically in the Block Deduplicator optimizer step. This bug, present since Solidity version 0.8.5, caused incorrect deduplication of verbatim assembly items surrounded by identical opcodes, considering them identical regardless of their data. The bug was confined to pure Yul compilation with optimization enabled and was unlikely to be exploited as an attack vector. The conditions triggering the bug were very specific, and its occurrence was deemed to have a low likelihood. The bug was rated with an overall low score due to these factors.
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

Review and assess any Solidity contracts, especially those involving Yul code, that may be impacted by this deduplication bug. If your contracts rely on the Block Deduplicator optimizer and use verbatim blocks in a way that could be affected by this issue, consider updating your Solidity version to one where this bug is fixed, or adjust your contract to avoid this specific scenario. Stay informed about updates from the Solidity Team regarding this and similar issues, and regularly update your Solidity compiler to the latest version to benefit from bug fixes and optimizations. Given the specific and limited nature of this bug, its impact may be minimal, but caution is advised for contracts with complex assembly code or those heavily reliant on optimizer behaviors.

### Consider adding formal verification proofs
Formal verification is the act of proving or disproving the correctness of intended algorithms underlying a system with respect to a certain formal specification/property/invariant, using formal methods of mathematics.
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

Consider integrating formal verification into your Solidity contract development process. This can be done by defining formal specifications and properties that your contract should adhere to and using mathematical methods to verify these aspects. Tools and platforms like Certora Prover, Scribble, or OpenZeppelin's test environment can assist in this process. Formal verification should complement traditional testing and auditing methods, offering an additional layer of security assurance. Keep in mind that formal verification requires a thorough understanding of mathematical logic and contract specifications, so it may necessitate additional resources or expertise. Nevertheless, the investment in formal verification can significantly enhance the trustworthiness and robustness of your smart contracts.

### Contracts should have full test coverage
While 100% code coverage does not guarantee that there are no bugs, it often will catch easy-to-find bugs, and will ensure that there are fewer regressions when the code invariably has to be modified. Furthermore, in order to get full coverage, code authors will often have to re-organize their code so that it is more modular, so that each component can be tested separately, which reduces interdependencies between modules and layers, and makes for code that is easier to reason about and audit.
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

Consider writing test cases.

### Large or complicated code bases should implement invariant tests
This includes: large code bases, or code with lots of inline-assembly, complicated math, or complicated interactions between multiple contracts. Invariant fuzzers such as Echidna require the test writer to come up with invariants which should not be violated under any circumstances, and the fuzzer tests various inputs and function calls to ensure that the invariants always hold. Even code with 100% code coverage can still have bugs due to the order of the operations a user performs, and invariant fuzzers may help significantly.
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

Consider writing invariant test cases.

### Consider adding formal verification proofs
Consider using formal verification to mathematically prove that your code does what is intended, and does not have any edge cases with unexpected behavior. The solidity compiler itself has this functionality [built in](https://docs.soliditylang.org/en/latest/smtchecker.html#smtchecker-and-formal-verification)
```solidity
/// @audit Global finding.
```
[1](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L1), 


#### Recommendation

Consider using formal verification.

### NatSpec: Whitespace in Expressions
See the [Whitespace in Expressions](https://docs.soliditylang.org/en/latest/style-guide.html#whitespace-in-expressions) section of the Solidity Style Guide.


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

125:            uint256 y0 =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

131:                uint256 y1 =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[125](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L125-L125), [131](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L131-L131), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

69:        DebtPosition memory debtPosition =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

118:            CreditPosition memory creditPosition =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[69](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L69-L69), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L118-L118), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

31:            uint256 debtATokenSupplyDecrease =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L31-L31), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

54:        IAToken aToken =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

77:        IAToken aToken =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L54-L54), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L77-L77), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

43:        CreditPosition storage creditPositionWithDebtToRepay =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

45:        DebtPosition storage debtPositionToRepay =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

64:            CreditPosition storage creditPositionToCompensate =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

66:            DebtPosition storage debtPositionToCompensate =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

111:        CreditPosition storage creditPositionWithDebtToRepay =	// @audit-issue: Variable declaration should be like `uint256 x = 3`

113:        DebtPosition storage debtPositionToRepay =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L43-L43), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L45-L45), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L64-L64), [66](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L66-L66), [111](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L111-L111), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L113-L113), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

107:            uint256 collateralRemainderCap =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L107-L107), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

30:        LoanOffer memory loanOffer =	// @audit-issue: Variable declaration should be like `uint256 x = 3`
```
[30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L30-L30), 


#### Recommendation

Review and align your Solidity code with the whitespace guidelines provided in the Solidity Style Guide, particularly for expressions. Ensure consistent spacing around operators and commas, and adopt a uniform style for indentation and line breaks in complex expressions. This alignment not only enhances the readability of your code but also promotes a standardized coding style that aligns with community best practices. Utilize linters or automated code formatting tools, where possible, to enforce these whitespace guidelines consistently across your codebase.

### NatSpec: Body of `if` statement should be placed on a new line
According to the [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html#control-structures), `if` statements whose body contains a single line should look like this: solidity `if (x < 10)     x += 1; `

```solidity
Path: ./src/oracle/PriceFeed.sol

88:        if (price <= 0) revert Errors.INVALID_PRICE(address(aggregator), price);	// @audit-issue
```
[88](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L88-L88), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

53:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue

81:        if (tenor == 0) revert Errors.NULL_TENOR();	// @audit-issue
```
[53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L53-L53), [81](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L81-L81), 


#### Recommendation

Adhere to the Solidity style guide by formatting single-line `if` statements with the body on the same line as the condition. For example, use `if (x < 10) x += 1;` for concise and clear code. This style of formatting enhances readability and maintains consistency with Solidity's recommended best practices. It's particularly effective for simple and short conditional operations within the contract's code.

### NatSpec: Structs, enums, events and errors should be named using CapWords style
According to the Solidity [style guide](https://docs.soliditylang.org/en/latest/style-guide.html#event-names) event,struct, error and enum names should be in CapWords style (CamelCase)

```solidity
Path: ./src/libraries/Errors.sol

10:    error USER_IS_UNDERWATER(address account, uint256 cr);	// @audit-issue

11:    error NULL_ADDRESS();	// @audit-issue

12:    error NULL_AMOUNT();	// @audit-issue

13:    error NULL_TENOR();	// @audit-issue

14:    error NULL_MAX_DUE_DATE();	// @audit-issue

15:    error NULL_ARRAY();	// @audit-issue

16:    error NULL_OFFER();	// @audit-issue

17:    error INVALID_MSG_VALUE(uint256 value);	// @audit-issue

18:    error TENORS_NOT_STRICTLY_INCREASING();	// @audit-issue

19:    error ARRAY_LENGTHS_MISMATCH();	// @audit-issue

20:    error INVALID_TOKEN(address token);	// @audit-issue

21:    error INVALID_KEY(string key);	// @audit-issue

22:    error INVALID_COLLATERAL_RATIO(uint256 cr);	// @audit-issue

23:    error INVALID_COLLATERAL_PERCENTAGE_PREMIUM(uint256 percentage);	// @audit-issue

24:    error INVALID_MAXIMUM_TENOR(uint256 maxTenor);	// @audit-issue

25:    error VALUE_GREATER_THAN_MAX(uint256 value, uint256 max);	// @audit-issue

26:    error INVALID_LIQUIDATION_COLLATERAL_RATIO(uint256 crOpening, uint256 crLiquidation);	// @audit-issue

27:    error PAST_DEADLINE(uint256 deadline);	// @audit-issue

28:    error PAST_MAX_DUE_DATE(uint256 maxDueDate);	// @audit-issue

29:    error APR_LOWER_THAN_MIN_APR(uint256 apr, uint256 minAPR);	// @audit-issue

30:    error APR_GREATER_THAN_MAX_APR(uint256 apr, uint256 maxAPR);	// @audit-issue

31:    error DUE_DATE_NOT_COMPATIBLE(uint256 dueDate1, uint256 dueDate2);	// @audit-issue

32:    error DUE_DATE_GREATER_THAN_MAX_DUE_DATE(uint256 dueDate, uint256 maxDueDate);	// @audit-issue

33:    error TENOR_OUT_OF_RANGE(uint256 tenor, uint256 minTenor, uint256 maxTenor);	// @audit-issue

34:    error INVALID_POSITION_ID(uint256 positionId);	// @audit-issue

35:    error INVALID_DEBT_POSITION_ID(uint256 debtPositionId);	// @audit-issue

36:    error INVALID_CREDIT_POSITION_ID(uint256 creditPositionId);	// @audit-issue

37:    error INVALID_LENDER(address account);	// @audit-issue

38:    error INVALID_LOAN_OFFER(address lender);	// @audit-issue

39:    error INVALID_BORROW_OFFER(address borrower);	// @audit-issue

41:    error CREDIT_NOT_FOR_SALE(uint256 creditPositionId);	// @audit-issue

42:    error NOT_ENOUGH_CREDIT(uint256 credit, uint256 required);	// @audit-issue

43:    error NOT_ENOUGH_CASH(uint256 cash, uint256 required);	// @audit-issue

45:    error BORROWER_IS_NOT_LENDER(address borrower, address lender);	// @audit-issue

46:    error COMPENSATOR_IS_NOT_BORROWER(address compensator, address borrower);	// @audit-issue

47:    error LIQUIDATOR_IS_NOT_LENDER(address liquidator, address lender);	// @audit-issue

49:    error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);	// @audit-issue

50:    error NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(uint256 liquidity, uint256 required);	// @audit-issue

51:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue

52:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT_OPENING(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue

54:    error CREDIT_POSITION_ALREADY_CLAIMED(uint256 positionId);	// @audit-issue

56:    error CREDIT_POSITION_NOT_TRANSFERRABLE(uint256 creditPositionId, LoanStatus status, uint256 borrowerCR);	// @audit-issue

58:    error LOAN_ALREADY_REPAID(uint256 positionId);	// @audit-issue

59:    error LOAN_NOT_REPAID(uint256 positionId);	// @audit-issue

60:    error LOAN_NOT_ACTIVE(uint256 positionId);	// @audit-issue

62:    error LOAN_NOT_LIQUIDATABLE(uint256 debtPositionId, uint256 cr, LoanStatus status);	// @audit-issue

63:    error LOAN_NOT_SELF_LIQUIDATABLE(uint256 creditPositionId, uint256 cr, LoanStatus status);	// @audit-issue

64:    error LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(	// @audit-issue

67:    error CR_BELOW_OPENING_LIMIT_BORROW_CR(address account, uint256 cr, uint256 riskCollateralRatio);	// @audit-issue

68:    error LIQUIDATION_NOT_AT_LOSS(uint256 positionId, uint256 cr);	// @audit-issue

70:    error INVALID_DECIMALS(uint8 decimals);	// @audit-issue

71:    error INVALID_PRICE(address aggregator, int256 price);	// @audit-issue

72:    error STALE_PRICE(address aggregator, uint256 updatedAt);	// @audit-issue

73:    error NULL_STALE_PRICE();	// @audit-issue

74:    error NULL_STALE_RATE();	// @audit-issue

75:    error STALE_RATE(uint128 updatedAt);	// @audit-issue

77:    error BORROW_ATOKEN_INCREASE_EXCEEDS_DEBT_TOKEN_DECREASE(uint256 borrowATokenIncrease, uint256 debtTokenDecrease);	// @audit-issue

78:    error BORROW_ATOKEN_CAP_EXCEEDED(uint256 cap, uint256 amount);	// @audit-issue

80:    error NOT_SUPPORTED();	// @audit-issue

82:    error SEQUENCER_DOWN();	// @audit-issue

83:    error GRACE_PERIOD_NOT_OVER();	// @audit-issue
```
[10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L12-L12), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L14-L14), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L15-L15), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L16-L16), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L17-L17), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L18-L18), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L19-L19), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L25-L25), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L27-L27), [28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L28-L28), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L30-L30), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L31-L31), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L33-L33), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L34-L34), [35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L35-L35), [36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L36-L36), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L39-L39), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L41-L41), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L42-L42), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L43-L43), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L45-L45), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L46-L46), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L47-L47), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L49-L49), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L50-L50), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L52-L52), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L54-L54), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L56-L56), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L58-L58), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L59-L59), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L60-L60), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L62-L62), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L63-L63), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L64-L64), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L67-L67), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L68-L68), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L70-L70), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L71-L71), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L73-L73), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L74-L74), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L75-L75), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L77-L77), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L78-L78), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L80-L80), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L82-L82), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L83-L83), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Contract declarations should have `@dev` tag
NatSpec comments are a critical part of Solidity's documentation system, designed to help developers and others understand the behavior and purpose of a contract. The `@dev` tag, in particular, provides context and insight into the contract's development considerations. A missing `@dev` comment can lead to misunderstandings about the contract, making it harder for others to contribute to or use the contract effectively. Therefore, it's highly recommended to include `@dev` comments in the documentation to enhance code readability and maintainability. [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/Size.sol

62:contract Size is ISize, SizeView, Initializable, AccessControlUpgradeable, PausableUpgradeable, UUPSUpgradeable {	// @audit-issue missing `@dev` tag
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L62-L62), 


```solidity
Path: ./src/SizeView.sol

37:abstract contract SizeView is SizeStorage, ISizeView {	// @audit-issue missing `@dev` tag
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L37-L37), 


```solidity
Path: ./src/oracle/IPriceFeed.sol

7:interface IPriceFeed {	// @audit-issue missing `@dev` tag
```
[7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L7-L7), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

14:library RiskLibrary {	// @audit-issue missing `@dev` tag
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L14-L14), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

26:library OfferLibrary {	// @audit-issue missing `@dev` tag
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L26-L26), 


```solidity
Path: ./src/libraries/Errors.sol

9:library Errors {	// @audit-issue missing `@dev` tag
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L9-L9), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

56:library LoanLibrary {	// @audit-issue missing `@dev` tag
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L56-L56), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

16:library AccountingLibrary {	// @audit-issue missing `@dev` tag
```
[16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L16-L16), 


```solidity
Path: ./src/libraries/CapsLibrary.sol

11:library CapsLibrary {	// @audit-issue missing `@dev` tag
```
[11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/CapsLibrary.sol#L11-L11), 


```solidity
Path: ./src/libraries/Events.sol

15:library Events {	// @audit-issue missing `@dev` tag
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L15-L15), 


```solidity
Path: ./src/libraries/Math.sol

14:library Math {	// @audit-issue missing `@dev` tag
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L14-L14), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

25:library SetUserConfiguration {	// @audit-issue missing `@dev` tag
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L25-L25), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

24:library SelfLiquidate {	// @audit-issue missing `@dev` tag
```
[24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L24-L24), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

31:library Compensate {	// @audit-issue missing `@dev` tag
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

40:library SellCreditMarket {	// @audit-issue missing `@dev` tag
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

29:library Liquidate {	// @audit-issue missing `@dev` tag
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L29-L29), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

36:library LiquidateWithReplacement {	// @audit-issue missing `@dev` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L36-L36), 


```solidity
Path: ./src/libraries/actions/Claim.sol

23:library Claim {	// @audit-issue missing `@dev` tag
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L23-L23), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

26:library Withdraw {	// @audit-issue missing `@dev` tag
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L26-L26), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

19:library SellCreditLimit {	// @audit-issue missing `@dev` tag
```
[19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L19-L19), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

23:library BuyCreditLimit {	// @audit-issue missing `@dev` tag
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L23-L23), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

40:library BuyCreditMarket {	// @audit-issue missing `@dev` tag
```
[40](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L40-L40), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

29:library Deposit {	// @audit-issue missing `@dev` tag
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L29-L29), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Contract declarations should have `@notice` tag
The `@notice` is used to explain to users what the contract does. The compiler interprets `///` or `/**` comments [as this tag](https://docs.soliditylang.org/en/latest/natspec-format.html#tags) if one wasn't explicitly provided.

```solidity
Path: ./src/oracle/IPriceFeed.sol

7:interface IPriceFeed {	// @audit-issue missing `@notice` tag
```
[7](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L7-L7), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

14:library RiskLibrary {	// @audit-issue missing `@notice` tag
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L14-L14), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

26:library OfferLibrary {	// @audit-issue missing `@notice` tag
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L26-L26), 


```solidity
Path: ./src/libraries/Errors.sol

9:library Errors {	// @audit-issue missing `@notice` tag
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L9-L9), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

56:library LoanLibrary {	// @audit-issue missing `@notice` tag
```
[56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L56-L56), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

16:library AccountingLibrary {	// @audit-issue missing `@notice` tag
```
[16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L16-L16), 


```solidity
Path: ./src/libraries/Events.sol

15:library Events {	// @audit-issue missing `@notice` tag
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L15-L15), 


```solidity
Path: ./src/libraries/Math.sol

14:library Math {	// @audit-issue missing `@notice` tag
```
[14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L14-L14), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

25:library SetUserConfiguration {	// @audit-issue missing `@notice` tag
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L25-L25), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Contract declarations should have `@title` tag
The `@title` is used to explain to users what the contract does. The compiler interprets `///` or `/**` comments [as this tag](https://docs.soliditylang.org/en/latest/natspec-format.html#tags) if one wasn't explicitly provided.

```solidity
Path: ./src/libraries/Multicall.sol

20:library Multicall {	// @audit-issue missing `@title` tag
```
[20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L20-L20), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Use `@inheritdoc` for overriden functions.
Natural Specification (NatSpec) comments are crucial for understanding the role of function arguments in your Solidity code. Including @param tags will not only improve your code's readability but also its maintainability by clearly defining each argument's purpose. [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/Size.sol

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue missing `@inheritdoc` tag
```
[107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@inheritdoc` tag

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@inheritdoc` tag

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue missing `@inheritdoc` tag

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue missing `@inheritdoc` tag

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue missing `@inheritdoc` tag
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue missing `@inheritdoc` tag

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue missing `@inheritdoc` tag

76:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@inheritdoc` tag

105:    function balanceOf(address account) public view override returns (uint256) {	// @audit-issue missing `@inheritdoc` tag

117:    function totalSupply() public view override returns (uint256) {	// @audit-issue missing `@inheritdoc` tag
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L76-L76), [105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L105-L105), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L117-L117), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Function `@return` tag is missing
Natural Specification (NatSpec) comments are crucial for understanding the role of function arguments in your Solidity code. Including `@return` tag will not only improve your code's readability but also its maintainability by clearly defining each argument's purpose. [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/Size.sol

142:    function multicall(bytes[] calldata _data)	// @audit-issue missing `@return` tag

210:    function liquidate(LiquidateParams calldata params)	// @audit-issue missing `@return` tag

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)	// @audit-issue missing `@return` tag
```
[142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L142), [210](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L210-L210), [229](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L229), 


```solidity
Path: ./src/SizeView.sol

48:    function collateralRatio(address user) external view returns (uint256) {	// @audit-issue missing `@return` tag

53:    function isUserUnderwater(address user) external view returns (bool) {	// @audit-issue missing `@return` tag

58:    function isDebtPositionLiquidatable(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@return` tag

63:    function debtTokenAmountToCollateralTokenAmount(uint256 borrowATokenAmount) external view returns (uint256) {	// @audit-issue missing `@return` tag

68:    function feeConfig() external view returns (InitializeFeeConfigParams memory) {	// @audit-issue missing `@return` tag

73:    function riskConfig() external view returns (InitializeRiskConfigParams memory) {	// @audit-issue missing `@return` tag

78:    function oracle() external view returns (InitializeOracleParams memory) {	// @audit-issue missing `@return` tag

83:    function data() external view returns (DataView memory) {	// @audit-issue missing `@return` tag

97:    function getUserView(address user) external view returns (UserView memory) {	// @audit-issue missing `@return` tag

108:    function isDebtPositionId(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@return` tag

113:    function isCreditPositionId(uint256 creditPositionId) external view returns (bool) {	// @audit-issue missing `@return` tag

118:    function getDebtPosition(uint256 debtPositionId) external view returns (DebtPosition memory) {	// @audit-issue missing `@return` tag

123:    function getCreditPosition(uint256 creditPositionId) external view returns (CreditPosition memory) {	// @audit-issue missing `@return` tag

128:    function getLoanStatus(uint256 positionId) external view returns (LoanStatus) {	// @audit-issue missing `@return` tag

133:    function getPositionsCount() external view returns (uint256, uint256) {	// @audit-issue missing `@return` tag

141:    function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@return` tag

157:    function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@return` tag

173:    function getDebtPositionAssignedCollateral(uint256 debtPositionId) external view returns (uint256) {	// @audit-issue missing `@return` tag

179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {	// @audit-issue missing `@return` tag
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L48-L48), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L53-L53), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L58-L58), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L63-L63), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L68-L68), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L73-L73), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L78-L78), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L83-L83), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L97-L97), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L108-L108), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L113-L113), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L118-L118), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L123-L123), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L128-L128), [133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L133-L133), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L141-L141), [157](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L157-L157), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L173-L173), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L179-L179), 


```solidity
Path: ./src/oracle/IPriceFeed.sol

9:    function getPrice() external view returns (uint256);	// @audit-issue missing `@return` tag

11:    function decimals() external view returns (uint256);	// @audit-issue missing `@return` tag
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L11-L11), 


```solidity
Path: ./src/oracle/PriceFeed.sol

63:    function getPrice() external view returns (uint256) {	// @audit-issue missing `@return` tag

84:    function _getPrice(AggregatorV3Interface aggregator, uint256 stalePriceInterval) internal view returns (uint256) {	// @audit-issue missing `@return` tag
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L63-L63), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L84-L84), 


```solidity
Path: ./src/libraries/RiskLibrary.sol

121:    function isUserUnderwater(State storage state, address account) public view returns (bool) {	// @audit-issue missing `@return` tag
```
[121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/RiskLibrary.sol#L121-L121), 


```solidity
Path: ./src/libraries/Multicall.sol

26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {	// @audit-issue missing `@return` tag
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L26-L26), 


```solidity
Path: ./src/libraries/Math.sol

15:    function min(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@return` tag

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@return` tag

23:    function mulDivUp(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@return` tag

27:    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@return` tag

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue missing `@return` tag
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L15-L15), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L23-L23), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L27-L27), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

127:    function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)	// @audit-issue missing `@return` tag
```
[127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L127-L127), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@return` tag

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@return` tag

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue missing `@return` tag

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue missing `@return` tag

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue missing `@return` tag
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Function declarations should have `@notice` tag
The `@notice` tag in NatSpec comments is used to provide important explanations to end users about what a function does. It appears that this contract's function declarations are missing `@notice` tags in their NatSpec annotations.

The absence of `@notice` tags reduces the contract's transparency and could lead to misunderstandings about a function's purpose and behavior.  [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/Size.sol

83:    constructor() {	// @audit-issue missing `@notice` tag

87:    function initialize(	// @audit-issue missing `@notice` tag

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue missing `@notice` tag

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue missing `@notice` tag

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue missing `@notice` tag

132:    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue missing `@notice` tag

137:    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue missing `@notice` tag

142:    function multicall(bytes[] calldata _data)	// @audit-issue missing `@notice` tag

153:    function deposit(DepositParams calldata params) public payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

159:    function withdraw(WithdrawParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

166:    function buyCreditLimit(BuyCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

172:    function sellCreditLimit(SellCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

178:    function buyCreditMarket(BuyCreditMarketParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

188:    function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

198:    function repay(RepayParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

204:    function claim(ClaimParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

210:    function liquidate(LiquidateParams calldata params)	// @audit-issue missing `@notice` tag

223:    function selfLiquidate(SelfLiquidateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)	// @audit-issue missing `@notice` tag

247:    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@notice` tag

254:    function setUserConfiguration(SetUserConfigurationParams calldata params)	// @audit-issue missing `@notice` tag
```
[83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L83-L83), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L87), [107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L110), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L120), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L132-L132), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L137-L137), [142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L142), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L153-L153), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L159-L159), [166](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L166-L166), [172](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L172-L172), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L178-L178), [188](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L188-L188), [198](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L198-L198), [204](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L204-L204), [210](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L210-L210), [223](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L223-L223), [229](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L229), [247](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L247-L247), [254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L254-L254), 


```solidity
Path: ./src/SizeView.sol

48:    function collateralRatio(address user) external view returns (uint256) {	// @audit-issue missing `@notice` tag

53:    function isUserUnderwater(address user) external view returns (bool) {	// @audit-issue missing `@notice` tag

58:    function isDebtPositionLiquidatable(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@notice` tag

63:    function debtTokenAmountToCollateralTokenAmount(uint256 borrowATokenAmount) external view returns (uint256) {	// @audit-issue missing `@notice` tag

68:    function feeConfig() external view returns (InitializeFeeConfigParams memory) {	// @audit-issue missing `@notice` tag

73:    function riskConfig() external view returns (InitializeRiskConfigParams memory) {	// @audit-issue missing `@notice` tag

78:    function oracle() external view returns (InitializeOracleParams memory) {	// @audit-issue missing `@notice` tag

83:    function data() external view returns (DataView memory) {	// @audit-issue missing `@notice` tag

97:    function getUserView(address user) external view returns (UserView memory) {	// @audit-issue missing `@notice` tag

108:    function isDebtPositionId(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@notice` tag

113:    function isCreditPositionId(uint256 creditPositionId) external view returns (bool) {	// @audit-issue missing `@notice` tag

118:    function getDebtPosition(uint256 debtPositionId) external view returns (DebtPosition memory) {	// @audit-issue missing `@notice` tag

123:    function getCreditPosition(uint256 creditPositionId) external view returns (CreditPosition memory) {	// @audit-issue missing `@notice` tag

128:    function getLoanStatus(uint256 positionId) external view returns (LoanStatus) {	// @audit-issue missing `@notice` tag

133:    function getPositionsCount() external view returns (uint256, uint256) {	// @audit-issue missing `@notice` tag

141:    function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@notice` tag

157:    function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@notice` tag

173:    function getDebtPositionAssignedCollateral(uint256 debtPositionId) external view returns (uint256) {	// @audit-issue missing `@notice` tag

179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {	// @audit-issue missing `@notice` tag
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L48-L48), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L53-L53), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L58-L58), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L63-L63), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L68-L68), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L73-L73), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L78-L78), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L83-L83), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L97-L97), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L108-L108), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L113-L113), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L118-L118), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L123-L123), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L128-L128), [133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L133-L133), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L141-L141), [157](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L157-L157), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L173-L173), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L179-L179), 


```solidity
Path: ./src/oracle/PriceFeed.sol

36:    constructor(	// @audit-issue missing `@notice` tag

63:    function getPrice() external view returns (uint256) {	// @audit-issue missing `@notice` tag

84:    function _getPrice(AggregatorV3Interface aggregator, uint256 stalePriceInterval) internal view returns (uint256) {	// @audit-issue missing `@notice` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L36-L36), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L63-L63), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L84-L84), 


```solidity
Path: ./src/libraries/Multicall.sol

26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {	// @audit-issue missing `@notice` tag
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L26-L26), 


```solidity
Path: ./src/libraries/Math.sol

15:    function min(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@notice` tag

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@notice` tag

23:    function mulDivUp(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@notice` tag

27:    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@notice` tag

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue missing `@notice` tag
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L15-L15), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L23-L23), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L27-L27), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue missing `@notice` tag
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L79), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

29:    function validateWithdraw(State storage state, WithdrawParams calldata params) external view {	// @audit-issue missing `@notice` tag

52:    function executeWithdraw(State storage state, WithdrawParams calldata params) public {	// @audit-issue missing `@notice` tag
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L29-L29), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

36:    function validateDeposit(State storage state, DepositParams calldata params) external view {	// @audit-issue missing `@notice` tag

64:    function executeDeposit(State storage state, DepositParams calldata params) public {	// @audit-issue missing `@notice` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L36-L36), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L64-L64), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue missing `@notice` tag

29:    function mint(address to, uint256 value) external virtual onlyOwner {	// @audit-issue missing `@notice` tag

33:    function burn(address from, uint256 value) external virtual onlyOwner {	// @audit-issue missing `@notice` tag

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@notice` tag

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@notice` tag

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue missing `@notice` tag

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue missing `@notice` tag

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue missing `@notice` tag
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L29-L29), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L33-L33), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

25:    constructor(	// @audit-issue missing `@notice` tag

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue missing `@notice` tag

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue missing `@notice` tag
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L25), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Function declarations should have `@dev` tag
Some functions have an incomplete NatSpec: add a `@dev` notation to describe the function to improve the code documentation.

```solidity
Path: ./src/Size.sol

83:    constructor() {	// @audit-issue missing `@dev` tag

87:    function initialize(	// @audit-issue missing `@dev` tag

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue missing `@dev` tag

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue missing `@dev` tag

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue missing `@dev` tag

132:    function pause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue missing `@dev` tag

137:    function unpause() public override(ISizeAdmin) onlyRole(PAUSER_ROLE) {	// @audit-issue missing `@dev` tag

142:    function multicall(bytes[] calldata _data)	// @audit-issue missing `@dev` tag

153:    function deposit(DepositParams calldata params) public payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

159:    function withdraw(WithdrawParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

166:    function buyCreditLimit(BuyCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

172:    function sellCreditLimit(SellCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

178:    function buyCreditMarket(BuyCreditMarketParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

188:    function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

198:    function repay(RepayParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

204:    function claim(ClaimParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

210:    function liquidate(LiquidateParams calldata params)	// @audit-issue missing `@dev` tag

223:    function selfLiquidate(SelfLiquidateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)	// @audit-issue missing `@dev` tag

247:    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@dev` tag

254:    function setUserConfiguration(SetUserConfigurationParams calldata params)	// @audit-issue missing `@dev` tag
```
[83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L83-L83), [87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L87), [107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L110), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L120), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L132-L132), [137](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L137-L137), [142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L142), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L153-L153), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L159-L159), [166](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L166-L166), [172](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L172-L172), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L178-L178), [188](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L188-L188), [198](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L198-L198), [204](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L204-L204), [210](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L210-L210), [223](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L223-L223), [229](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L229), [247](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L247-L247), [254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L254-L254), 


```solidity
Path: ./src/SizeView.sol

48:    function collateralRatio(address user) external view returns (uint256) {	// @audit-issue missing `@dev` tag

53:    function isUserUnderwater(address user) external view returns (bool) {	// @audit-issue missing `@dev` tag

58:    function isDebtPositionLiquidatable(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@dev` tag

63:    function debtTokenAmountToCollateralTokenAmount(uint256 borrowATokenAmount) external view returns (uint256) {	// @audit-issue missing `@dev` tag

68:    function feeConfig() external view returns (InitializeFeeConfigParams memory) {	// @audit-issue missing `@dev` tag

73:    function riskConfig() external view returns (InitializeRiskConfigParams memory) {	// @audit-issue missing `@dev` tag

78:    function oracle() external view returns (InitializeOracleParams memory) {	// @audit-issue missing `@dev` tag

83:    function data() external view returns (DataView memory) {	// @audit-issue missing `@dev` tag

97:    function getUserView(address user) external view returns (UserView memory) {	// @audit-issue missing `@dev` tag

108:    function isDebtPositionId(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@dev` tag

113:    function isCreditPositionId(uint256 creditPositionId) external view returns (bool) {	// @audit-issue missing `@dev` tag

118:    function getDebtPosition(uint256 debtPositionId) external view returns (DebtPosition memory) {	// @audit-issue missing `@dev` tag

123:    function getCreditPosition(uint256 creditPositionId) external view returns (CreditPosition memory) {	// @audit-issue missing `@dev` tag

128:    function getLoanStatus(uint256 positionId) external view returns (LoanStatus) {	// @audit-issue missing `@dev` tag

133:    function getPositionsCount() external view returns (uint256, uint256) {	// @audit-issue missing `@dev` tag

141:    function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@dev` tag

157:    function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@dev` tag

173:    function getDebtPositionAssignedCollateral(uint256 debtPositionId) external view returns (uint256) {	// @audit-issue missing `@dev` tag

179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {	// @audit-issue missing `@dev` tag
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L48-L48), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L53-L53), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L58-L58), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L63-L63), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L68-L68), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L73-L73), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L78-L78), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L83-L83), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L97-L97), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L108-L108), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L113-L113), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L118-L118), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L123-L123), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L128-L128), [133](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L133-L133), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L141-L141), [157](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L157-L157), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L173-L173), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L179-L179), 


```solidity
Path: ./src/oracle/IPriceFeed.sol

9:    function getPrice() external view returns (uint256);	// @audit-issue missing `@dev` tag

11:    function decimals() external view returns (uint256);	// @audit-issue missing `@dev` tag
```
[9](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L9-L9), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/IPriceFeed.sol#L11-L11), 


```solidity
Path: ./src/oracle/PriceFeed.sol

36:    constructor(	// @audit-issue missing `@dev` tag

63:    function getPrice() external view returns (uint256) {	// @audit-issue missing `@dev` tag

84:    function _getPrice(AggregatorV3Interface aggregator, uint256 stalePriceInterval) internal view returns (uint256) {	// @audit-issue missing `@dev` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L36-L36), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L63-L63), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L84-L84), 


```solidity
Path: ./src/libraries/OfferLibrary.sol

32:    function isNull(LoanOffer memory self) internal pure returns (bool) {	// @audit-issue missing `@dev` tag

39:    function isNull(BorrowOffer memory self) internal pure returns (bool) {	// @audit-issue missing `@dev` tag

48:    function getAPRByTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue missing `@dev` tag

62:    function getRatePerTenor(LoanOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue missing `@dev` tag

76:    function getAPRByTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue missing `@dev` tag

90:    function getRatePerTenor(BorrowOffer memory self, VariablePoolBorrowRateParams memory params, uint256 tenor)	// @audit-issue missing `@dev` tag
```
[32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L32-L32), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L39-L39), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L48-L48), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L62-L62), [76](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L76-L76), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/OfferLibrary.sol#L90-L90), 


```solidity
Path: ./src/libraries/LoanLibrary.sol

63:    function isDebtPositionId(State storage state, uint256 positionId) internal view returns (bool) {	// @audit-issue missing `@dev` tag

71:    function isCreditPositionId(State storage state, uint256 positionId) internal view returns (bool) {	// @audit-issue missing `@dev` tag

109:    function getDebtPositionByCreditPositionId(State storage state, uint256 creditPositionId)	// @audit-issue missing `@dev` tag

122:    function getLoanStatus(State storage state, uint256 positionId) public view returns (LoanStatus) {	// @audit-issue missing `@dev` tag

148:    function getDebtPositionAssignedCollateral(State storage state, DebtPosition memory debtPosition)	// @audit-issue missing `@dev` tag
```
[63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L63-L63), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L71-L71), [109](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L109-L109), [122](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L122-L122), [148](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/LoanLibrary.sol#L148-L148), 


```solidity
Path: ./src/libraries/YieldCurveLibrary.sol

38:    function isNull(YieldCurve memory self) internal pure returns (bool) {	// @audit-issue missing `@dev` tag
```
[38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/YieldCurveLibrary.sol#L38-L38), 


```solidity
Path: ./src/libraries/AccountingLibrary.sol

164:    function getSwapFeePercent(State storage state, uint256 tenor) internal view returns (uint256) {	// @audit-issue missing `@dev` tag

173:    function getSwapFee(State storage state, uint256 cash, uint256 tenor) internal view returns (uint256) {	// @audit-issue missing `@dev` tag

185:    function getCashAmountOut(	// @audit-issue missing `@dev` tag

228:    function getCreditAmountIn(	// @audit-issue missing `@dev` tag

274:    function getCreditAmountOut(	// @audit-issue missing `@dev` tag

311:    function getCashAmountIn(	// @audit-issue missing `@dev` tag
```
[164](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L164-L164), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L173-L173), [185](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L185-L185), [228](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L228-L228), [274](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L274-L274), [311](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/AccountingLibrary.sol#L311-L311), 


```solidity
Path: ./src/libraries/DepositTokenLibrary.sol

23:    function depositUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount) external {	// @audit-issue missing `@dev` tag

34:    function withdrawUnderlyingCollateralToken(State storage state, address from, address to, uint256 amount)	// @audit-issue missing `@dev` tag
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L23-L23), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/DepositTokenLibrary.sol#L34-L34), 


```solidity
Path: ./src/libraries/Math.sol

15:    function min(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@dev` tag

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@dev` tag

23:    function mulDivUp(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@dev` tag

27:    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@dev` tag

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue missing `@dev` tag
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L15-L15), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L23-L23), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L27-L27), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/SetUserConfiguration.sol

31:    function validateSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params)	// @audit-issue missing `@dev` tag

63:    function executeSetUserConfiguration(State storage state, SetUserConfigurationParams calldata params) external {	// @audit-issue missing `@dev` tag
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L31-L31), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SetUserConfiguration.sol#L63-L63), 


```solidity
Path: ./src/libraries/actions/SelfLiquidate.sol

34:    function validateSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external view {	// @audit-issue missing `@dev` tag

59:    function executeSelfLiquidate(State storage state, SelfLiquidateParams calldata params) external {	// @audit-issue missing `@dev` tag
```
[34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L34-L34), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SelfLiquidate.sol#L59-L59), 


```solidity
Path: ./src/libraries/actions/Compensate.sol

42:    function validateCompensate(State storage state, CompensateParams calldata params) external view {	// @audit-issue missing `@dev` tag

106:    function executeCompensate(State storage state, CompensateParams calldata params) external {	// @audit-issue missing `@dev` tag
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L42-L42), [106](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Compensate.sol#L106-L106), 


```solidity
Path: ./src/libraries/actions/SellCreditMarket.sol

51:    function validateSellCreditMarket(State storage state, SellCreditMarketParams calldata params) external view {	// @audit-issue missing `@dev` tag

127:    function executeSellCreditMarket(State storage state, SellCreditMarketParams calldata params)	// @audit-issue missing `@dev` tag
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L51-L51), [127](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditMarket.sol#L127-L127), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

42:    function feeConfigParams(State storage state) public view returns (InitializeFeeConfigParams memory) {	// @audit-issue missing `@dev` tag

56:    function riskConfigParams(State storage state) public view returns (InitializeRiskConfigParams memory) {	// @audit-issue missing `@dev` tag

70:    function oracleParams(State storage state) public view returns (InitializeOracleParams memory) {	// @audit-issue missing `@dev` tag

86:    function executeUpdateConfig(State storage state, UpdateConfigParams calldata params) external {	// @audit-issue missing `@dev` tag
```
[42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L56-L56), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L70-L70), [86](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L86-L86), 


```solidity
Path: ./src/libraries/actions/Liquidate.sol

37:    function validateLiquidate(State storage state, LiquidateParams calldata params) external view {	// @audit-issue missing `@dev` tag

59:    function validateMinimumCollateralProfit(	// @audit-issue missing `@dev` tag

75:    function executeLiquidate(State storage state, LiquidateParams calldata params)	// @audit-issue missing `@dev` tag
```
[37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L37-L37), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L59-L59), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Liquidate.sol#L75-L75), 


```solidity
Path: ./src/libraries/actions/LiquidateWithReplacement.sol

47:    function validateLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)	// @audit-issue missing `@dev` tag

99:    function validateMinimumCollateralProfit(	// @audit-issue missing `@dev` tag

120:    function executeLiquidateWithReplacement(State storage state, LiquidateWithReplacementParams calldata params)	// @audit-issue missing `@dev` tag
```
[47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L47-L47), [99](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L99-L99), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/LiquidateWithReplacement.sol#L120-L120), 


```solidity
Path: ./src/libraries/actions/Claim.sol

31:    function validateClaim(State storage state, ClaimParams calldata params) external view {	// @audit-issue missing `@dev` tag

48:    function executeClaim(State storage state, ClaimParams calldata params) external {	// @audit-issue missing `@dev` tag
```
[31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L31-L31), [48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Claim.sol#L48-L48), 


```solidity
Path: ./src/libraries/actions/Initialize.sol

62:    function validateOwner(address owner) internal pure {	// @audit-issue missing `@dev` tag

70:    function validateInitializeFeeConfigParams(InitializeFeeConfigParams memory f) internal pure {	// @audit-issue missing `@dev` tag

98:    function validateInitializeRiskConfigParams(InitializeRiskConfigParams memory r) internal pure {	// @audit-issue missing `@dev` tag

132:    function validateInitializeOracleParams(InitializeOracleParams memory o) internal view {	// @audit-issue missing `@dev` tag

146:    function validateInitializeDataParams(InitializeDataParams memory d) internal view {	// @audit-issue missing `@dev` tag

175:    function validateInitialize(	// @audit-issue missing `@dev` tag

193:    function executeInitializeFeeConfig(State storage state, InitializeFeeConfigParams memory f) internal {	// @audit-issue missing `@dev` tag

207:    function executeInitializeRiskConfig(State storage state, InitializeRiskConfigParams memory r) internal {	// @audit-issue missing `@dev` tag

222:    function executeInitializeOracle(State storage state, InitializeOracleParams memory o) internal {	// @audit-issue missing `@dev` tag

230:    function executeInitializeData(State storage state, InitializeDataParams memory d) internal {	// @audit-issue missing `@dev` tag

267:    function executeInitialize(	// @audit-issue missing `@dev` tag
```
[62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L62-L62), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L70-L70), [98](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L98-L98), [132](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L132-L132), [146](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L146-L146), [175](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L175-L175), [193](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L193-L193), [207](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L207-L207), [222](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L222-L222), [230](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L230-L230), [267](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Initialize.sol#L267-L267), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

29:    function validateWithdraw(State storage state, WithdrawParams calldata params) external view {	// @audit-issue missing `@dev` tag

52:    function executeWithdraw(State storage state, WithdrawParams calldata params) public {	// @audit-issue missing `@dev` tag
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L29-L29), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/SellCreditLimit.sol

25:    function validateSellCreditLimit(State storage state, SellCreditLimitParams calldata params) external view {	// @audit-issue missing `@dev` tag
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/SellCreditLimit.sol#L25-L25), 


```solidity
Path: ./src/libraries/actions/Repay.sol

33:    function validateRepay(State storage state, RepayParams calldata params) external view {	// @audit-issue missing `@dev` tag

46:    function executeRepay(State storage state, RepayParams calldata params) external {	// @audit-issue missing `@dev` tag
```
[33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L33-L33), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Repay.sol#L46-L46), 


```solidity
Path: ./src/libraries/actions/BuyCreditLimit.sol

29:    function validateBuyCreditLimit(State storage state, BuyCreditLimitParams calldata params) external view {	// @audit-issue missing `@dev` tag
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditLimit.sol#L29-L29), 


```solidity
Path: ./src/libraries/actions/BuyCreditMarket.sol

51:    function validateBuyCreditMarket(State storage state, BuyCreditMarketParams calldata params) external view {	// @audit-issue missing `@dev` tag

121:    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)	// @audit-issue missing `@dev` tag
```
[51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L51-L51), [121](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/BuyCreditMarket.sol#L121-L121), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

36:    function validateDeposit(State storage state, DepositParams calldata params) external view {	// @audit-issue missing `@dev` tag

64:    function executeDeposit(State storage state, DepositParams calldata params) public {	// @audit-issue missing `@dev` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L36-L36), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L64-L64), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue missing `@dev` tag

29:    function mint(address to, uint256 value) external virtual onlyOwner {	// @audit-issue missing `@dev` tag

33:    function burn(address from, uint256 value) external virtual onlyOwner {	// @audit-issue missing `@dev` tag

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@dev` tag

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@dev` tag

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue missing `@dev` tag

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue missing `@dev` tag

54:    function decimals() public view virtual override returns (uint8) {	// @audit-issue missing `@dev` tag
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L29-L29), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L33-L33), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L54-L54), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

25:    constructor(	// @audit-issue missing `@dev` tag

90:    function scaledBalanceOf(address account) public view returns (uint256) {	// @audit-issue missing `@dev` tag

105:    function balanceOf(address account) public view override returns (uint256) {	// @audit-issue missing `@dev` tag

111:    function scaledTotalSupply() public view returns (uint256) {	// @audit-issue missing `@dev` tag

117:    function totalSupply() public view override returns (uint256) {	// @audit-issue missing `@dev` tag

123:    function liquidityIndex() public view returns (uint256) {	// @audit-issue missing `@dev` tag
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L25), [90](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L90-L90), [105](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L105-L105), [111](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L111-L111), [117](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L117-L117), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L123-L123), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Function/Constructor `@param` tag is missing
Natural Specification (NatSpec) comments are crucial for understanding the role of function arguments in your Solidity code. Including @param tags will not only improve your code's readability but also its maintainability by clearly defining each argument's purpose. [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/Size.sol

87:    function initialize(	// @audit-issue missing `@param` tag

107:    function _authorizeUpgrade(address newImplementation) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}	// @audit-issue missing `@param` tag

110:    function updateConfig(UpdateConfigParams calldata params)	// @audit-issue missing `@param` tag

120:    function setVariablePoolBorrowRate(uint128 borrowRate)	// @audit-issue missing `@param` tag

142:    function multicall(bytes[] calldata _data)	// @audit-issue missing `@param` tag

153:    function deposit(DepositParams calldata params) public payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

159:    function withdraw(WithdrawParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

166:    function buyCreditLimit(BuyCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

172:    function sellCreditLimit(SellCreditLimitParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

178:    function buyCreditMarket(BuyCreditMarketParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

188:    function sellCreditMarket(SellCreditMarketParams memory params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

198:    function repay(RepayParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

204:    function claim(ClaimParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

210:    function liquidate(LiquidateParams calldata params)	// @audit-issue missing `@param` tag

223:    function selfLiquidate(SelfLiquidateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

229:    function liquidateWithReplacement(LiquidateWithReplacementParams calldata params)	// @audit-issue missing `@param` tag

247:    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {	// @audit-issue missing `@param` tag

254:    function setUserConfiguration(SetUserConfigurationParams calldata params)	// @audit-issue missing `@param` tag
```
[87](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L87-L87), [107](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L107-L107), [110](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L110-L110), [120](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L120-L120), [142](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L142-L142), [153](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L153-L153), [159](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L159-L159), [166](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L166-L166), [172](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L172-L172), [178](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L178-L178), [188](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L188-L188), [198](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L198-L198), [204](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L204-L204), [210](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L210-L210), [223](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L223-L223), [229](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L229-L229), [247](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L247-L247), [254](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/Size.sol#L254-L254), 


```solidity
Path: ./src/SizeView.sol

48:    function collateralRatio(address user) external view returns (uint256) {	// @audit-issue missing `@param` tag

53:    function isUserUnderwater(address user) external view returns (bool) {	// @audit-issue missing `@param` tag

58:    function isDebtPositionLiquidatable(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@param` tag

63:    function debtTokenAmountToCollateralTokenAmount(uint256 borrowATokenAmount) external view returns (uint256) {	// @audit-issue missing `@param` tag

97:    function getUserView(address user) external view returns (UserView memory) {	// @audit-issue missing `@param` tag

108:    function isDebtPositionId(uint256 debtPositionId) external view returns (bool) {	// @audit-issue missing `@param` tag

113:    function isCreditPositionId(uint256 creditPositionId) external view returns (bool) {	// @audit-issue missing `@param` tag

118:    function getDebtPosition(uint256 debtPositionId) external view returns (DebtPosition memory) {	// @audit-issue missing `@param` tag

123:    function getCreditPosition(uint256 creditPositionId) external view returns (CreditPosition memory) {	// @audit-issue missing `@param` tag

128:    function getLoanStatus(uint256 positionId) external view returns (LoanStatus) {	// @audit-issue missing `@param` tag

141:    function getBorrowOfferAPR(address borrower, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@param` tag

157:    function getLoanOfferAPR(address lender, uint256 tenor) external view returns (uint256) {	// @audit-issue missing `@param` tag

173:    function getDebtPositionAssignedCollateral(uint256 debtPositionId) external view returns (uint256) {	// @audit-issue missing `@param` tag

179:    function getSwapFee(uint256 cash, uint256 tenor) public view returns (uint256) {	// @audit-issue missing `@param` tag
```
[48](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L48-L48), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L53-L53), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L58-L58), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L63-L63), [97](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L97-L97), [108](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L108-L108), [113](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L113-L113), [118](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L118-L118), [123](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L123-L123), [128](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L128-L128), [141](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L141-L141), [157](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L157-L157), [173](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L173-L173), [179](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/SizeView.sol#L179-L179), 


```solidity
Path: ./src/oracle/PriceFeed.sol

36:    constructor(	// @audit-issue missing `@param` tag

84:    function _getPrice(AggregatorV3Interface aggregator, uint256 stalePriceInterval) internal view returns (uint256) {	// @audit-issue missing `@param` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L36-L36), [84](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/oracle/PriceFeed.sol#L84-L84), 


```solidity
Path: ./src/libraries/Multicall.sol

26:    function multicall(State storage state, bytes[] calldata data) internal returns (bytes[] memory results) {	// @audit-issue missing `@param` tag
```
[26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Multicall.sol#L26-L26), 


```solidity
Path: ./src/libraries/Math.sol

15:    function min(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@param` tag

19:    function max(uint256 a, uint256 b) internal pure returns (uint256) {	// @audit-issue missing `@param` tag

23:    function mulDivUp(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@param` tag

27:    function mulDivDown(uint256 x, uint256 y, uint256 z) internal pure returns (uint256) {	// @audit-issue missing `@param` tag

31:    function amountToWad(uint256 amount, uint8 decimals) internal pure returns (uint256) {	// @audit-issue missing `@param` tag
```
[15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L15-L15), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L19-L19), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L23-L23), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L27-L27), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Math.sol#L31-L31), 


```solidity
Path: ./src/libraries/actions/UpdateConfig.sol

79:    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {	// @audit-issue missing `@param` tag
```
[79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/UpdateConfig.sol#L79-L79), 


```solidity
Path: ./src/libraries/actions/Withdraw.sol

29:    function validateWithdraw(State storage state, WithdrawParams calldata params) external view {	// @audit-issue missing `@param` tag

52:    function executeWithdraw(State storage state, WithdrawParams calldata params) public {	// @audit-issue missing `@param` tag
```
[29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L29-L29), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Withdraw.sol#L52-L52), 


```solidity
Path: ./src/libraries/actions/Deposit.sol

36:    function validateDeposit(State storage state, DepositParams calldata params) external view {	// @audit-issue missing `@param` tag

64:    function executeDeposit(State storage state, DepositParams calldata params) public {	// @audit-issue missing `@param` tag
```
[36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L36-L36), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/actions/Deposit.sol#L64-L64), 


```solidity
Path: ./src/token/NonTransferrableToken.sol

18:    constructor(address owner_, string memory name_, string memory symbol_, uint8 decimals_)	// @audit-issue missing `@param` tag

29:    function mint(address to, uint256 value) external virtual onlyOwner {	// @audit-issue missing `@param` tag

33:    function burn(address from, uint256 value) external virtual onlyOwner {	// @audit-issue missing `@param` tag

37:    function transferFrom(address from, address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@param` tag

42:    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {	// @audit-issue missing `@param` tag

46:    function allowance(address, address spender) public view virtual override returns (uint256) {	// @audit-issue missing `@param` tag

50:    function approve(address, uint256) public virtual override returns (bool) {	// @audit-issue missing `@param` tag
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L18-L18), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L29-L29), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L33-L33), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L37-L37), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L42-L42), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L46-L46), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableToken.sol#L50-L50), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

25:    constructor(	// @audit-issue missing `@param` tag

42:    function mint(address, uint256) external view override onlyOwner {	// @audit-issue missing `@param` tag

56:    function burn(address, uint256) external view override onlyOwner {	// @audit-issue missing `@param` tag
```
[25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L25-L25), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L42-L42), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L56-L56), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Event declarations should have `@notice` tag
The `@notice` tag in NatSpec comments is used to provide important explanations to end users about what a event does. It appears that this contract's modifier declarations are missing `@notice` tags in their NatSpec annotations.

The absence of `@notice` tags reduces the contract's transparency and could lead to misunderstandings about a events's purpose and behavior.  [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/libraries/Events.sol

18:    event Initialize(	// @audit-issue missing `@notice` tag

21:    event Deposit(address indexed token, address indexed to, uint256 amount);	// @audit-issue missing `@notice` tag

22:    event Withdraw(address indexed token, address indexed to, uint256 amount);	// @audit-issue missing `@notice` tag

23:    event UpdateConfig(string indexed key, uint256 value);	// @audit-issue missing `@notice` tag

24:    event VariablePoolBorrowRateUpdated(uint128 indexed oldBorrowRate, uint128 indexed newBorrowRate);	// @audit-issue missing `@notice` tag

25:    event SellCreditMarket(	// @audit-issue missing `@notice` tag

33:    event SellCreditLimit(	// @audit-issue missing `@notice` tag

38:    event BuyCreditMarket(	// @audit-issue missing `@notice` tag

45:    event BuyCreditLimit(	// @audit-issue missing `@notice` tag

51:    event Repay(uint256 indexed debtPositionId);	// @audit-issue missing `@notice` tag

52:    event Claim(uint256 indexed creditPositionId, uint256 indexed debtPositionId);	// @audit-issue missing `@notice` tag

53:    event Liquidate(	// @audit-issue missing `@notice` tag

56:    event SelfLiquidate(uint256 indexed creditPositionId);	// @audit-issue missing `@notice` tag

57:    event LiquidateWithReplacement(	// @audit-issue missing `@notice` tag

60:    event Compensate(	// @audit-issue missing `@notice` tag

63:    event SetUserConfiguration(	// @audit-issue missing `@notice` tag

72:    event CreateDebtPosition(	// @audit-issue missing `@notice` tag

79:    event CreateCreditPosition(	// @audit-issue missing `@notice` tag

89:    event UpdateDebtPosition(	// @audit-issue missing `@notice` tag

92:    event UpdateCreditPosition(uint256 indexed creditPositionId, address indexed lender, uint256 credit, bool forSale);	// @audit-issue missing `@notice` tag
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L18-L18), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L25-L25), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L33-L33), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L38-L38), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L45-L45), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L52-L52), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L53-L53), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L56-L56), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L57-L57), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L60-L60), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L63-L63), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L72-L72), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L79-L79), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L89-L89), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L92-L92), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

23:    event TransferUnscaled(address indexed from, address indexed to, uint256 value);	// @audit-issue missing `@notice` tag
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L23-L23), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Event declarations should have `@dev` tag
Some events have an incomplete NatSpec: add a `@dev` notation to describe the event to improve the code documentation.

```solidity
Path: ./src/libraries/Events.sol

18:    event Initialize(	// @audit-issue missing `@dev` tag

21:    event Deposit(address indexed token, address indexed to, uint256 amount);	// @audit-issue missing `@dev` tag

22:    event Withdraw(address indexed token, address indexed to, uint256 amount);	// @audit-issue missing `@dev` tag

23:    event UpdateConfig(string indexed key, uint256 value);	// @audit-issue missing `@dev` tag

24:    event VariablePoolBorrowRateUpdated(uint128 indexed oldBorrowRate, uint128 indexed newBorrowRate);	// @audit-issue missing `@dev` tag

25:    event SellCreditMarket(	// @audit-issue missing `@dev` tag

33:    event SellCreditLimit(	// @audit-issue missing `@dev` tag

38:    event BuyCreditMarket(	// @audit-issue missing `@dev` tag

45:    event BuyCreditLimit(	// @audit-issue missing `@dev` tag

51:    event Repay(uint256 indexed debtPositionId);	// @audit-issue missing `@dev` tag

52:    event Claim(uint256 indexed creditPositionId, uint256 indexed debtPositionId);	// @audit-issue missing `@dev` tag

53:    event Liquidate(	// @audit-issue missing `@dev` tag

56:    event SelfLiquidate(uint256 indexed creditPositionId);	// @audit-issue missing `@dev` tag

57:    event LiquidateWithReplacement(	// @audit-issue missing `@dev` tag

60:    event Compensate(	// @audit-issue missing `@dev` tag

63:    event SetUserConfiguration(	// @audit-issue missing `@dev` tag

72:    event CreateDebtPosition(	// @audit-issue missing `@dev` tag

79:    event CreateCreditPosition(	// @audit-issue missing `@dev` tag

89:    event UpdateDebtPosition(	// @audit-issue missing `@dev` tag

92:    event UpdateCreditPosition(uint256 indexed creditPositionId, address indexed lender, uint256 credit, bool forSale);	// @audit-issue missing `@dev` tag
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L18-L18), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L25-L25), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L33-L33), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L38-L38), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L45-L45), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L52-L52), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L53-L53), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L56-L56), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L57-L57), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L60-L60), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L63-L63), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L72-L72), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L79-L79), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L89-L89), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L92-L92), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

23:    event TransferUnscaled(address indexed from, address indexed to, uint256 value);	// @audit-issue missing `@dev` tag
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L23-L23), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Event `@param` tag is missing
Natural Specification (NatSpec) comments are crucial for understanding the role of event arguments in your Solidity code. Including @param tags will not only improve your code's readability but also its maintainability by clearly defining each argument's purpose. [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/libraries/Events.sol

18:    event Initialize(	// @audit-issue missing `@param` tag

21:    event Deposit(address indexed token, address indexed to, uint256 amount);	// @audit-issue missing `@param` tag

22:    event Withdraw(address indexed token, address indexed to, uint256 amount);	// @audit-issue missing `@param` tag

23:    event UpdateConfig(string indexed key, uint256 value);	// @audit-issue missing `@param` tag

24:    event VariablePoolBorrowRateUpdated(uint128 indexed oldBorrowRate, uint128 indexed newBorrowRate);	// @audit-issue missing `@param` tag

25:    event SellCreditMarket(	// @audit-issue missing `@param` tag

33:    event SellCreditLimit(	// @audit-issue missing `@param` tag

38:    event BuyCreditMarket(	// @audit-issue missing `@param` tag

45:    event BuyCreditLimit(	// @audit-issue missing `@param` tag

51:    event Repay(uint256 indexed debtPositionId);	// @audit-issue missing `@param` tag

52:    event Claim(uint256 indexed creditPositionId, uint256 indexed debtPositionId);	// @audit-issue missing `@param` tag

53:    event Liquidate(	// @audit-issue missing `@param` tag

56:    event SelfLiquidate(uint256 indexed creditPositionId);	// @audit-issue missing `@param` tag

57:    event LiquidateWithReplacement(	// @audit-issue missing `@param` tag

60:    event Compensate(	// @audit-issue missing `@param` tag

63:    event SetUserConfiguration(	// @audit-issue missing `@param` tag

72:    event CreateDebtPosition(	// @audit-issue missing `@param` tag

79:    event CreateCreditPosition(	// @audit-issue missing `@param` tag

89:    event UpdateDebtPosition(	// @audit-issue missing `@param` tag

92:    event UpdateCreditPosition(uint256 indexed creditPositionId, address indexed lender, uint256 credit, bool forSale);	// @audit-issue missing `@param` tag
```
[18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L18-L18), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L25-L25), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L33-L33), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L38-L38), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L45-L45), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L52-L52), [53](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L53-L53), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L56-L56), [57](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L57-L57), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L60-L60), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L63-L63), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L72-L72), [79](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L79-L79), [89](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L89-L89), [92](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Events.sol#L92-L92), 


```solidity
Path: ./src/token/NonTransferrableScaledToken.sol

23:    event TransferUnscaled(address indexed from, address indexed to, uint256 value);	// @audit-issue missing `@param` tag
```
[23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/token/NonTransferrableScaledToken.sol#L23-L23), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Error declarations should have `@notice` tag
The `@notice` tag in NatSpec comments is used to provide important explanations to end users about what a error does. It appears that this contract's modifier declarations are missing `@notice` tags in their NatSpec annotations.

The absence of `@notice` tags reduces the contract's transparency and could lead to misunderstandings about a error's purpose and behavior.  [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/libraries/Errors.sol

10:    error USER_IS_UNDERWATER(address account, uint256 cr);	// @audit-issue missing `@notice` tag

11:    error NULL_ADDRESS();	// @audit-issue missing `@notice` tag

12:    error NULL_AMOUNT();	// @audit-issue missing `@notice` tag

13:    error NULL_TENOR();	// @audit-issue missing `@notice` tag

14:    error NULL_MAX_DUE_DATE();	// @audit-issue missing `@notice` tag

15:    error NULL_ARRAY();	// @audit-issue missing `@notice` tag

16:    error NULL_OFFER();	// @audit-issue missing `@notice` tag

17:    error INVALID_MSG_VALUE(uint256 value);	// @audit-issue missing `@notice` tag

18:    error TENORS_NOT_STRICTLY_INCREASING();	// @audit-issue missing `@notice` tag

19:    error ARRAY_LENGTHS_MISMATCH();	// @audit-issue missing `@notice` tag

20:    error INVALID_TOKEN(address token);	// @audit-issue missing `@notice` tag

21:    error INVALID_KEY(string key);	// @audit-issue missing `@notice` tag

22:    error INVALID_COLLATERAL_RATIO(uint256 cr);	// @audit-issue missing `@notice` tag

23:    error INVALID_COLLATERAL_PERCENTAGE_PREMIUM(uint256 percentage);	// @audit-issue missing `@notice` tag

24:    error INVALID_MAXIMUM_TENOR(uint256 maxTenor);	// @audit-issue missing `@notice` tag

25:    error VALUE_GREATER_THAN_MAX(uint256 value, uint256 max);	// @audit-issue missing `@notice` tag

26:    error INVALID_LIQUIDATION_COLLATERAL_RATIO(uint256 crOpening, uint256 crLiquidation);	// @audit-issue missing `@notice` tag

27:    error PAST_DEADLINE(uint256 deadline);	// @audit-issue missing `@notice` tag

28:    error PAST_MAX_DUE_DATE(uint256 maxDueDate);	// @audit-issue missing `@notice` tag

29:    error APR_LOWER_THAN_MIN_APR(uint256 apr, uint256 minAPR);	// @audit-issue missing `@notice` tag

30:    error APR_GREATER_THAN_MAX_APR(uint256 apr, uint256 maxAPR);	// @audit-issue missing `@notice` tag

31:    error DUE_DATE_NOT_COMPATIBLE(uint256 dueDate1, uint256 dueDate2);	// @audit-issue missing `@notice` tag

32:    error DUE_DATE_GREATER_THAN_MAX_DUE_DATE(uint256 dueDate, uint256 maxDueDate);	// @audit-issue missing `@notice` tag

33:    error TENOR_OUT_OF_RANGE(uint256 tenor, uint256 minTenor, uint256 maxTenor);	// @audit-issue missing `@notice` tag

34:    error INVALID_POSITION_ID(uint256 positionId);	// @audit-issue missing `@notice` tag

35:    error INVALID_DEBT_POSITION_ID(uint256 debtPositionId);	// @audit-issue missing `@notice` tag

36:    error INVALID_CREDIT_POSITION_ID(uint256 creditPositionId);	// @audit-issue missing `@notice` tag

37:    error INVALID_LENDER(address account);	// @audit-issue missing `@notice` tag

38:    error INVALID_LOAN_OFFER(address lender);	// @audit-issue missing `@notice` tag

39:    error INVALID_BORROW_OFFER(address borrower);	// @audit-issue missing `@notice` tag

41:    error CREDIT_NOT_FOR_SALE(uint256 creditPositionId);	// @audit-issue missing `@notice` tag

42:    error NOT_ENOUGH_CREDIT(uint256 credit, uint256 required);	// @audit-issue missing `@notice` tag

43:    error NOT_ENOUGH_CASH(uint256 cash, uint256 required);	// @audit-issue missing `@notice` tag

45:    error BORROWER_IS_NOT_LENDER(address borrower, address lender);	// @audit-issue missing `@notice` tag

46:    error COMPENSATOR_IS_NOT_BORROWER(address compensator, address borrower);	// @audit-issue missing `@notice` tag

47:    error LIQUIDATOR_IS_NOT_LENDER(address liquidator, address lender);	// @audit-issue missing `@notice` tag

49:    error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);	// @audit-issue missing `@notice` tag

50:    error NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(uint256 liquidity, uint256 required);	// @audit-issue missing `@notice` tag

51:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue missing `@notice` tag

52:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT_OPENING(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue missing `@notice` tag

54:    error CREDIT_POSITION_ALREADY_CLAIMED(uint256 positionId);	// @audit-issue missing `@notice` tag

56:    error CREDIT_POSITION_NOT_TRANSFERRABLE(uint256 creditPositionId, LoanStatus status, uint256 borrowerCR);	// @audit-issue missing `@notice` tag

58:    error LOAN_ALREADY_REPAID(uint256 positionId);	// @audit-issue missing `@notice` tag

59:    error LOAN_NOT_REPAID(uint256 positionId);	// @audit-issue missing `@notice` tag

60:    error LOAN_NOT_ACTIVE(uint256 positionId);	// @audit-issue missing `@notice` tag

62:    error LOAN_NOT_LIQUIDATABLE(uint256 debtPositionId, uint256 cr, LoanStatus status);	// @audit-issue missing `@notice` tag

63:    error LOAN_NOT_SELF_LIQUIDATABLE(uint256 creditPositionId, uint256 cr, LoanStatus status);	// @audit-issue missing `@notice` tag

64:    error LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(	// @audit-issue missing `@notice` tag

67:    error CR_BELOW_OPENING_LIMIT_BORROW_CR(address account, uint256 cr, uint256 riskCollateralRatio);	// @audit-issue missing `@notice` tag

68:    error LIQUIDATION_NOT_AT_LOSS(uint256 positionId, uint256 cr);	// @audit-issue missing `@notice` tag

70:    error INVALID_DECIMALS(uint8 decimals);	// @audit-issue missing `@notice` tag

71:    error INVALID_PRICE(address aggregator, int256 price);	// @audit-issue missing `@notice` tag

72:    error STALE_PRICE(address aggregator, uint256 updatedAt);	// @audit-issue missing `@notice` tag

73:    error NULL_STALE_PRICE();	// @audit-issue missing `@notice` tag

74:    error NULL_STALE_RATE();	// @audit-issue missing `@notice` tag

75:    error STALE_RATE(uint128 updatedAt);	// @audit-issue missing `@notice` tag

77:    error BORROW_ATOKEN_INCREASE_EXCEEDS_DEBT_TOKEN_DECREASE(uint256 borrowATokenIncrease, uint256 debtTokenDecrease);	// @audit-issue missing `@notice` tag

78:    error BORROW_ATOKEN_CAP_EXCEEDED(uint256 cap, uint256 amount);	// @audit-issue missing `@notice` tag

80:    error NOT_SUPPORTED();	// @audit-issue missing `@notice` tag

82:    error SEQUENCER_DOWN();	// @audit-issue missing `@notice` tag

83:    error GRACE_PERIOD_NOT_OVER();	// @audit-issue missing `@notice` tag
```
[10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L12-L12), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L14-L14), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L15-L15), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L16-L16), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L17-L17), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L18-L18), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L19-L19), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L25-L25), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L27-L27), [28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L28-L28), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L30-L30), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L31-L31), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L33-L33), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L34-L34), [35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L35-L35), [36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L36-L36), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L39-L39), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L41-L41), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L42-L42), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L43-L43), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L45-L45), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L46-L46), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L47-L47), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L49-L49), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L50-L50), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L52-L52), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L54-L54), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L56-L56), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L58-L58), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L59-L59), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L60-L60), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L62-L62), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L63-L63), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L64-L64), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L67-L67), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L68-L68), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L70-L70), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L71-L71), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L73-L73), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L74-L74), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L75-L75), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L77-L77), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L78-L78), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L80-L80), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L82-L82), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L83-L83), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Error declarations should have `@dev` tag
Some errors have an incomplete NatSpec: add a `@dev` notation to describe the error to improve the code documentation.

```solidity
Path: ./src/libraries/Errors.sol

10:    error USER_IS_UNDERWATER(address account, uint256 cr);	// @audit-issue missing `@dev` tag

11:    error NULL_ADDRESS();	// @audit-issue missing `@dev` tag

12:    error NULL_AMOUNT();	// @audit-issue missing `@dev` tag

13:    error NULL_TENOR();	// @audit-issue missing `@dev` tag

14:    error NULL_MAX_DUE_DATE();	// @audit-issue missing `@dev` tag

15:    error NULL_ARRAY();	// @audit-issue missing `@dev` tag

16:    error NULL_OFFER();	// @audit-issue missing `@dev` tag

17:    error INVALID_MSG_VALUE(uint256 value);	// @audit-issue missing `@dev` tag

18:    error TENORS_NOT_STRICTLY_INCREASING();	// @audit-issue missing `@dev` tag

19:    error ARRAY_LENGTHS_MISMATCH();	// @audit-issue missing `@dev` tag

20:    error INVALID_TOKEN(address token);	// @audit-issue missing `@dev` tag

21:    error INVALID_KEY(string key);	// @audit-issue missing `@dev` tag

22:    error INVALID_COLLATERAL_RATIO(uint256 cr);	// @audit-issue missing `@dev` tag

23:    error INVALID_COLLATERAL_PERCENTAGE_PREMIUM(uint256 percentage);	// @audit-issue missing `@dev` tag

24:    error INVALID_MAXIMUM_TENOR(uint256 maxTenor);	// @audit-issue missing `@dev` tag

25:    error VALUE_GREATER_THAN_MAX(uint256 value, uint256 max);	// @audit-issue missing `@dev` tag

26:    error INVALID_LIQUIDATION_COLLATERAL_RATIO(uint256 crOpening, uint256 crLiquidation);	// @audit-issue missing `@dev` tag

27:    error PAST_DEADLINE(uint256 deadline);	// @audit-issue missing `@dev` tag

28:    error PAST_MAX_DUE_DATE(uint256 maxDueDate);	// @audit-issue missing `@dev` tag

29:    error APR_LOWER_THAN_MIN_APR(uint256 apr, uint256 minAPR);	// @audit-issue missing `@dev` tag

30:    error APR_GREATER_THAN_MAX_APR(uint256 apr, uint256 maxAPR);	// @audit-issue missing `@dev` tag

31:    error DUE_DATE_NOT_COMPATIBLE(uint256 dueDate1, uint256 dueDate2);	// @audit-issue missing `@dev` tag

32:    error DUE_DATE_GREATER_THAN_MAX_DUE_DATE(uint256 dueDate, uint256 maxDueDate);	// @audit-issue missing `@dev` tag

33:    error TENOR_OUT_OF_RANGE(uint256 tenor, uint256 minTenor, uint256 maxTenor);	// @audit-issue missing `@dev` tag

34:    error INVALID_POSITION_ID(uint256 positionId);	// @audit-issue missing `@dev` tag

35:    error INVALID_DEBT_POSITION_ID(uint256 debtPositionId);	// @audit-issue missing `@dev` tag

36:    error INVALID_CREDIT_POSITION_ID(uint256 creditPositionId);	// @audit-issue missing `@dev` tag

37:    error INVALID_LENDER(address account);	// @audit-issue missing `@dev` tag

38:    error INVALID_LOAN_OFFER(address lender);	// @audit-issue missing `@dev` tag

39:    error INVALID_BORROW_OFFER(address borrower);	// @audit-issue missing `@dev` tag

41:    error CREDIT_NOT_FOR_SALE(uint256 creditPositionId);	// @audit-issue missing `@dev` tag

42:    error NOT_ENOUGH_CREDIT(uint256 credit, uint256 required);	// @audit-issue missing `@dev` tag

43:    error NOT_ENOUGH_CASH(uint256 cash, uint256 required);	// @audit-issue missing `@dev` tag

45:    error BORROWER_IS_NOT_LENDER(address borrower, address lender);	// @audit-issue missing `@dev` tag

46:    error COMPENSATOR_IS_NOT_BORROWER(address compensator, address borrower);	// @audit-issue missing `@dev` tag

47:    error LIQUIDATOR_IS_NOT_LENDER(address liquidator, address lender);	// @audit-issue missing `@dev` tag

49:    error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);	// @audit-issue missing `@dev` tag

50:    error NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(uint256 liquidity, uint256 required);	// @audit-issue missing `@dev` tag

51:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue missing `@dev` tag

52:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT_OPENING(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue missing `@dev` tag

54:    error CREDIT_POSITION_ALREADY_CLAIMED(uint256 positionId);	// @audit-issue missing `@dev` tag

56:    error CREDIT_POSITION_NOT_TRANSFERRABLE(uint256 creditPositionId, LoanStatus status, uint256 borrowerCR);	// @audit-issue missing `@dev` tag

58:    error LOAN_ALREADY_REPAID(uint256 positionId);	// @audit-issue missing `@dev` tag

59:    error LOAN_NOT_REPAID(uint256 positionId);	// @audit-issue missing `@dev` tag

60:    error LOAN_NOT_ACTIVE(uint256 positionId);	// @audit-issue missing `@dev` tag

62:    error LOAN_NOT_LIQUIDATABLE(uint256 debtPositionId, uint256 cr, LoanStatus status);	// @audit-issue missing `@dev` tag

63:    error LOAN_NOT_SELF_LIQUIDATABLE(uint256 creditPositionId, uint256 cr, LoanStatus status);	// @audit-issue missing `@dev` tag

64:    error LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(	// @audit-issue missing `@dev` tag

67:    error CR_BELOW_OPENING_LIMIT_BORROW_CR(address account, uint256 cr, uint256 riskCollateralRatio);	// @audit-issue missing `@dev` tag

68:    error LIQUIDATION_NOT_AT_LOSS(uint256 positionId, uint256 cr);	// @audit-issue missing `@dev` tag

70:    error INVALID_DECIMALS(uint8 decimals);	// @audit-issue missing `@dev` tag

71:    error INVALID_PRICE(address aggregator, int256 price);	// @audit-issue missing `@dev` tag

72:    error STALE_PRICE(address aggregator, uint256 updatedAt);	// @audit-issue missing `@dev` tag

73:    error NULL_STALE_PRICE();	// @audit-issue missing `@dev` tag

74:    error NULL_STALE_RATE();	// @audit-issue missing `@dev` tag

75:    error STALE_RATE(uint128 updatedAt);	// @audit-issue missing `@dev` tag

77:    error BORROW_ATOKEN_INCREASE_EXCEEDS_DEBT_TOKEN_DECREASE(uint256 borrowATokenIncrease, uint256 debtTokenDecrease);	// @audit-issue missing `@dev` tag

78:    error BORROW_ATOKEN_CAP_EXCEEDED(uint256 cap, uint256 amount);	// @audit-issue missing `@dev` tag

80:    error NOT_SUPPORTED();	// @audit-issue missing `@dev` tag

82:    error SEQUENCER_DOWN();	// @audit-issue missing `@dev` tag

83:    error GRACE_PERIOD_NOT_OVER();	// @audit-issue missing `@dev` tag
```
[10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L10-L10), [11](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L11-L11), [12](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L12-L12), [13](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L13-L13), [14](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L14-L14), [15](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L15-L15), [16](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L16-L16), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L17-L17), [18](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L18-L18), [19](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L19-L19), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L25-L25), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L27-L27), [28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L28-L28), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L30-L30), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L31-L31), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L33-L33), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L34-L34), [35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L35-L35), [36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L36-L36), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L39-L39), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L41-L41), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L42-L42), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L43-L43), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L45-L45), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L46-L46), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L47-L47), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L49-L49), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L50-L50), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L52-L52), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L54-L54), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L56-L56), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L58-L58), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L59-L59), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L60-L60), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L62-L62), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L63-L63), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L64-L64), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L67-L67), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L68-L68), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L70-L70), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L71-L71), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L72-L72), [73](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L73-L73), [74](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L74-L74), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L75-L75), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L77-L77), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L78-L78), [80](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L80-L80), [82](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L82-L82), [83](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L83-L83), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

### NatSpec: Error `@param` tag is missing
Natural Specification (NatSpec) comments are crucial for understanding the role of error arguments in your Solidity code. Including @param tags will not only improve your code's readability but also its maintainability by clearly defining each argument's purpose. [Dive Deeper into NatSpec Guidelines](https://docs.soliditylang.org/en/develop/natspec-format.html)

```solidity
Path: ./src/libraries/Errors.sol

10:    error USER_IS_UNDERWATER(address account, uint256 cr);	// @audit-issue missing `@param` tag

17:    error INVALID_MSG_VALUE(uint256 value);	// @audit-issue missing `@param` tag

20:    error INVALID_TOKEN(address token);	// @audit-issue missing `@param` tag

21:    error INVALID_KEY(string key);	// @audit-issue missing `@param` tag

22:    error INVALID_COLLATERAL_RATIO(uint256 cr);	// @audit-issue missing `@param` tag

23:    error INVALID_COLLATERAL_PERCENTAGE_PREMIUM(uint256 percentage);	// @audit-issue missing `@param` tag

24:    error INVALID_MAXIMUM_TENOR(uint256 maxTenor);	// @audit-issue missing `@param` tag

25:    error VALUE_GREATER_THAN_MAX(uint256 value, uint256 max);	// @audit-issue missing `@param` tag

26:    error INVALID_LIQUIDATION_COLLATERAL_RATIO(uint256 crOpening, uint256 crLiquidation);	// @audit-issue missing `@param` tag

27:    error PAST_DEADLINE(uint256 deadline);	// @audit-issue missing `@param` tag

28:    error PAST_MAX_DUE_DATE(uint256 maxDueDate);	// @audit-issue missing `@param` tag

29:    error APR_LOWER_THAN_MIN_APR(uint256 apr, uint256 minAPR);	// @audit-issue missing `@param` tag

30:    error APR_GREATER_THAN_MAX_APR(uint256 apr, uint256 maxAPR);	// @audit-issue missing `@param` tag

31:    error DUE_DATE_NOT_COMPATIBLE(uint256 dueDate1, uint256 dueDate2);	// @audit-issue missing `@param` tag

32:    error DUE_DATE_GREATER_THAN_MAX_DUE_DATE(uint256 dueDate, uint256 maxDueDate);	// @audit-issue missing `@param` tag

33:    error TENOR_OUT_OF_RANGE(uint256 tenor, uint256 minTenor, uint256 maxTenor);	// @audit-issue missing `@param` tag

34:    error INVALID_POSITION_ID(uint256 positionId);	// @audit-issue missing `@param` tag

35:    error INVALID_DEBT_POSITION_ID(uint256 debtPositionId);	// @audit-issue missing `@param` tag

36:    error INVALID_CREDIT_POSITION_ID(uint256 creditPositionId);	// @audit-issue missing `@param` tag

37:    error INVALID_LENDER(address account);	// @audit-issue missing `@param` tag

38:    error INVALID_LOAN_OFFER(address lender);	// @audit-issue missing `@param` tag

39:    error INVALID_BORROW_OFFER(address borrower);	// @audit-issue missing `@param` tag

41:    error CREDIT_NOT_FOR_SALE(uint256 creditPositionId);	// @audit-issue missing `@param` tag

42:    error NOT_ENOUGH_CREDIT(uint256 credit, uint256 required);	// @audit-issue missing `@param` tag

43:    error NOT_ENOUGH_CASH(uint256 cash, uint256 required);	// @audit-issue missing `@param` tag

45:    error BORROWER_IS_NOT_LENDER(address borrower, address lender);	// @audit-issue missing `@param` tag

46:    error COMPENSATOR_IS_NOT_BORROWER(address compensator, address borrower);	// @audit-issue missing `@param` tag

47:    error LIQUIDATOR_IS_NOT_LENDER(address liquidator, address lender);	// @audit-issue missing `@param` tag

49:    error NOT_ENOUGH_BORROW_ATOKEN_BALANCE(address account, uint256 balance, uint256 required);	// @audit-issue missing `@param` tag

50:    error NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(uint256 liquidity, uint256 required);	// @audit-issue missing `@param` tag

51:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue missing `@param` tag

52:    error CREDIT_LOWER_THAN_MINIMUM_CREDIT_OPENING(uint256 credit, uint256 minimumCreditBorrowAToken);	// @audit-issue missing `@param` tag

54:    error CREDIT_POSITION_ALREADY_CLAIMED(uint256 positionId);	// @audit-issue missing `@param` tag

56:    error CREDIT_POSITION_NOT_TRANSFERRABLE(uint256 creditPositionId, LoanStatus status, uint256 borrowerCR);	// @audit-issue missing `@param` tag

58:    error LOAN_ALREADY_REPAID(uint256 positionId);	// @audit-issue missing `@param` tag

59:    error LOAN_NOT_REPAID(uint256 positionId);	// @audit-issue missing `@param` tag

60:    error LOAN_NOT_ACTIVE(uint256 positionId);	// @audit-issue missing `@param` tag

62:    error LOAN_NOT_LIQUIDATABLE(uint256 debtPositionId, uint256 cr, LoanStatus status);	// @audit-issue missing `@param` tag

63:    error LOAN_NOT_SELF_LIQUIDATABLE(uint256 creditPositionId, uint256 cr, LoanStatus status);	// @audit-issue missing `@param` tag

64:    error LIQUIDATE_PROFIT_BELOW_MINIMUM_COLLATERAL_PROFIT(	// @audit-issue missing `@param` tag

67:    error CR_BELOW_OPENING_LIMIT_BORROW_CR(address account, uint256 cr, uint256 riskCollateralRatio);	// @audit-issue missing `@param` tag

68:    error LIQUIDATION_NOT_AT_LOSS(uint256 positionId, uint256 cr);	// @audit-issue missing `@param` tag

70:    error INVALID_DECIMALS(uint8 decimals);	// @audit-issue missing `@param` tag

71:    error INVALID_PRICE(address aggregator, int256 price);	// @audit-issue missing `@param` tag

72:    error STALE_PRICE(address aggregator, uint256 updatedAt);	// @audit-issue missing `@param` tag

75:    error STALE_RATE(uint128 updatedAt);	// @audit-issue missing `@param` tag

77:    error BORROW_ATOKEN_INCREASE_EXCEEDS_DEBT_TOKEN_DECREASE(uint256 borrowATokenIncrease, uint256 debtTokenDecrease);	// @audit-issue missing `@param` tag

78:    error BORROW_ATOKEN_CAP_EXCEEDED(uint256 cap, uint256 amount);	// @audit-issue missing `@param` tag
```
[10](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L10-L10), [17](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L17-L17), [20](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L20-L20), [21](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L21-L21), [22](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L22-L22), [23](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L23-L23), [24](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L24-L24), [25](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L25-L25), [26](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L26-L26), [27](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L27-L27), [28](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L28-L28), [29](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L29-L29), [30](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L30-L30), [31](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L31-L31), [32](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L32-L32), [33](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L33-L33), [34](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L34-L34), [35](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L35-L35), [36](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L36-L36), [37](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L37-L37), [38](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L38-L38), [39](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L39-L39), [41](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L41-L41), [42](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L42-L42), [43](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L43-L43), [45](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L45-L45), [46](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L46-L46), [47](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L47-L47), [49](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L49-L49), [50](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L50-L50), [51](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L51-L51), [52](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L52-L52), [54](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L54-L54), [56](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L56-L56), [58](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L58-L58), [59](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L59-L59), [60](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L60-L60), [62](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L62-L62), [63](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L63-L63), [64](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L64-L64), [67](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L67-L67), [68](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L68-L68), [70](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L70-L70), [71](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L71-L71), [72](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L72-L72), [75](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L75-L75), [77](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L77-L77), [78](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/./src/libraries/Errors.sol#L78-L78), 


#### Recommendation

Follow the official [Solidity guidelines](https://docs.soliditylang.org/en/v0.8.17/style-guide.html).

#### [L-01] `Multicall` does not allow user to deposit 2 different tokens.

```solidity
function test_my_deposit_2_tokens() public {
    vm.startPrank(alice);
    uint256 amount = 100e6;
    deal(address(usdc), alice, amount);
    IERC20Metadata(address(usdc)).approve(address(size), amount);
    vm.deal(alice, amount);
    bytes[] memory data = new bytes[](2);
    data[0] = abi.encodeCall(size.deposit, (DepositParams({token: address(usdc), amount: amount, to: alice}))); 
    data[1] = abi.encodeCall(size.deposit, (DepositParams({token: address(weth), amount: amount, to: alice})));
    bytes[] memory results = size.multicall{value: amount}(data);
}
```

A simple code like the above will revert due to this if block in `validateDeposit`
```solidity
if (msg.value != 0 && (msg.value != params.amount || params.token != address(state.data.weth))) {
    revert Errors.INVALID_MSG_VALUE(msg.value);
}
```

While the above if block may be the intention of developers to prevent users from losing `eth` transferred in `msg.value`, not allowing users to deposit 2 different tokens in one transaction using `multicall` incurs unnecessary gas costs.

## Suggested Improvement
Use a variable as to keep track of the sum of `params.amount` and at the end of the `multicall` function, revert if `sum == msg.value` does not hold true. That way, users will not lose eth sent through `msg.value`.
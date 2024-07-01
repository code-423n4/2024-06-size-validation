# LOW

# [L-1] Accumulation of allowance on every ETH deposit, due to unnecessary approval in`Deposit::executeDeposit()`

## Description
When a user calls the `executeDeposit()` function with `msg.value > 0` with an intention to deposit ETH (WETH) as collateral the following branch is executed:
```javascript
 if (msg.value > 0) {
            amount = address(this).balance;
            // slither-disable-next-line arbitrary-send-eth
            state.data.weth.deposit{value: amount}();
            emit TempDeposit(address(this));
            state.data.weth.forceApprove(address(this), amount);
            from = address(this);
        }
```
This increases the allowance `proxy => proxy` by amount, as when calling `forceApprove()` it internally calls `approve()` function which uses `_msgSender()` - which is the proxy address.

Later on the `DepositTokenLibrary::depositUnderlyingCollateralToken()` function is called, where the user's funds exchanged to WETH are deposited using `underlyingCollateralToken.safeTransferFrom(from, address(this), amount);`.  The issue is that in the simplified mock WETH implementation this logic from the WETH9 smart contract for the `transferFrom()` function is not accounted for:
```javascript
        if (
            src != msg.sender &&
            _allowances[src][msg.sender] != type(uint256).max
        ) {
            require(_allowances[src][msg.sender] >= wad);
            _allowances[src][msg.sender] -= wad;
        }
    }
```
Because `msg.sender == src`, as we call `depositUnderlyingCollateralToken()` through a proxy, the allowance is NOT decreased, but the transfer is successful.  This means that for *all* user's ETH deposits protocol's allowance is increased.
##  Impact
 This allowance does not change the fact that an owner could steal all of the user's collateral by simply deploying a library that would allow to call `transferFrom()` on the WETH contract with `msg.sender` set to the proxy contract. Essentially, it's a Low/Info finding as the allowance would be accumulating infinitely, but without any loss of funds.

## Proof of Concept
To show that the proxy => proxy allowance is indeed accumulating a proof of code was prepared. In `./test/mocks/WETH.sol` make the following change:
```diff
+function transferFrom(
+       address src,
+       address dst,
+       uint256 wad
+   ) public override returns (bool) {
+       require(_balances[src] >= wad);

+       if (
+           src != msg.sender &&
+           _allowances[src][msg.sender] != type(uint256).max
+       ) {
+           require(_allowances[src][msg.sender] >= wad);
+           _allowances[src][msg.sender] -= wad;
+       }

+       _balances[src] -= wad;
+       _balances[dst] += wad;
+       emit Transfer(src, dst, wad);
+       return true;
+   }
```
it's a simplified way to account for transfer logic from the real WETH9 contract, however for it to compile in the ERC20 contract this Mock WETH inherits `_balances` and `_allowances` mappings` visibility has to be changed:
```diff
-   mapping(address account => uint256) private_balances;
-   mapping(address account => mapping(address spender => uint256)) private _allowances;
+   mapping(address account => uint256) public _balances;
+   mapping(address account => mapping(address spender => uint256)) public _allowances;
```
now simply put the following test case in the `./test/local/actions/Deposit.t.sol` test file and run it with `forge test --mt test_accumulation_of_allowance_on_ETH_deposit -vvv`:
```
function test_accumulation_of_allowance_on_ETH_deposit() public {
        vm.deal(alice, 1 ether);

        assertEq(address(alice).balance, 1 ether);
        assertEq(_state().alice.collateralTokenBalance, 0);

        vm.prank(alice);
        size.deposit{value: 1 ether}(
            DepositParams({token: address(weth), amount: 1 ether, to: alice})
        );
        console.log(
            "proxy => proxy allowance: ",
            weth.allowance(address(proxy), address(proxy)) / 1e18,
            "ETH"
        );
        assertEq(address(alice).balance, 0);
        assertEq(_state().alice.collateralTokenBalance, 1 ether);
}
```
To prove that the funds are indeed accessible for the protocol ( which demands lack of trust for the owner ) add the following code to the test file above:
```diff
+address maliciousOwner = makeAddr("malicious");
+vm.prank(address(size));
+weth.transferFrom(address(size), address(maliciousOwner), 1 ether);
+assertEq(weth.balanceOf(maliciousOwner), 1 ether);
+assertEq(_state().alice.collateralTokenBalance, 1 ether);
```
Obviously, the `collateralTokenBalance` is not changed so as explained before the user can still withdraw their funds - unless the malicious owner left out enough WETH in the Size contract.

## Recommended Mitigation
With the appropriate changes to the mock WETH contract described in the proof of concept, it can be easily visualized how removing `forceApprove()` does not cause the deposit to fail.

In `Deposit.sol` comment-out:
```
    // state.data.weth.forceApprove(address(this),amount);
```
and run `forge test --mt test_accumulation_of_allowance_on_ETH_deposit -vvv` to see that the issue was resolved and the assertions for a successful deposit still pass.

# [L-2] User can receive an arbitrary amount of extra ETH to their deposit
## Impact
In `Deposit::executeDeposit()` function the following change to the `amount` variable is made:
```javascript
amount = address(this).balance;
```
This is the result of OpenZeppelin's suggestions for multicall implenetation: *Functions should not rely on `msg.value`.* However, this change is made **always**, not only during a multicall. This means, that in a situation where there is extra ether in the proxy contract, any user calling regular deposit function will receive arbitrary amount of extra deposit tokens. This functionality makes sense for the multicall, where deposit limits may be exceeded, and appropriate checks are done before and after multicall execution. While for the regular deposit transaction, this functionality is supposed to account for any leftover Wei, and not necessarily larger amounts of ETH.

## Recommended Mitigation
For the reasons explained above something similar to this changes should be considered in `executeDeposit()`:
```diff
+uint256 public constant LEFTOVERS = 0.01e18
...
if (msg.value > 0) {
            // do not trust msg.value a(see `Multicall.sol`)
-     amount = address(this).balance;
-           if (state.data.isMulticall) {
+               amount = address(this).balance;
+           } else {
+     if (address(this).balance <= LEFTOVERS) {
+         amount = address(this).balance;
+         }
+           }
            // slither-disable-next-line arbitrary-send-eth
            state.data.weth.deposit{value: amount}();
            state.data.weth.forceApprove(address(this), amount);
            from = address(this);
        }
...
}
```
This kind of approach would ensure that if the extra ETH is added to the deposit it is either during a multicall, or otherwise, if during a normal deposit function flow, the added ETH cannot exceed some value `LEFTOVERS`.
# L-01 LiquidateWithReplacement benefits the protocol at the expense of lenders

`LiquidateByReplacement` pays a `liquidatorProfitBorrowToken` fee to the protocol. This fee is essentially carried by lenders in the form of opportunity cost.

In a normal liquidation of an underwater borrower, lenders can claim the full `futureAmount` at the time of liquidation, and earn interest by re-lending those funds immediately, rather than having to wait for the due date. A regular liquidation also reduces risk for the lenders compared to `LiquidateByReplacement`.

This design creates a "lenders vs. the protocol" situation, where lenders are incentivized to always liquidate underwater positions the conventional way rather than allowing `executeLiquidateWithReplacement` to be triggered.

This is a fairness/design issue rather than a 'vulnerability' but probably still useful to consider so I'm submitting this as a QA report.

## Proof of Concept

This issue doesn't require a proof-of-concept. 

## Tools Used

Manual review

## Recommended Mitigation Steps

Remove the `executeLiquidateWithReplacement` functionality.

# L-02 Multicall deposits bypass the borrowAToken cap risk parameter

# Vulnerability details

## Impact
The `borrowAToken` cap is ineffective when depositing USDC via `multicall`. As a result, protocol risk can grow beyond the risk parameters set by the admin.

The issue exists because `Multicall.sol` checks `state.data.borrowAToken.balanceOf(address(this))` which is the incorrect value.

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Multicall.sol#L37

I'm rating this as low severity since it does not directly put user funds at risk.

## Proof of Concept

I created a test to reproduce this issue:

```
    function test_Multicall_multicall_always_bypasses_cap() public {
        _setPrice(1e18);
        uint256 amount = 100e6;
        uint256 cap = 100e6;
        _updateConfig("borrowATokenCap", cap);

        _deposit(alice, usdc, amount);
        // _deposit(bob, weth, 200e18);

        _mint(address(usdc), bob, amount);
        _approve(bob, address(usdc), address(size), amount);

        // attempt to reposit via multicall
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encodeCall(size.deposit, DepositParams({token: address(usdc), amount: amount, to: bob}));
        vm.prank(bob);
        size.multicall(data);

        assertGt(size.data().borrowAToken.totalSupply() , cap);        
    }
```


## Tools Used

Manual review

## Recommended Mitigation Steps

 To get the total outstanding balance of `borrowATokens`, this should be either:

```
uint256 borrowATokenSupplyBefore = state.data.borrowAToken.totalSupply();
```

Or:

```
uint256 borrowATokenSupplyBefore = state.data.underlyingBorrowToken.balanceOf(address(this));
```

## Assessed type

Invalid Validation
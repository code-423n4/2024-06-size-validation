### [G-1] Size.sol Functions Visibility Optimization
**Description**
In the Size.sol contract, several functions are currently marked as `public`, yet they are not called internally within the contract. Functions such as `pause()`, `unpause()`, and `deposit()` can be changed from `public` to `external` to optimize gas usage. This change leverages the fact that external functions are more efficient when not called from within the same contract.

**Recommended Mitigation**
Change the visibility of the listed functions from `public` to `external` to reduce gas costs and enhance performance.

``` @diff
   
   function pause() external override(ISizeAdmin) onlyRole(PAUSER_ROLE) {
    _pause();
}

function unpause() external override(ISizeAdmin) onlyRole(PAUSER_ROLE) {
    _unpause();
}

function deposit(DepositParams calldata params) external payable override(ISize) whenNotPaused {
    state.validateDeposit(params);
    state.executeDeposit(params);
}
```
###
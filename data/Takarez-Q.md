## Summary

The [updateConfig](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L110-L111) funcyion of the `size.Sol` is used to update the configuration data:

```js
/// @inheritdoc ISizeAdmin
    function updateConfig(UpdateConfigParams calldata params)
        external
        override(ISizeAdmin)
        onlyRole(DEFAULT_ADMIN_ROLE)
    {
        state.validateUpdateConfig(params);
        state.executeUpdateConfig(params);
    }
```

Whats the issue with the function? Well, the `state.validateUppdateConfig` is called withing the function which is actually meant to validate the param values whenever the function gets called, but the issue is that, from the code natpsec, its stated that the validation is done during `execution` and not actually in the `validate` function:
```js
 /// @dev Validation is done at execution
    ///      We purposefuly leave this function empty for documentation purposes
    function validateUpdateConfig(State storage, UpdateConfigParams calldata) external pure {
        // validation is done at execution
    }
```

although the function is in place for documentation purpose, there's acttually no need of calling it since its of no use, this will only increase the computation when called. 


## Recommendation

The function in place for documentation is fine, but calling it is not, so its better to be left uncalled since it does nothing.
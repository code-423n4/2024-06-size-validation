### Recipient of `liquidatorProfitCollateralToken` is inconsistant between natspec and code

In the `LiquidateWithReplacement` library, the [natspec](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/LiquidateWithReplacement.sol#L119) indicates that the recipient of the `liquidatorProfitBorrowToken` is the 'liquidator'.

However the value of `borrowAToken` is transfered to the `feeRecipient` [here](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/LiquidateWithReplacement.sol#L162).

`LiquidateWithReplacement()` is only callable by keepers, so the 'liquidator' and the `feeRecipient` will always be the same entitity, just different wallets.
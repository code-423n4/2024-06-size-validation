# [L-01] Front running market orders
`Size` defines a number of functions for market buy/sell orders. These functions take a list of orders and a target asset amount to buy or sell. They fill each order in turn until the target has been reached.

These functions provide an appealing opportunity for front running because of the near-guaranteed profit to be had. This is most easily explained with an example:

Alice wishes to buy 10 tokens. She creates a market buy order to purchase tokens first from Bob, who is selling 4 tokens at $9 each, and then from Eve, who is selling 20 tokens at $10 each.
Eve front runs this market order with a transaction that buys all 4 tokens from Bob for $9 each.
Alice’s transaction goes through, but because Bob’s inventory has been depleted, all 10 tokens are purchased from Eve at a price of $10 each. By front running, Eve gained $4.

With a market order, however, Eve’s front running is nearly risk free because she knows the market order already commits Alice to buying at the higher price.

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L178
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L188

## Recommendation
For the most part, traders will simply have to understand the risks of market orders and take care to only authorize trades they will be happy with.

That said, each order in a market order could specify a maximum quantity, e.g. “I want 10 tokens, and I’m willing to buy up to 10 from Bob but only up to 5 from Eve.” This would limit the trader’s exposure to increased prices due to front running, but it would retain the convenience and efficiency of market orders.
# LiquidateWithReplacement benefits the protocol at the expense of lenders

LiquidateByReplacement disadvantages lenders who would otherwise be able to claim the full `futureAmount` at the time of liquidation, and could lend out those funds again for interest, rather than having to wait for the original due date. This potential profit instead goes directly to the protocol in the form of the `liquidatorProfitBorrowToken` fee. This could be viewed as the protocol unfairly siphoning off profits from its users, as well as offloading some risk.

Essentially, this design creates a "lenders vs. the protocol" situation, where lenders are incentivized to always liquidate underwater positions the conventional way rather than allowing `executeLiquidateWithReplacement` to be triggered.

We're not talking about outright theft of funds here though and the stability of the protocol is not threatened, so I'm not sure this qualifies as a 'vulnerability'.

## Proof of Concept

This issue doesn't require a proof-of-concept. 

## Tools Used

Manual review

## Recommended Mitigation Steps

Remove the `executeLiquidateWithReplacement` functionality.
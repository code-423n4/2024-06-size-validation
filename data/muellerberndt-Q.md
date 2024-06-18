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
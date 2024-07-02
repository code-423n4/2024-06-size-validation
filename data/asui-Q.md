
# [QA-01]: 
A borrower can borrow way past dueDate without being liquidated and still repay. This is because [```validateRepay```](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Repay.sol#L33C1-L39C1) only checks if the loan is already repaid, this means an overdue loan can be repaid.
This alone is not a problem because, the position will probably be liquidated as soon as it cross dueDate. 

But this combined with the protocol's pause functionality can be exploited by a malicious borrower.

Also, the minimum tenor allow by protocol is 1 hour, which is very less.
This is for lenders who want to earn quickly by lending large amounts with small interests. 

Example;
* a lender sets the tenor to 5 hours and with his desired apr
* admin calls pause 
* borrower frontruns admin transaction by using multicall to borrow 10000, with minimum tenor to minimize interest rate, and withdraw the USDC
* protocol is now paused for 5 months
* 5 hours is passed loan is overdue, but the borrower cannot get liquidated 
* after 5 months protocol is unpaused
* borrower can backrun this to repay so that he is not liquidated.
* borrower sucessfully borrows 10000 for 5 months with an interest rate for 5 hourr, without being liquidated.

Note that this is different form this [issue](https://code4rena.com/audits/2024-06-size#top:~:text=In%20case%20the%20protocol%20is%20paused%2C%20the%20price%20of%20the%20collateral%20may%20change%20during%20the%20unpause%20event.%20This%20may%20cause%20unforeseen%20liquidations%2C%20among%20other%20issues), which states how positions can be affected form protocol pause, but this one is different in a sense that it is instead exploiting the pause funcitonality which is possible because of repayments allowed for overdue loans.

Validate repay should revert when a loan is overdue. 


# [QA-02]: 
Minimum position of 50e6 borrows may still be too less, which disincentivize liquidations, which at current market is approximately 1.8 dollars profit which might be less than the gas fees on mainnet. Increase to min borrow to 100e6
Also borrowers can borrow the desired amount in multiple 50 USDC debt positions using multicall, to increase his chance of being unliquidated.

# [QA-03]: 
```LiquidateWithReplacement``` can be frontrun by the borrower to change the apr and borrow with 0 interest incase keeper bots forgets to set minimum apr, because ```SellCreditLimit``` doesn't check the apr value, it checks that it is not null but 0 value can still be set by the borrower. Either check ```aprs[i] != 0``` or call ```LiquidateWithReplacement``` with flashbots.

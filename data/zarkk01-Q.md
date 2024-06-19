## Summary
1.Incorrect implementation of binary search in `Math.sol`

## Vulnerability Detail
Binary search is a Computer Science algorithm that since it's inception from John Mauchly at 1946 has a story full of wrong and buggy implementations. Joshua Bloch, one of the famous JAVA developers, discovered at around 2000s that the famous implementation of binary search (which is used in `Math.sol` of Size protocol, too) is vulnerable in the part `(high + low) / 2`. For this reason and from then, in the field of Computer Science, an implementation in which `low + (high - low) / 2` is used is the most appropriete way to do binary search.

You can read the full interesting story behind binary search in this article : https://thebittheories.com/the-curious-case-of-binary-search-the-famous-bug-that-remained-undetected-for-20-years-973e89fc212 .

## Code Snippet
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/Math.sol#L51

## Tool used
Manual Review

## Recommendation
Consider using this implementation of binary search : 
```diff
function binarySearch(uint256[] memory array, uint256 value) internal pure returns (uint256 low, uint256 high) {
        low = 0;
        high = array.length - 1;
        if (value < array[low] || value > array[high]) {
            return (type(uint256).max, type(uint256).max);
        }
        while (low <= high) {
@>            uint256 mid = low + (high-low) / 2;
            if (array[mid] == value) {
                return (mid, mid);
            } else if (array[mid] < value) {
                low = mid + 1;
            } else {
                high = mid - 1;
            }
        }
        return (high, low);
    }
```

## Summury
2.Wrong event emission on `SellCreditMarket` event

## Description 
In `SellCreditMarket.sol` contract in `executeSellCreditMarket()` function an event is emitted but with a wrong parameter. Instead of the `dueDate` which is `block.timestamp + params.tenor` of the DebtPosition , `params.tenor` is passed incorrectly. But this is not the only problem, furthermore the event is emitted with parameters that will maybe change until the end of the function call like `tenor` and `borrower`, so the event will be complete wrong.

## Code snippet 
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SellCreditMarket.sol#L131

## Fix
Consider making this change for the first issue :
```solidity
emit Events.SellCreditMarket(
            params.lender, params.creditPositionId, params.tenor, params.amount, params.tenor + block.timestamp, params.exactAmountIn
        );
```
For the second issue, move the event emission to the end with the right parameters.

## Summary
3. Wrong usage of storage variables when it will more gas-efficient to use memory

## Description
There is lot of cases where storage variables are retrieved just to make a `get` call on them which is non-sence and, of course, not so gas efficient. 

## Code snippet
For example, here a `storage` `debtPosition` is retrieved just to take the `dueDate` of it. It will be more efficient if it was `memory`.

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/SellCreditMarket.sol#L141C34-L141C46

## Fix
Check again where `memory` can be used and `storage` is non-sense.
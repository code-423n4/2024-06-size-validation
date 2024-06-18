## Summary
Incorrect implementation of binary search in `Math.sol`

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
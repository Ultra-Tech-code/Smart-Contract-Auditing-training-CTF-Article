# Naive Receiver #
There are two contract involved here 
The receiver/borrower contract which is the vulnerable contract.
and the flahloan contract
flashloan contract = NaiveReceiverLenderPool
receiver/borrower contract = FlashLoanReceiver

`pragma solidity ^0.6.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/math/SafeMath.sol";

contract FlashLoanReceiver {
    using SafeMath for uint256;
    using Address for address payable;

    address payable private pool;

    constructor(address payable poolAddress) public {
        pool = poolAddress;
    }

    // Function called by the pool during flash loan
    function receiveEther(uint256 fee) public payable {
        require(msg.sender == pool, "Sender must be pool");

        uint256 amountToBeRepaid = msg.value.add(fee);

        require(address(this).balance >= amountToBeRepaid, "Cannot borrow that much");
        
        _executeActionDuringFlashLoan();
        
        // Return funds to pool
        pool.sendValue(amountToBeRepaid);
    }

    // Internal function where the funds received are used
    function _executeActionDuringFlashLoan() internal { }

    // Allow deposits of ETH
    receive () external payable {}
}`

The vulnerability here is with the msg.sender check in the `receiveEther` function. the check was made only on the pool not the tx.origin. the `receiveEther` function is called by the `naiveReceiverLenderPool` to receive the platform fee. 

let's break it down.

we have the `naiveReceiverLenderPool `contract which have the `flashloan` function

`pragma solidity ^0.6.0;

import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Address.sol";

contract NaiveReceiverLenderPool is ReentrancyGuard {
    using SafeMath for uint256;
    using Address for address;

    uint256 private constant FIXED_FEE = 1 ether; // not the cheapest flash loan

    function fixedFee() external pure returns (uint256) {
        return FIXED_FEE;
    }

    function flashLoan(address payable borrower, uint256 borrowAmount) external nonReentrant {

        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= borrowAmount, "Not enough ETH in pool");


        require(address(borrower).isContract(), "Borrower must be a deployed contract");
        // Transfer ETH and handle control to receiver
        (bool success, ) = borrower.call{value: borrowAmount}(
            abi.encodeWithSignature(
                "receiveEther(uint256)",
                FIXED_FEE
            )
        );
        require(success, "External call failed");
        
        require(
            address(this).balance >= balanceBefore.add(FIXED_FEE),
            "Flash loan hasn't been paid back"
        );
    }

    // Allow deposits of ETH
    receive () external payable {}
}`

In the flashloan contract the receiver/borrower contract must pay a fee(1 ether)  when returning the borrowed amount.

How can we drain the "FlashLoanReceiver" contract which is the borrower contract? 

anybody can call the flashloan contract. the only check is the borrower must be a contract. 
let's say the borrower contract has 20 ether. I as an attacker can passed in the borrowers address and 0 as the amount to borrow to the `flashloan` function in the `"NaiveReceiverLenderPool"` contract. the call will passed and the fee will be deducted from the borrowers contract.

how can i drain the whole 20 ether? i can run the call multiple times.\

here is the snippet on how to drain the 20 ether
`        for (let i = 0; i < 20; i++) {
            await this.pool.flashLoan(this.receiver.address, ether("1"), {
              from: attacker,
            });
          }`

The solution to the vulnerability here is we should be sure that the person
here is the solution to the challenge [solution](https://github.com/Ultra-Tech-code/damn-vulnerable-defi/blob/ca9b3a333d388cdbde095355f27efcbfd91d81a7/test/naive-receiver/naive-receiver.challenge.js#L35)
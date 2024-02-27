## LOW-[1]If the `receiver` of `ClaimedYield` is not msg.sender, the parameters of the YieldClaimed event will be incorrect.
  
Let's take a look at the parameter definition of the YieldClaimed event.  
```solidity
    event YieldClaimed(address indexed owner, address indexed receiver, uint256 indexed yieldInIBT);
```
The parameters are respectively the initiator of YieldClaimed, the receiver of ibt, and the amount of ibt.  
The event is triggered in the _claimYield() function, but the function does not pass the receiver, causing both owner and receiver to be msg.sender when the event is triggered.
github:[[848](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L848)]
```solidity
    function _claimYield() internal returns (uint256 yieldInIBT) {
        yieldInIBT = updateYield(msg.sender);
        if (yieldInIBT == 0) {
            return 0;
        } else {
            yieldOfUserInIBT[msg.sender] = 0;
            uint256 yieldFeeInIBT = PrincipalTokenUtil._computeYieldFee(yieldInIBT, registry);
            _updateFees(yieldFeeInIBT);
            yieldInIBT -= yieldFeeInIBT;
            emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
        }
    }
```
It is recommended to pass the receiver when calling the _claimYield() function to ensure that the event is triggered correctly.
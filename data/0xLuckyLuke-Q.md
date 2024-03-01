# L-1 Addressing Maturity-Related Redemption Inconsistencies

## Summary
The function _beforeRedeem allows users to redeem shares without considering maturity, which contradicts standard behavior.

https://eips.ethereum.org/EIPS/eip-5095
"PTs mature at a precise second, but given the reactive nature of smart contracts, there can’t be an event marking maturity, because there is no guarantee of any activity at or after maturity. Emitting an event to notify of maturity in the first transaction after maturity would be imprecise and expensive. Instead, integrators are recommended to either use the first Redeem event, or to track themselves when each PT is expected to have matured."

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L805

## Vulnerability Detail
Users can initiate redemptions regardless of the maturity status of the shares. This violates the expected behavior specified in EIP-5095, which recommends redemptions only after maturity.

## Impact
Off-chain information may be inaccurate as redemptions can occur before maturity, leading to potential misinformation.
Contradicts standard behavior, potentially causing confusion among users and integrators.

## Tool used
Manual Review

## Recommendation
Implement maturity checks to ensure redemptions occur only after maturity, adhering to the EIP-5095 standard. Enhance off-chain communication to provide accurate information regarding redemption eligibility and maturity status.

# L-2 Return value of updateYield() is not validated

## Proof of Concept

```solidity
    function beforeYtTransfer(address _from, address _to) external override {
        if (msg.sender != yt) {
            revert UnauthorizedCaller();
        }
        updateYield(_from);//@audit H: doesn't check the return value
        updateYield(_to);
    }
```
## Tools Used
VScode
## Recommended Mitigation Steps
check the return value of updateYield(_from)

# I-1 Streamlining Token Handling in Flash Loan Functions
Simplified Token Handling: The _token parameter in the flashLoan function seems unnecessary since the token address (ibt) remains constant throughout the function. Removing this parameter would streamline the interface.

Efficient Loan Calculation: The maxFlashLoan function accurately determines the maximum loan amount based on the contract's token balance, returning 0 if the token address differs from ibt.

Secure Fee Computation: The flashFee function calculates fees exclusively for the designated token (ibt) and ensures safety by reverting with an AddressError if the provided token address doesn't match.

Overall: The functions demonstrate clear logic and adhere to best practices for flash loan implementations, with potential for interface simplification and efficient token handling.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L611

```diff
function flashLoan(
    IERC3156FlashBorrower _receiver,
-    address _token,
    uint256 _amount,
    bytes calldata _data
) external override returns (bool) {
-    if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();
+    if (_amount > maxFlashLoan(ibt)) revert FlashLoanExceedsMaxAmount();

-    uint256 fee = flashFee(_token, _amount);
+    uint256 fee = flashFee(ibt, _amount);
    _updateFees(fee);

    // Initiate the flash loan by lending the requested IBT amount
    IERC20(ibt).safeTransfer(address(_receiver), _amount);

    // Execute the flash loan
-    if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)
+    if (_receiver.onFlashLoan(msg.sender, ibt, _amount, fee, _data) != ON_FLASH_LOAN)
        revert FlashLoanCallbackFailed();

    // Repay the debt + fee
    IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

    return true;
}
```


# I-2 - Only `msg.sender` can redeem and withdraw
## Impact
It can reduce the level of compatiblity and integrity in different situations and with other platforms. 
## Proof of Concept
in [_beforeRedeem](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L806) and [_beforeWithdraw](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L829) functions of the `PrincipalToken` contract if `msg.sender` is not equal to the owner, it reverts.
```solidity
 function _beforeRedeem(uint256 _shares, address _owner) internal nonReentrant whenNotPaused {
        if (_owner != msg.sender) {
            revert UnauthorizedCaller();
        }
     ...
```

## Tools Used
VScode
## Recommended Mitigation Steps
It is recommended that when input address is not the same as `msg.sender` check if `msg.sender` has alloance from the `_owner` address.  

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L805
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L828
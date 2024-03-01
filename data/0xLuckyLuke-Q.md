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


# I-2 Enhancing _beforeWithdraw/_beforeRedeem Preparations:

Conditional Optimization:

The function checks _owner against msg.sender and reverts if they differ.
Permit verification should occur only when _owner is not the caller.
Refactored Logic:

Reorder conditionals to prioritize permit checking for non-owner callers.
Consolidate conditional checks for improved readability and efficiency.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L805

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L828

```diff
    if (_owner != msg.sender) {
+       // Permit check for non-owner callers
+       if (!checkPermit(_owner)) {
            revert UnauthorizedCaller();
+       }
    ...
```
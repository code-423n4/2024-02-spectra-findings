# G-1 Require Balance Check in transferFrom Function
@gJoCV3t8Sx2V_1U_-yEsMQ 

To optimize gas usage and prevent unnecessary checks in the transferFrom function, you can include a requirement to ensure that the from address has a balance of at least amount before initiating the transfer. This can help reduce gas consumption by avoiding unnecessary transactions when the sender's balance is insufficient.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L95-L102

Here's the proposed modification:

```diff
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
+       require(balanceOf(from) >= amount);
        IPrincipalToken(pt).beforeYtTransfer(from, to);
        return super.transferFrom(from, to, amount);
    }
```

# G-2 Optimizing Rate Update Functionality:

Original Implementation:
Calls _getPTandIBTRates() unconditionally, potentially leading to redundant function calls.
Refactored Improvement:
Modified to avoid unnecessary function calls when the expiry condition is met.
Enhances gas efficiency and streamlines the update process.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L879

```diff
function _updatePTandIBTRates() internal returns (uint256 _ptRate, uint256 _ibtRate) {
     if (block.timestamp >= expiry) {
         if (!ratesAtExpiryStored) {
             storeRatesAtExpiry();
         }
     }       
-    (_ptRate, _ibtRate) = _getPTandIBTRates(false);
     else if (block.timestamp < expiry) {
+        (_ptRate, _ibtRate) = _getPTandIBTRates(false);
         if (_ibtRate != ibtRate) {
             ibtRate = _ibtRate;
         }
         if (_ptRate != ptRate) {
             ptRate = _ptRate;
         }
     }
}
```
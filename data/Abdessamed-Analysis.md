# [N-01] Some `PrincipalToken` functions visibilities do not match with `IPrincipalToken`
The following functions visibilities have `public` visibility whereas they are not called from inside the contract and are not compatible with the `IPrincipalToken` interface: [deposit](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L171), [deposit](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L193), [depositIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L201), [depositIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L221), [redeem](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L245), [redeemForIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L270), [withdraw](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L295), [withdrawIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L321), [updateYield](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L340), [claimYield](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L369), [claimYieldInIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L377), [previewDeposit](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L430C14-L430C28), [maxWithdrawIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L466), [getTokenizationFee](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L602C14-L602C32)

Consider changing the visibilities of the mentioned functions to `external` to make them consistent with the interface.

# [N-02] Some functions on `IPrincipalToken` should have visibility `public` instead of `external`
The following functions are called inside the `PrincipalToken` contract, so they should have `public` visibility in their signature in the interface instead of `external`: [redeem](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L125), [withdraw](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L180), [deposit](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L64), [depositIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L100), [storeRatesAtExpiry](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L275), [previewWithdrawIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L319), [maxWithdraw](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L327), [redeemForIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L155), [withdrawIBT](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/interfaces/IPrincipalToken.sol#L212)


# [N-03] `IPrincipalToken` should inherit from `IERC5095`
The `PrincipalToken` contract implements the functions defined by the `IERC5095` interface, but it does not inherit from it

Consider inheriting from `IERC5095` in `IPrincipalToken` to make it consistent
```diff
interface IPrincipalToken is 
    IERC20, 
    IERC20Metadata, 
    IERC3156FlashLender 
+   IERC5095 
```

# [N-04] `_computeYield` and `_tryGetTokenDecimals` functions of `PrincipalTokenUtil` should have `internal` visibility
Libraries are meant to be used by contracts and can only be accessed by contracts using them. Consider updating the visibility of [_computeYield](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L63) and [_tryGetTokenDecimals](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L137C14-L137C34) to `internal`

### Time spent:
20 hours
## [L-01] The pause blocks withdrawals
The `PrincipalToken` is pausable however the pause stops the protocol completely. That means, even if users might want to redeem their funds, they will not be able to do so, which is a funds freeze for them, inflicting potential loss as they cannot withdraw / sell them. 

However some protocols choose such design consciously, accepting the risk. In an ideal world, pause stops entering the protocol but allows for exiting it. But on the other hand, the complete emergency pause allows for stopping the whole protocol and potentially stop an entire attack.

Recommendation: Consider, if allowing users to exit the protocol during pause is acceptable for you.


## [L-02] Lack of slippage control on `claimFees`
Function [claimFees](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L329) does not have option to specify or force a minimum amount accepted as fees. It just blindly redeems the `ibts` equal to amount of fees, sets remaining fees to 0 and the recovered amount of assets is sent to fee collector. 

However it should be kept in mind that redeeming from a vault might be subject to volatility, and ability to set a maximal slippage e.g. 5-10% would rule out a possibility where fee collector will redeem almost nothing due to slippage.

On the other hand, the amounts recovered from that source are probably not going to be significant, which might not incentive sandwichers, thus some losses may occur only due to organic volatility. 

Recommendation: Consider adding ability to specify % slippage, minimal amount, or set it to auto, if fee claiming is meant to be autonomous.


## [I-03] Inconsistent / redundant code in flashloan logic

In function [flashLoan](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L609), there is argument `_token` which can be set by user, and is user throughout the function code.

The `flashLoan` utility is limited by the [maxFlashLoan](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L583) function, which currently says that if the requested token is not `ibt`, then return 0 effectively disallowing flashloans in other tokens. 

Back to `flashLoan` function, at line 621, it says

```
        IERC20(ibt).safeTransfer(address(_receiver), _amount);
```
and in 628:
```
IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);
```

and all other functions still use the argument `_token`. As of now, this makes sense, as only `ibt` is allowed. However, the code looks like it's prepared to be ready to loan also other tokens. But 

Recommendation: Consider changing

```
        IERC20(_token).safeTransfer(address(_receiver), _amount);
...
IERC20(_token).safeTransferFrom(address(_receiver), address(this), _amount + fee);
```

To 

```
        IERC20(_token).safeTransfer(address(_receiver), _amount);
...
IERC20(_token).safeTransferFrom(address(_receiver), address(this), _amount + fee);
```
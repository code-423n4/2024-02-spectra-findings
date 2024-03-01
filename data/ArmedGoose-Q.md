## [L-01] The pause blocks withdrawals
The `PrincipalToken` is pausable however the pause stops the protocol completely. That means, even if users might want to redeem their funds, they will not be able to do so, which is a funds freeze for them, inflicting potential loss as they cannot withdraw / sell them. 

However some protocols choose such design consciously, accepting the risk. In an ideal world, pause stops entering the protocol but allows for exiting it. But on the other hand, the complete emergency pause allows for stopping the whole protocol and potentially stop an entire attack.

Recommendation: Consider, if allowing users to exit the protocol during pause is acceptable for you.


## [L-02] Lack of slippage control on `claimFees`
Function [claimFees](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L329) does not have option to specify or force a minimum amount accepted as fees. It just blindly redeems the `ibts` equal to amount of fees, sets remaining fees to 0 and the recovered amount of assets is sent to fee collector. 

However it should be kept in mind that redeeming from a vault might be subject to volatility, and ability to set a maximal slippage e.g. 5-10% would rule out a possibility where fee collector will redeem almost nothing due to slippage.

On the other hand, the amounts recovered from that source are probably not going to be significant, which might not incentive sandwichers, thus some losses may occur only due to organic volatility. 

Recommendation: Consider adding ability to specify % slippage, minimal amount, or set it to auto, if fee claiming is meant to be autonomous.


## [I-01] Consider adding a rescue/sweep function
The `PrincipalToken` contract is used to transfer tokens to it. The flashloan function has a placeholder to custom token address. However, there is no utility to rescue tokens which might be sent to the contract. A function allowing the DAO for example to claim dust balance, but not considering IBTs or fees, might be helpful to prevent dust or mistakenly transferred tokens to be stuck on the contract. 
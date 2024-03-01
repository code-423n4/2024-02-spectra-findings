## [L-01] Place an assertion to ensure the system is in paused/unpaused state
The functions `pause()/unpause()` are super important in case something suspicious starts to happen. The DAO address with the `PAUSER_ROLE` might accidentally invoke `unpause()` instead of `pause()` and be in the belief that he had stopped the suspicious activity. However, this might give an attacker the chance to continue his dirty activity.

#### Example of an occurrence
Could be added, for instance, [here](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L161-L168).
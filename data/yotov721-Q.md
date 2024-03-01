[NC-1] Do not pass address zero when accessing function selector 
In `PrincipalToken::_deployYT` a call to initialize YT token is made using `encodeWithSelector` . The initialize function if the YT token is called using `IYieldToken(address(0)).initialize.selector`. Use                     `IYieldToken.initialize.selector` instead as passing address 0 could lead to unexpected behavior. 

[NC-2] Wrong event parameter passed. 
In `PrincipalToken::_depositIBT()` the Mint event is emited. The 1st pparamter being `minter` is set to `msg.sender`. Should be `address(0)` as it is the default address set to `from` when tokens are minted

[NC-3] Weird pragma can confusing
In `RayMath.sol` the pragma is set to `pragma solidity =0.8.20;` which is the same as `0.8.20;`. 
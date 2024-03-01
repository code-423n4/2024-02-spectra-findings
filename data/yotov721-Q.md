[NC-1] Do not pass address zero when accessing function selector 
In `PrincipalToken::_deployYT` a call to initialize YT token is made using `encodeWithSelector` . The initialize function if the YT token is called using `IYieldToken(address(0)).initialize.selector`. Use                     `IYieldToken.initialize.selector` instead as passing address 0 could lead to unexpected behavior. 

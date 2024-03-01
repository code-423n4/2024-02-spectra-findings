## [G-01] Use assembly to write address storage values

When writing value for variables whose type is address, make use of assembly code instead of solidity code. 

There are 5 instances of this issue in 3 files:

```
File: src/proxy/AMBeacon.sol	

79: _implementation = newImplementation;
```
    diff --git a/src/proxy/AMBeacon.sol b/src/proxy/AMBeacon.sol
    index 5f820d6..0334866 100644
    --- a/src/proxy/AMBeacon.sol
    +++ b/src/proxy/AMBeacon.sol
    @@ -76,7 +76,9 @@ contract AMBeacon is IBeacon, AccessManaged {
             if (newImplementation.code.length == 0) {
                 revert BeaconInvalidImplementation(newImplementation);
             }
    -        _implementation = newImplementation;
    +       assembly{
    +               sstore(_implementation,newImplementation)
    +       }
             emit Upgraded(newImplementation);
         }
     }

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol

```
File: src/tokens/PrincipalToken.sol	

106: registry = _registry;

150: ibt = _ibt;

423: rewardsProxy = _rewardsProxy;
```
    diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
    index 9f4b912..799141b 100644
    --- a/src/tokens/PrincipalToken.sol
    +++ b/src/tokens/PrincipalToken.sol
    @@ -103,7 +103,9 @@ contract PrincipalToken is
             if (_registry == address(0)) {
                 revert AddressError();
             }
    -        registry = _registry;
    +       assembly{
    +               sstore(registry,_registry)
    +       }
             _disableInitializers(); // using this so that the deployed logic contract cannot later be initialized
         }

    @@ -147,7 +149,9 @@ contract PrincipalToken is
             ) {
                 revert InvalidDecimals();
             }
    -        ibt = _ibt;
    +       assembly{
    +               sstore(ibt,_ibt)
    +       }
             ibtUnit = 10 ** _ibtDecimals;
             ibtRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
             ptRate = RayMath.RAY_UNIT;
    @@ -420,7 +424,9 @@ contract PrincipalToken is
         function setRewardsProxy(address _rewardsProxy) external restricted {
             // Note: address zero is allowed in order to disable the claim proxy
             emit RewardsProxyChange(rewardsProxy, _rewardsProxy);
    -        rewardsProxy = _rewardsProxy;
    +       assembly{
    +               sstore(rewardsProxy, _rewardsProxy)
    +       }
         }

         /* GETTERS

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol

```
File: src/tokens/YieldToken.sol	
 
38: pt = _pt;
```
    diff --git a/src/tokens/YieldToken.sol b/src/tokens/YieldToken.sol
    index 5f1b11b..57f47e3 100644
    --- a/src/tokens/YieldToken.sol
    +++ b/src/tokens/YieldToken.sol
    @@ -35,7 +35,9 @@ contract YieldToken is IYieldToken, ERC20PermitUpgradeable {
         ) external initializer {
             __ERC20_init(_name, _symbol);
             __ERC20Permit_init(_name);
    -        pt = _pt;
    +       assembly{
    +               sstore(pt, _pt)
    +       }
         }

         /** @dev See {IYieldToken-burnWithoutUpdate} */

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol

#### Test Code

    contract GasTest is DSTest {
        Contract0 c0;
        Contract1 c1;
    
        function setUp() public {
            c0 = new Contract0();
            c1 = new Contract1();
        }
    
        function testGas() public {
            c0.setOwnerAssembly(0xFD2dabe9DFcc4d88a12A9D0D40D834E81217Cccf);
            c1.setOwner(0xFD2dabe9DFcc4d88a12A9D0D40D834E81217Cccf);
        }
    }
    
    contract Contract0 {
    
        address owner;
        function setOwnerAssembly(address _owner) public {
            assembly{
                sstore(owner.slot,_owner)
            }
        }
    
    }
    
    contract Contract1 {
        address owner;
        function setOwner(address _owner) public {
            owner = _owner;
        }
    
    }

#### Gas Report

|  Contract0 contract                       |                 |       |        |       |         |
|-------------------------------------------|-----------------|-------|--------|-------|---------|
| Deployment Cost                           | Deployment Size |       |        |       |         |
| 35287                                     | 207             |       |        |       |         |
| Function Name                             | min             | avg   | median | max   | # calls |
| setOwnerAssembly                          | 22324           | 22324 | 22324  | 22324 | 1       |

|  Contract1 contract                       |                 |       |        |       |         |
|-------------------------------------------|-----------------|-------|--------|-------|---------|
| Deployment Cost                           | Deployment Size |       |        |       |         |
| 48499                                     | 273             |       |        |       |         |
| Function Name                             | min             | avg   | median | max   | # calls |
| setOwner                                  | 22363           | 22363 | 22363  | 22363 | 1       |

## [G-02] Using XOR (^) and AND (&) bitwise equivalents

Given 4 variables a, b, c and d represented as such:

    0 0 0 0 0 1 1 0 <- a
    0 1 1 0 0 1 1 0 <- b
    0 0 0 0 0 0 0 0 <- c
    1 1 1 1 1 1 1 1 <- d

To have a == b means that every 0 and 1 match on both variables. Meaning that a XOR (operator ^) would evaluate to 0 ((a ^ b) == 0), as it excludes by definition any equalities.Now, if a != b, this means that there’s at least somewhere a 1 and a 0 not matching between a and b, making (a ^ b) != 0.Both formulas are logically equivalent and using the XOR bitwise operator costs actually the same amount of gas.However, it is much cheaper to use the bitwise OR operator (|) than comparing the truthy or falsy values.These are logically equivalent too, as the OR bitwise operator (|) would result in a 1 somewhere if any value is not 0 between the XOR (^) statements, meaning if any XOR (^) statement verifies that its arguments are different.

There is 1 instance of this issue in 1 file:

```
File: src/tokens/PrincipalToken.sol	

125: if (_ibt == address(0) || _initialAuthority == address(0)) {
```
    diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
    index 9f4b912..42580e7 100644
    --- a/src/tokens/PrincipalToken.sol
    +++ b/src/tokens/PrincipalToken.sol
    @@ -122,7 +122,7 @@ contract PrincipalToken is
             uint256 _duration,
             address _initialAuthority
         ) external initializer {
    -        if (_ibt == address(0) || _initialAuthority == address(0)) {
    +        if (((_ibt ^ address(0)) & (_initialAuthority ^ address(0))) == 0) {
                 revert AddressError();
             }
             if (IERC4626(_ibt).totalAssets() == 0) {

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol

#### Test Code

    contract GasTest is DSTest {
        Contract0 c0;
        Contract1 c1;
        function setUp() public {
            c0 = new Contract0();
            c1 = new Contract1();
        }
        function testGas() public {
            c0.not_optimized(1,2);
            c1.optimized(1,2);
        }
    }
    
    contract Contract0 {
        function not_optimized(uint8 a,uint8 b) public returns(bool){
            return ((a==1) || (b==1));
        }
    }
    
    contract Contract1 {
        function optimized(uint8 a,uint8 b) public returns(bool){
            return ((a ^ 1) & (b ^ 1)) == 0;
        }
    }

#### Gas Report

| Contract0 contract                        |                 |     |        |     |         |
|-------------------------------------------|-----------------|-----|--------|-----|---------|
| Deployment Cost                           | Deployment Size |     |        |     |         |
| 46099                                     | 261             |     |        |     |         |
| Function Name                             | min             | avg | median | max | # calls |
| not_optimized                             | 456             | 456 | 456    | 456 | 1       |

| Contract1 contract                        |                 |     |        |     |         |
|-------------------------------------------|-----------------|-----|--------|-----|---------|
| Deployment Cost                           | Deployment Size |     |        |     |         |
| 42493                                     | 243             |     |        |     |         |
| Function Name                             | min             | avg | median | max | # calls |
| optimized                                 | 430             | 430 | 430    | 430 | 1       |
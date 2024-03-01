# [1] Setters should prevent re-setting of the same value

**File:** `AMBeacon.sol`

[File: src/proxy/AMBeacon.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L63)
```solidity
63:     function upgradeTo(address newImplementation) public virtual restricted {
64:         _setImplementation(newImplementation);
65:     }
```

[File: src/proxy/AMBeacon.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L75)
```solidity
75:     function _setImplementation(address newImplementation) private {
76:         if (newImplementation.code.length == 0) {
77:             revert BeaconInvalidImplementation(newImplementation);
78:         }
79:         _implementation = newImplementation;
80:         emit Upgraded(newImplementation);
81:     }
```

It's possible to call function which updates the state variable with the same value.
It's a good practice to make sure, that whenever we update some value, it's not being set to the same value which it was before the update.
E.g., let's consider a scenario where `_implementation = 0xAAA;`. Calling `upgradeTo(0xAAA)` is possible, even though it won't change the value of the variable: `_implementation` will remain `0xAAA`.
Moreover, after calling `upgradeTo(0xAAA)`, function will still emit an `Upgraded(0xAAA)` event, even though nothing had been upgraded (the `_implementation` would remain the same). Seeing that event might be confusing for the end-user.

Our recommendation is to implement additional check which verifies if value is really changed. This can be done in `upgradeTo()` function, by adding additional `require` check:

```
require(newImplementation != _implementation, "Nothing to update!");
```

# [2] Implement modifier instead of repeating the same check

**File:** `YieldToken.sol`

[File: src/tokens/YieldToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L42)
```solidity
42:     function burnWithoutUpdate(address from, uint256 amount) external override {
43:         if (msg.sender != pt) {
44:             revert CallerIsNotPtContract();
45:         }
```

[File: src/tokens/YieldToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L50)
```solidity
50:     function mint(address to, uint256 amount) external override {
51:         if (msg.sender != pt) {
52:             revert CallerIsNotPtContract();
53:         }
```

Both functions (`mint()` and `burnWithoutUpdate()`) share the same conditional check - which can be easily refactored into a modifier. Using a modifier will increase the code readability -  thus it's highly recommended.


# [3] Comment in `_dispatchUpgradeToAndCall()` should be more detailed

**File:** `AMTransparentUpgradeableProxy.sol`

[File: src/proxy/AMTransparentUpgradeableProxy.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L118)
```solidity
118:      * - If `data` is empty, `msg.value` must be zero.
```

Above comment is not very descriptive - since it does not explain what would happen when `data` is empty and `msg.value` would not be zero. Since `_dispatchUpgradeToAndCall()` calls `upgradeToAndCall()` from Open Zeppelin's `ERC1967Utils` - we can spot in the Open Zeppelin's sources that in the above scenario - function will revert with `ERC1967NonPayable()`. The Open Zeppelin's implementation explains this condition as below: 

```
     * This function is payable only if the setup call is performed, otherwise `msg.value` is rejected
     * to avoid stuck value in the contract.
```

Our recommendation is to extend the current comment, to better explain this scenario.

```
If `data` is empty, `msg.value` must be zero.
```

should be changed to:

```
If `data` is empty, `msg.value` must be zero. If `data` is empty and `msg.value` is zero - function will revert with `ERC1967NonPayable()`. This behavior helps to avoid stuck the `msg.value` in the contract. 
```

# [4] Change order of parameters in `if` condition

**File:** `PrincipalTokenUtil.sol`

[File: src/libraries/PrincipalTokenUtil.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L64)
```solidity
64:         if (_oldPTRate == _ptRate && _ibtRate == _oldIBTRate) {
65:             return _userYieldIBT;
66:         }
```

Above `if` condition compares two old values to the new ones. `_oldPTRate` is compared against the current `_ptRate` and `_oldIBTRate` is compared against `_ibtRate`. Our recommendation is to always use the new values on the same side and the old ones, on the another side. In the above example: `_oldPTRate == _ptRate` - old is on the left, while new one is on the right. While in `_ibtRate == _oldIBTRate` - the new one is on the left and the old one on the right. 
It's recommended to put the old values on the right side and the new ones to the left. This will increase the code readability.

Above line: `if (_oldPTRate == _ptRate && _ibtRate == _oldIBTRate)` should be changed to: `if (_ptRate == _oldPTRate && _ibtRate == _oldIBTRate)`. After this change, the new values (`_ptRate`, `_ibtRate`) are on the left side, while the old ones (`_oldPTRate`, `_oldIBTRate`) - on the right side.

The same issue occurs in below instance:

[File: src/libraries/PrincipalTokenUtil.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L73)
```solidity
73:         if (_oldPTRate == _ptRate && _ibtRate > _oldIBTRate) {
```

Our recommendation is to change `if (_oldPTRate == _ptRate && _ibtRate > _oldIBTRate)` to `if (_ptRate == _oldPTRate && _ibtRate > _oldIBTRate)`.


# [5] Incorrect punctuation

**File:** `AMTransparentUpgradeableProxy.sol`

[File: src/proxy/AMTransparentUpgradeableProxy.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L99)
```solidity
99:      * @dev If caller is the admin process the call internally, otherwise transparently fallback to the proxy behavior.
```
Use a comma if the if clause is at the beginning of the sentence.
The sentence `If caller is the admin process the call internally`  should be changed to: `If caller is the admin, process the call internally`.


# [6] Typos

**Files:** `YieldToken.sol`, `PrincipalToken.sol`

[File: src/tokens/PrincipalToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L115)
```solidity
115:      * it deploys yt and intializes values of required variables
```

`intializes` should be changed to `initializes`.

[File: src/tokens/PrincipalToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L774)
```solidity
774:      * @param _receiver The addresss of the receiver of the assets
```

`addresss` should be changed to `address`.

[File: src/tokens/YieldToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L27)
```solidity
27:      * @param _name The name of the yt token.
28:      * @param _symbol The symbol of the yt token.
29:      * @param _pt The address of the pt associated with this yt token.
```

`yt` should be uppercased - `YT`.
`YT` stands for `YieldToken`, thus `yt token` basically means `YieldToken token`. It's better to change `yt token` to `YieldToken`:

```
     * @param _name The name of the YieldToken.
     * @param _symbol The symbol of the YieldToken.
     * @param _pt The address of the pt associated with this YieldToken.
```

# [7] Stick to one way of commenting style

**Files:** `AMBeacon.sol`, `AMProxyAdmin.sol`

Across the whole code-base, variables inside comments are inside grave accents, e.g.:

[File: src/proxy/AMProxyAdmin.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L37)
```solidity
37:      * - If `data` is empty, `msg.value` must be zero.
```
As demonstrated above, `data` and `msg.value` are between grave accents symbols.

During the code-review process, the deviation from this rule was detected in two instances. As demonstrated below, `msg.sender` is not surrounded by grave accents, while other variables are. Our recommendation is to alter below comments by use grave accent before and after `msg.sender`. Sticking to one commenting-style increases the code readability, thus it's very recommended.

[File: src/proxy/AMProxyAdmin.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L38)
```solidity
38:      * - msg.sender must have a role that allows them to upgrade the proxy.
```

[File: src/proxy/AMBeacon.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L57)
```solidity
57:      * - msg.sender must have the appropriate role in the authority.
```

# [8] Typo in function name

**File:** `PrincipalToken.sol`

[File: src/tokens/PrincipalToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L166)
```solidity
166:     function unPause() external override restricted {
167:         _unpause();
168:     }
```

When contract is PausableUpgradeable, it should implement external `pause()` and `unpause()` methods, since by default it inherits only internal `_pause()` and `_unpause()`. The current implementation of `PrincipalToken.sol` follows this rule by implementing external functions: `pause()` and `unPause()`, which call internal `_pause()` and `_unpause()`. 

Since function `unPause()` calls `_unpause()` - it's a good practice to rename `unPause()` to `unpause()` (note the lower-cased letter `p`). Another reason for this recommendation, is that when function `unPause()` will be called - the Open Zeppelin's `_unpause()` will emit an `Unpause()` event. Since this is `Unpause()` (and not `UnPause()`) event - similarly changing the name of the function will increase the code readability.

Our recommendation is to change the name of the function from `unPause()` to `unpause()`.

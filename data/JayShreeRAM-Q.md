### 1. PrincipalToken: Potential integer overflow in _calculateTotalDebt()

**Title:** Potential integer overflow in _calculateTotalDebt()

**Description:** In PrincipalToken._calculateTotalDebt(), there is a potential integer overflow due to the multiplication of _totalSupply and _currentRate.

**Contract:** PrincipalToken.sol#L44

**Proof of Concept:** Suppose _totalSupply is equal to the maximum value of uint256 (type(uint256).max), and _currentRate is greater than 1. In this case, multiplying these values can result in an integer overflow, causing the function to revert.

**Impact:** This issue can lead to the function reverting, potentially causing disruptions in the system.

**Recommendation:** Consider adding checks to prevent integer overflows:

if (_totalSupply > 0 && _currentRate > 0 && _currentRate > type(uint256).max / _totalSupply) {
    // Overflow handling
}
### 2. PrincipalToken: Missing event logs for important state changes

**Title:** Missing event logs for important state changes

**Description:** The PrincipalToken contract lacks event logs for several important state changes, such as changing the name, symbol, and decimals.

**Contract:** PrincipalToken.sol

**Proof of Concept:** Consider the following line in PrincipalToken.sol#L29:

name = name_;
No event log is emitted to notify external parties about this change.

**Impact:** Missing event logs can make it difficult for external parties to track important state changes.

**Recommendation:** Add event logs for state changes, as demonstrated in this example:

event NameChanged(string oldName, string newName);
...
name = name_;
emit NameChanged(name, name_);
### 3. PrincipalToken: Incorrect handling of zero-valued decimals

**Title:** Incorrect handling of zero-valued decimals

**Description:** The PrincipalToken contract does not handle the case where the decimals value is set to 0.

**Contract:** PrincipalToken.sol#L77

**Proof of Concept:** The calculation for _currentDebt includes a division operation, which would fail if decimals is equal to 0.

**Impact:** Incorrect handling of zero-valued decimals can lead to errors in calculations involving the _currentDebt.

**Recommendation:** Limit the minimum value of decimals to a positive integer, or handle the case of zero-valued decimals appropriately.

### 4. PrincipalToken: Incorrect handling of negative and zero interest rates

**Title:** Incorrect handling of negative and zero interest rates

**Description:** The PrincipalToken contract does not handle the case where the _currentRate value is less than or equal to 0.

**Contract:** PrincipalToken.sol#L64

**Proof of Concept:** The _calculateTotalDebt() function uses the _currentRate variable in calculations without checking its value. A negative or zero value would result in incorrect or undefined behavior in the calculations.

**Impact:** Incorrect handling of negative and zero interest rates can lead to errors in calculations involving debt.

**Recommendation:** Limit the minimum value of _currentRate to a positive value, or handle the case of non-positive interest rates appropriately.

### 5. PrincipalTokenUtil: Infinite loop due to recursive call

**Title:** Infinite loop due to recursive call

**Description:** The PrincipalTokenUtil._transferWithCheck() function can result in an infinite loop when _to is the zero address.

**Contract:** PrincipalTokenUtil.sol#L26

**Proof of Concept:** If _to is the zero address, the _checkTransfer() function will transfer the tokens back to the original _from address, causing the _transferWithCheck() function to be called recursively and resulting in an infinite loop.

**Impact:** An infinite loop can lead to a denial-of-service attack.

**Recommendation:** Prevent the recursive call by checking whether the _to address is zero before calling _transferWithCheck().

if (_to == address(0)) {
    revert InvalidToAddress();
}
### 6. AMProxyAdmin: Missing constructors for admin-related functions

**Title:** Missing constructors for admin-related functions

**Description:** The AMProxyAdmin contract's initializeAdmin() function is not marked as internal, making it callable by any external address.

**Contract:** AMProxyAdmin.sol#L20

**Proof of Concept:** Consider the following example:

AMProxyAdmin proxyAdmin = new AMProxyAdmin();
proxyAdmin.initializeAdmin(owner, admin);
In this example, the initializeAdmin() function can be called by any address, including non-owners, which might not be intended behavior.

**Impact:** Missing constructor protection allows admin-related functions to be called by any external address.

**Recommendation:** Make initializeAdmin() an internal function or implement a constructor to initialize the admin.

### 7. PrincipalTokenUtil: Incorrect handling of zero-valued token transfers

**Title:** Incorrect handling of zero-valued token transfers

**Description:** The _transferWithCheck() function in PrincipalTokenUtil does not handle the case where _value is zero.

**Contract:** PrincipalTokenUtil.sol#L25

**Proof of Concept:** If _value is zero, the function will still call the _transfer() function and _checkTransfer(), incurring unnecessary gas costs.

**Impact:** Incorrect handling of zero-valued token transfers leads to unnecessary gas costs.

**Recommendation:** Check whether the _value is zero before continuing with token transfers and _checkTransfer().

### 8. PrincipalTokenUtil: Missing event logs for important state changes

**Title:** Missing event logs for important state changes

**Description:** The PrincipalTokenUtil library lacks event logs for several important state changes, such as setting the minTransferAmount.

**Contract:** PrincipalTokenUtil.sol#L52

**Proof of Concept:** Consider the following line in PrincipalTokenUtil.sol#L52:

minTransferAmount = _minTransferAmount_;
No event log is emitted to notify external parties about this change.

**Impact:** Missing event logs can make it difficult for external parties to track important state changes.

**Recommendation:** Add event logs for state changes, as demonstrated in this example:

event MinTransferAmountChanged(uint256 oldValue, uint256 newValue);
...
minTransferAmount = _minTransferAmount_;
emit MinTransferAmountChanged(minTransferAmount, _minTransferAmount_);
### 9. PrincipalTokenUtil: Sending tokens to zero address

**Title:** Sending tokens to zero address

**Description:** The _transferWithCheck() function in PrincipalTokenUtil sends tokens to the zero address under certain conditions.

**Contract:** PrincipalTokenUtil.sol#L29

**Proof of Concept:** If the _transfer() function reverts, the function calls _checkTransfer(), which transfers the tokens back to the original _from address. If _from is the zero address, the tokens will be lost.

**Impact:** Sending tokens to the zero address leads to a loss of tokens and potential denial-of-service attacks.

**Recommendation:** Check whether the _from address is the zero address before performing the token transfer.

### 10. RayMath: Incorrect handling of zero-valued multiplication

**Title:** Incorrect handling of zero-valued multiplication

**Description:** The _mul() function in RayMath does not have the appropriate handling for zero-valued multiplication.

**Contract:** RayMath.sol#L48

**Proof of Concept:** Consider the following example:

uint256 a = 0;
uint256 b = 10;
When multiplying these values, the result should be 0, but the _mul() function will return 0.000000000000000001 instead.

**Impact:** Incorrect handling of zero-valued multiplication can lead to errors in calculations involving multiplication.

**Recommendation:** Check for zero-valued inputs before performing the multiplication and return 0 in such cases.

### 11. RayMath: Incorrect handling of zero-valued subtraction

**Title:** Incorrect handling of zero-valued subtraction

**Description:** The _sub() function in RayMath does not have the appropriate handling for zero-valued subtraction.

**Contract:** RayMath.sol#L63

**Proof of Concept:** Consider the following example:

uint256 a = 0;
uint256 b = 10;
When subtracting these values, the result should be -10, but the _sub() function will return 0 instead.

**Impact:** Incorrect handling of zero-valued subtraction can lead to errors in calculations involving subtraction.

**Recommendation:** Check for zero-valued inputs before performing the subtraction and return the correct value in such cases.

### 12. RayMath: Incorrect handling of division by zero

**Title:** Incorrect handling of division by zero

**Description:** The _div() function in RayMath does not have the appropriate handling for division by zero.

**Contract:** RayMath.sol#L77

**Proof of Concept**: Consider the following example:

uint256 a = 10;
uint256 b = 0;
When dividing these values, the function should revert due to a division-by-zero error, but it does not.

**Impact:** Incorrect handling of division by zero can lead to errors in calculations involving division.

**Recommendation:** Check whether the denominator is zero before performing the division and revert the transaction if it is.

### 13. AMTransparentUpgradeableProxy: No validation of admin address

**Title:** No validation of admin address

**Description:** The AMTransparentUpgradeableProxy contract does not validate the admin address in the initialize() function.

**Contract:** AMTransparentUpgradeableProxy.sol#L23

**Proof of Concept:** Consider the following example:

AMTransparentUpgradeableProxy proxy = new AMTransparentUpgradeableProxy();
proxy.initialize(implementation_, address(0));
In this example, the admin address can be set to the zero address.

**Impact**: Assigning the admin address to the zero address can lead to a denial-of-service attack.

**Recommendation:** Validate the admin address in the initialize() function and revert the transaction if the admin address is the zero address.



### 14. YieldToken:_stake() and _unstake() functions do not always emit Staked() and Unstaked() events

**Title:** Event logs not always emitted in _stake() and _unstake() functions

**Description:** The _stake() and _unstake() functions in YieldToken.sol do not always emit Staked() and Unstaked() events.

**Contract:** YieldToken.sol #L75, L100

**Proof of Concept:** Consider the following example:

A user calls the _stake() function with a uint256 value less than the minStakeAmount.
The transaction succeeds, but the Staked() event is not emitted.
**Impact:** Missing event logs can make it difficult for external parties to track important state changes.

**Recommendation:** Ensure that all important state changes are accompanied by the required event logs.

### 15. YieldToken: Incorrect calculation of individual user debt in _calculateDebt()

**Title**: Incorrect calculation of individual user debt in _calculateDebt()

**Description:** The _calculateDebt() function in YieldToken.sol calculates the individual user debt incorrectly, using _currentTotalSupply instead of _currentPrincipalAmount.

**Contract:** YieldToken.sol #L93

**Proof of Concept**: Consider the following case:

User A stakes 100 tokens in the _stake() function.
User B stakes 100 tokens in the _stake() function.
User A calls the _unstake() function to unstake 90 tokens.
The _calculateDebt() function calculates individual user debt:
r = _currentTotalSupply.mul(_stake / _totalStaked);
In this case, the function calculates the user debt incorrectly, using the total supply instead of the total principal amount, which is 100 tokens.

**Impact**: Incorrect calculation of individual user debt can lead to the mismanagement of user funds or invalid debt calculations.

**Recommendation**: Change the calculation to use _currentPrincipalAmount instead of _currentTotalSupply.




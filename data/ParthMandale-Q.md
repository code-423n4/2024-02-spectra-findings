### [01] PrincipalToken::_depositIBT Violation of Checks-Effects-Interactions pattern

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L767

**Description:** Solidity recommends the usage of the Checks-Effects-Interactions pattern to avoid potential security vulnerabilities, [CEI/FREI-PI](https://www.nascent.xyz/idea/youre-writing-require-statements-wrong).

The _depositIBT function violates the Checks-Effects-Interactions (CEI) pattern by emitting the Mint event after the _mint function call, which is considered an interaction. According to CEI, the emission of events should occur before any state changes or external interactions to ensure that the events accurately reflect the state changes made by the function.


```javascript
     function _depositIBT(
        uint256 _ibts,
        address _ptReceiver,
        address _ytReceiver
    ) internal notExpired nonReentrant whenNotPaused returns (uint256 shares) {
        updateYield(_ytReceiver);
        uint256 tokenizationFee = PrincipalTokenUtil._computeTokenizationFee(
            _ibts,
            address(this),
            registry
        );
        _updateFees(tokenizationFee);
        shares = _convertIBTsToShares(_ibts - tokenizationFee, false);
        if (shares == 0) {
            revert RateError();
        }
        _mint(_ptReceiver, shares);
@>      emit Mint(msg.sender, _ptReceiver, shares);
        IYieldToken(yt).mint(_ytReceiver, shares);
    }
```

**Recommended Mitigation:** 
The recommended mitigation is to emit the event before the _mint function call.
```
        emit Mint(msg.sender, _ptReceiver, shares);
        _mint(_ptReceiver, shares);
```

### [02] In PrincipalToken.sol and YieldToken.sol contract - Missing Best Practices: Mark Functions as External When No Internal Calls Are Made

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L430

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L471

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L483

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L503

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L58

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L71

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L95

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L105

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L116

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L121

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L127

**Description:** It is a best practice to mark functions as external instead of public when no internal calls are made within the function. This ensures clarity and adherence to the principle of least privilege, enhancing code readability and reducing potential attack surfaces by explicitly indicating that the function is intended to be called from outside the contract.

**Recommended Mitigation:** 
Change the visibility specifier from `public` to `external` for functions where no internal calls are made, thereby explicitly indicating that the function is intended to be called from outside the contract and reducing potential attack surfaces.

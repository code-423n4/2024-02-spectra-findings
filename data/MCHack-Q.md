### [N-1] The CEI development model is not being followed

**Description:**

In  `PrincipalToken::_claimYield` function does not follow CEI/FREI-PI.

**Impact:**

In this particular case, there is no impact, but the culture of writing code must be present throughout the code base, otherwise it will eventually lead to more serious errors.

**Proof of Concept:**

**Recommended Mitigation:**

In the `PrincipalToken::_claimYield`  function, you need to change the order of the external call.

```diff

    function _claimYield() internal returns (uint256 yieldInIBT) {
        yieldInIBT = updateYield(msg.sender);
        if (yieldInIBT == 0) {
            return 0;
        } else {
            yieldOfUserInIBT[msg.sender] = 0;
            uint256 yieldFeeInIBT = PrincipalTokenUtil._computeYieldFee(yieldInIBT, registry);
-            _updateFees(yieldFeeInIBT);
-            yieldInIBT -= yieldFeeInIBT;
+            yieldInIBT -= yieldFeeInIBT;
+            _updateFees(yieldFeeInIBT);
            emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
        }
    }

```
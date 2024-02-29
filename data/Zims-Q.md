`PrincipalToken.sol` should add a gap for future variables to avoid storage clashes.
The contract uses the upgradeable pattern. If a need for new variables arrises it is good practice to introduce a gap variable to avoid future storage clashes.
Just add this as another variable definition to reserve 48 more slots for future use.
`uint256[48] private __gap;`
After these lines:
```
    uint256 private ptRate; // or PT price in asset (in Ray)
    uint256 private ibtRate; // or IBT price in asset (in Ray)
    uint256 private unclaimedFeesInIBT; // unclaimed fees
    uint256 private totalFeesInIBT; // total fees
    uint256 private expiry; // date of maturity (set at initialization)
    uint256 private duration; // duration to maturity
```

The gas costs are negligible but can avoid future mistakes.

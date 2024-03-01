## [N-01] Event is missing *indexed* fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it’s not necessarily best to index the maximum allowed per event (threefields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

There are 2 instances of this issue in 1 file:

    File: src/tokens/PrincipalToken.sol	

    68: event Redeem(address indexed from, address indexed to, uint256 amount);

    69: event Mint(address indexed from, address indexed to, uint256 amount);
    
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol
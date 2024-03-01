[L-01] The pause blocks withdrawals
The `PrincipalToken` is pausable however the pause stops the protocol completely. That means, even if users might want to redeem their funds, they will not be able to do so, which is a funds freeze for them, inflicting potential loss as they cannot withdraw / sell them. 

However some protocols choose such design consciously, accepting the risk. In an ideal world, pause stops entering the protocol but allows for exiting it. But on the other hand, the complete emergency pause allows for stopping the whole protocol and potentially stop an entire attack.


[L-02] Lack of slippage control on `claimFees`



[I-01] Inconsistent code in flashloan logic

todo3 (token / ibt)
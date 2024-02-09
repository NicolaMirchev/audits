# MorpheusAI - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Multisig participants will lose their rewards, because target address on destination chain is not owner by them](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: MorpheusAI

### Dates: Jan 30th, 2024 - Feb 3rd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrzgrole0007xtsq0gfdw8if)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Multisig participants will lose their rewards, because target address on destination chain is not owner by them            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/L1Sender.sol#L128

https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/Distribution.sol#L174

## Summary
If a participant in staking is a multisig wallet and tries to claim rewards, a mint message would be sent to Arbutrim chain with the address of the multisig wallet. However multisigs are contracts deployed on chains and that's why destination chain address, which corresponds to source chain address won't be owned by the same person/s
## Vulnerability Details
In common case rewards would be sent to some random user who may don't want to refund them.
In the worst case expoiter can deploy contract on the following address and claim MOR tokens.
See [Wintermute](https://www.investopedia.com/wintermute-got-hacked-6741864) 
## Impact
Lost rewards for participants, who use multisig wallets. This is more ofter practice. Also the problem could occur in other account abstraction scenarios.
## Tools Used
Manual Review 
## Recommendations
Make `claim` function callable only by the owner and let him pass `receiver` address for the destination chain:
```diff
-    function claim(uint256 poolId_, address user_) external payable poolExists(poolId_) {
+   function claim(uint256 poolId_, address receiverAddress ) external payable poolExists(poolId_) {
        Pool storage pool = pools[poolId_];
        PoolData storage poolData = poolsData[poolId_];
-       UserData storage userData = usersData[user][poolId_];
+       UserData storage userData = usersData[msg.sender][poolId_];

        require(block.timestamp > pool.payoutStart + pool.claimLockPeriod, "DS: pool claim is locked");

        uint256 currentPoolRate_ = _getCurrentPoolRate(poolId_);
        uint256 pendingRewards_ = _getCurrentUserReward(currentPoolRate_, userData);
        require(pendingRewards_ > 0, "DS: nothing to claim");

        // Update pool data
        poolData.lastUpdate = uint128(block.timestamp);
        poolData.rate = currentPoolRate_;

        // Update user data
        userData.rate = currentPoolRate_;
        userData.pendingRewards = 0;

        // @audit if user is multisig wallet, rewards would be lost
        // @audit also if the caller of the message is multisig, surplus gass would be lost
        // Transfer rewards
+        L1Sender(l1Sender).sendMintMessage{value: msg.value}(receiverAddress, pendingRewards_, _msgSender());

        emit UserClaimed(poolId_, user_, pendingRewards_);
    }
```
		






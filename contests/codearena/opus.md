
## H1 - Flashloan manipulation can force shrine to enter `recovery_mode` and liquidate participants
### Impact:
Shrine has a `recovery` mode feature, which is activated when theaggregate loan-to-value ratio of all troves in the Shrine is equal to or greater than 70% of the Shrine's threshold
[Recovery mode](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L1040) adjusted downwards depending on the the extent to which the Shrine's aggregate loan-to-value ratio is greater than the Shrine's recovery mode threshold, down to a floor of 50% of the yang's original threshold. We can see that `recoveryMode` is considered active [dynamically](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L1046C30-L1046C63), when a health for a trove is checked, which means that there is a function, which is calculation the weighted average of the thresholds of all yangs, which are supported by the system, amount of all depts and value of all deposited yangs (LTV). And it check whether `(value/weighted average of the thresholds of all yangs) > 70%`, and if it is true, threshold for give trove is being scaled by the corresponding value.
```
if self.is_recovery_mode_helper(shrine_health) {
                let recovery_mode_threshold: Ray = shrine_health.threshold * RECOVERY_MODE_THRESHOLD_MULTIPLIER.into();
                return max(
                    threshold * THRESHOLD_DECREASE_FACTOR.into() * (recovery_mode_threshold / shrine_health.ltv),
                    (threshold.val / 2_u128).into()
                );
            }
``` 

As the system is cross-margin and we expect different yangs with different thresholds, this means that average threshold depends on deposited amount of each yang and the corresponding dept minted agains it (it LTV value).
This means that if we have yangs for stablecoins, which threshold would be larger (~90%), this could significantly change the average threshold) and so -> LTV. 
- If a user manages to deposit large amount of the yang with largest threshold, mint all dept possible, this will influence the recovery mode calculations and can benefit from liquidating users, when the threshold for their troves is scaled
- Attacker can use multiple flashloans to influence the overall threshold, liquidate the users, withdraw and depay his flashloans
- The recovery is "activated", but due to manupulation bounded in the current transaction, which result in:
- - attacker always being first to liquidate a user
- - falsy user liquidations with larger penalty and collateral 

Here is the formula, which is calculating weighted average of the thresholds: 	
`(yangADeposited / totalDeposited) * yangA% + (yangBDeposited / totalDeposited) * yangBDeposited% = weighted average of the thresholds %` (Note that this is for only two yangs and it can go on for as many collateral assets there are)
and the total LTV is calculated as follows:
`allTrovesDept/allDepositedYangsValue`

### PoC
Lets take a look at an example values, which are real for production enviroment:
- Yang with threshold of 70% and current LTV 65% (minted yin = 3.25M ;deposited collateral = 5M)
- Yang with threshold of 60% and current LTV 55% (minted yin = 2.75M; deposited collateral =  5M)

- Yang with threshold of 90% (suppose a stablecoin) (18M minted)
1. Malicious actor sees that he can benefit from this exploit by liquidating all users, which LTV is x% below the threshold
2. He uses a flashloan of 20M from the stablecoin -> deposit all and mint max possible amount of yin (90% of 20M)
3. Lets now calculate weighted average of the thresholds and LTV:
- `(5M / 30M) * 70% + (5M / 30M) * 60% + (20M / 30M) * 90% = 81%`
, so the manipulation has raised the threshold from avg of 65% -> 81% (16%)
- And average LTV = `(2.75M + 3,25M + 18M) / 30M =  80%` (10% > recovery mode threshold) 
4. Exploiter is able to liquidate multiple users, which has been ~ 5% below the liquidation threshold, because their threshold for the operation is scaled by the recovery factor
5. Exploiter uses his minted yin to liquidate part of their positions. With the yild he made, he can repay his yin dept (18M), withdraw flashloaned assets and repay them.
6. He has successfully manipulated the system to think that those trove were liquidatable, when they were not
<a href="https://imgur.com/9ETzX5i"><img src="https://i.imgur.com/9ETzX5i.png" title="source: imgur.com" /></a>
### Mitigation
- Think of a way to obtain the recovery mode from the last block, which will remove the possibility of flashloan manipulations


-  My concern is that flash loan (from external souce) + deposit and mint all available values, can significantly influence shrine's TVL, which therefore instantly activate recovery mode, That can be used by an attacker to fake recovery mode, if he would benefit from troves, which are close to liquidations, liquidate them, repay his trove's loan to withdraw the flashloaned amount, which is than returned
Example calculations about:
Yang with threshold of 70% and current LTV 65% (deposited collateral 1M)
Yang with threshold of 60% and current LTV 55% (deposited collateral 1M)

Yang with threshold of 90% (suppose a stablecoin) -> flashmint of 5M -> LTV of 90%

Result: Avg LTV of 81%, which is above 70% threshold for recovery mode

# H1 -  Users can steal other users rewards by allocating weight to gauges after `notifyPnL` with positive impact
## Impact 
[ProfitManager](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L30) is responsible for allocating and distributing rewards to all parties in the system, including GUILD holders, which participate in gauge support and earn rewards on payed interest. The contract has a function to [claim rewards pending for given gauge](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L409), but it never check when the participant has allocated his funds to the given gauge. This can result in allocating weight after a profit for the term and getting the reward, which another staker deserves. Worse case is when GUILD transferability is enabled and a user, which owns 10% of the allocated tokens for the term can withdraw rewards for the other 90% of the profit by transferring his tokens to another address owned by him, allocating weight to this gauge and claiming reward. This would result stealing reward funds, which are meant for early gauge supporters

# H2 - `getRewards` would mark user as slashed even if the loss has been applied
## Imapct
[SurplusGuildMinter](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L114) provide functionality to users to stake their CREDIT tokens and in return mint GUILD tokens and allocate corresponding weight to a given gauge. Before any operation [getRewards()](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L216) function is called to claim all standing rewards for the given user, before his stake/unstake. We can note something very suspicious inside first lines of getRewards function regarding checking if the user should be marked as slashed. We compare last loss for the given gauge with the default value of unit256, because the variable userStake is still empty:
```
function getRewards(
        address user,
        address term
    )
        public
        returns (
            uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
            UserStake memory userStake, // stake state after execution of getRewards()
            bool slashed // true if the user has been slashed
        )
    {
        bool updateState;
        lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
        if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
            slashed = true;
        }

        // if the user is not staking, do nothing
        userStake = _stakes[user][term]; 
        ...
```
## PoC
Run the following test inside `test/unit/loan/SurplusGuildMinter.t.sol` with command `forge test --match-test testUnstakeWithLoss -vv`

 https://gist.github.com/cholakovvv/e8514283ad7bd8bbbd2da87bd7bb0b1e

**Recommended Mitigation Steps**:

First assign `UserStake memory userStake` to the storage variable for the given user are then compare it with `lastGaugeLoss`

- **NOTE** that you should also implement a logic to assign value to `userStake.lastGaugeLoss` somehow. Currently this field is never assigned to a value different from 0 in the whole contract, which leads to mistakes.
- Only the following change won’t solve the problem, because you never set the field. But you should think of a good way how to interpret this field, when stakers enter the system after a loss and also update it when staker were in the system when you slash him.


# M1 - If a term is onboarded again before cleanup after offboard, functionality to `redeem` would be DoS-ed and funds would be locked
## Imapct
To offboard a term GUILD holders should [agree](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L116) on that. After a offboard is accepted, the gauge is removed from active gauges list and [redemption](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SimplePSM.sol#L134) inside `SimplePSM` is paused until all loans are paid. To unpause the redemption, [LendingTermOffboarding::cleanup](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L175)  should be called. But we can [note](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L181-L184) that it is a valid scenario to re-onboard a term, before all conditions for `cleanup` are met. But lets examine what would be the consequences from such an action.

1. When `offboard` is called `nOffboardingsInProgress` is incremented by 1 and `SimplePSM` redemption will be paused as long as  `nOffboardingsInProgress > 0`
2. But if a term is re-onboarded, before all his loans has been repaid, or nobody has called `cleanup` function we can notice another concern:
    
    2.1 To offboard a term we need [canOffboard[term]](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L154) to be true, which is set back to false inside `cleanup` function, which has never been called
    
    2.2 This means that after re-onboarding a single person can offboard it again by simply calling `offboard`
    
    2.3 Which would lead to the worst impact, which is incrementing `nOffboardingsInProgress` again for the same lending term. This means that now it is impossible to set `nOffboardingsInProgress` back to 0, because `cleanup` can be called only once for this term, which will decrement progress variable by only 1. The result is constantly paused `SimplePSM` and blocked funds for stakers. 
    
    **NOTE** there is a way for community to vote on unpausing the PSM, but this would take a lot of time, during which all PSM functionalities (mint/redeem) would be blocked and even after it’s unblocking, when another term is off-boarded, we again enter in long pause, which is only changeable after long GOV vote and Timelock waiting period. 
    
- Impact is inconsistencies between important state variables inside `LendingTermOffboarding` and blocked functionality and funds of `SimplePSM`

- This is an issue, even if transferability is not allowed, because user can reallocate his funds after noticing “PnL notification" with positive income.
- Coded PoC, which should be placed inside `test/unit/governance/ProfitManager.t.sol` and executed with `forge test --match-test testProfitDistributionRewardsStealing -vv`
    
    https://gist.github.com/NicolaMirchev/66f6fa840485e899164401b9c9386e73
    
- This could also block staking in `SurplusGuildMinter` for the victim, because of:
```
(uint256 lastGaugeLoss, UserStake memory userStake, ) = getRewards(
            msg.sender,
            term
        );
```
# M2 -  Malicious actor can intentionally slash any successful term GUILD holders
## Impact
In combination with USDT/USDC blacklist and block stuffing a malicious user can intentionally generate bad debt for successful term and slash all GUILD holders as a result.

Tokens like **USDT** and **USDC** have functions that allow them to *blacklist* an address. The consequence of this action is that a blacklisted user can no longer transfer or receive tokens, which will make the first phase of the auction always revert when trying to send the remaining collateral to the borrower, which will completely block the first phase.

After that when the midPoint passes, the malicious user can delay the second phase making use of block stuffing spaming the network with transactions for one or more blocks, to delay the debt repayment from active participants of the protocol. The result of this action would generate bad debt, which will slash all GUILD holders no matter if this is the best lending term out there and GUILD liquidity is worth a lot. This is huge loss of participants capital, without any real reason.
## PoC
https://gist.github.com/NicolaMirchev/3a9d1cb926c6239493980c92136e5da8

## Recommendations

- Remove the transfer to the borrower inside `LendingTerm::onBid` so this function won’t be dependant on external ERC20 logic(blacklists)
- You can introduce a state variable `mapping(address -> uint256) collateralToBeRepayed` or other name and a function ``withdrawRepayedLoanCollatel`which will send the pending reward to the `msg.sender` based on the mapping and decrement it.
- The change inside `onBid` function may look like this:
```diff
// send collateral to borrower
        if (collateralToBorrower != 0) {
-             IERC20(params.collateralToken).safeTransfer(
-               loans[loanId].borrower,
-               collateralToBorrower
-            );
+              collateralToBeRepayed(loans[loanId].borrower) += collateralToBorrower;
        }
```



# L1 -  Malicious guild participant could front-run any lending term proposal and cancel it, so the term is unable to be included and used
## Impact

- [LendingTermOnboarding](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOnboarding.sol#L171) any GUILD holder, which own at least 0.1% of all GUILD tokens is able to propose a lending term. But there is also a [limit](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOnboarding.sol#L188-L191) to propose the same term only once a week.

The contract inherits from OpenZeppelin `Governor` contract, which has a functionality to cancel a proposal from the proposal. This leads to easy to implement attack from malicious actor. He should only own 0.1% of all GUILD tokens and he would be able to front-run any other user term proposal and cancel it, which would result in “delaying term proposal for another week”. The attacker can do the same after a week and so on… The impact is big, because one untrusted person can decide to cancel a term, which may be very successful. This could lead to unhappy community and participants leaving the protoco

## PoC

- Malicious participant see that a lending term, which is with good parameters and all other participants wants it in the mempool
- He front-run `LendingTermOnboarding::proposeOnboard` transaction with the same arguments
- As a result `lastProposal[{communityWantedTerm}]` is set to current timestamp. Proposal is created using the default `Governor::propose` behaviour.
- An attacker call `Governor::cancel` function in the same transaction to cancel his proposal, which will result in unavailability to execute the onboarding of the term, even if it has reached a quorum right after the proposal being loaded.
- This will result in DoS of the provided `healthy` and community wanted term, because of the check inside `Governor::propose` function `require(_proposals[proposalId].voteStart == 0, "Governor: proposal already exists");`
- Even if this check haven’t existed, the community would have to wait 1 week to propose the same term again and even then the same scenario could be repeated
- **Coded PoC** - https://gist.github.com/NicolaMirchev/f1be765686f15d38006c71479a0fa369

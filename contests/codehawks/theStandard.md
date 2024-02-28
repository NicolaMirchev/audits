# The Standard - Findings Report

# Table of contents

- ## Medium Risk Findings
  - ### [M-01. DoS 24 after a user spam `pendingStakes` with small amounts, until `consolidatePendingStakes` would cost more than block gas limit](#H-01)
  - ### [M-02. `deletePosition` will delete user from `holders` , when he has `pendingStakes` and he would lose rewards](#M-01)
- ## Low Risk Findings
  - ### [L-01. If a token is removed from `acceptedTokens`, liquidators won't receive reward](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- Medium: 2
- Low: 1

# Medium Risk Findings

## <a id='M-01'></a>M-01. DoS 24 after a user spam `pendingStakes` with small amounts, until `consolidatePendingStakes` would cost more than block gas limit

**Impact**:

- The problem comes from the complexity of `consolidatePendingStakes`, which iterates over all pending stakes twice if 24 hours has passed.
- A user can in one transaction create 200 pending stakes with same `timestamp` for `createdAt`, which would make function `consolidatePendingStakes` massive gas cost exactly 24 hours after user “attack start”. For all `200` pending transactions `deletePendingStake` would be called, which would read & write > `100` times storage variables. This is close to `100^2` complexity, which for only `SSTORE` operation = `100^2 * **20,000  = 200 000 000` , when block gas limit for Mainnet is only 30 000 000, which is clearly DoS.\*\*
- `consolidatePendingStakes` is used inside: `increasePosition` , `decreasePosition` & `distributeAssets` , which means DoS of those function. And the whole staking system is dependant to those functions, which is really bad. Noone would be able to withdraw his rewards & staked funds.
- Gas cost for the attacker is relatively low

**PoC**:

In the following gist I have provided coded PoC and instructions how to execute it

https://gist.github.com/NicolaMirchev/0322a697c3170958585086f02ce08411

**Recomendation**

- Limit `pedningStakes` per user:

```diff
+   mapping(address => bool) private pendingStakes;

function increasePosition(uint256 _tstVal, uint256 _eurosVal) external {
+       require(!pendingStake[msg.sender], "User has a pending stake");
        require(_tstVal > 0 || _eurosVal > 0);
        consolidatePendingStakes();
        ILiquidationPoolManager(manager).distributeFees();
        if (_tstVal > 0) IERC20(TST).safeTransferFrom(msg.sender, address(this), _tstVal);
        if (_eurosVal > 0) IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _eurosVal);
        pendingStakes.push(PendingStake(msg.sender, block.timestamp, _tstVal, _eurosVal));
+       pendingStake[msg.sender] = true;
        addUniqueHolder(msg.sender);
    }

function consolidatePendingStakes() private {
        uint256 deadline = block.timestamp - 1 days;
        for (int256 i = 0; uint256(i) < pendingStakes.length; i++) {
            PendingStake memory _stake = pendingStakes[uint256(i)];
            if (_stake.createdAt < deadline) {
                positions[_stake.holder].holder = _stake.holder;
                positions[_stake.holder].TST += _stake.TST;
                positions[_stake.holder].EUROs += _stake.EUROs;
                deletePendingStake(uint256(i));
+               pendingStake[msg.sender] = false;
                // pause iterating on loop because there has been a deletion. "next" item has same index
                i--;
            }
        }
    }
```

## <a id='M-02'></a>M-02. `deletePosition` will delete user from `holders` , when he has `pendingStakes` and he would lose rewards

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L161

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L144-L147

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L124C1-L126

## Summary

Inside [`LiquidationPool`](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L149C1-L149C1) a user can decrease his staked amount, which [check](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L161) whether his stake is 0 and if it is, he is removed from holders array. The problem arrises from the parallel structure `pendingStakes` , which hold pending stakes for the users.

## Vulnerability Details

If a user has a position of 1000 staked TST and increment it with another 1000, second 1000 TST would be queued in pendingStakes and would be [added](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L124-L126) to user’s position after 24 hours. If the same user decide to decrease his position with 1000 TST, this would remove him from holders array, because the check only look inside position mapping for the given user:
`if (empty(positions[msg.sender])) deletePosition(positions[msg.sender]);`

```
function empty(Position memory _position) private pure returns (bool) {
        return _position.TST == 0 && _position.EUROs == 0;
    }
```

But when the period of 24 passes positions[user] will be filled with his position, but the user won’t be presented inside holders array.

## PoC

In the following gist I have provided coded PoC and instructions on how to run it.

https://gist.github.com/NicolaMirchev/b50b266f76c8efbc20420d5f71f5ffaa

## Impact

- Calculation errors, due to `getTstTotal`, which iterates over `holders` array
- Missed rewards, because `distributeFees` & `distributeAssets` iterate over `holders` array

## Tools Used

- Hardhat
- Manual Review

## Recommendations

- One solution is to check `pendingStakes` for the user, before deleting him
- Another is to call `addUniqueHolder` inside `consolidatePendingStakes` , when a position is being incremented

# Low Risk Findings

## <a id='L-01'></a>L-01. If a token is removed from `acceptedTokens`, liquidators won't receive reward

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L119-L122

## Summary

I mark the following vulnerability with L, because the probability is low, but the impact is heavy and depending on protocol plan on how to manage tokens, impact could be bigger
Currently there is a whitelist with allowed tokens for collateral when a user borrows. The problem is that on every action, the list is fetched and iterated.

## Vulnerability Details

If a user borrow 100 euro with 200 USDC as collateral, but in some time team decides to remove USDC, because of it's blacklist feature and that it not working well with it. In that moment borrower position would immediately be counted as unhealthy, because [euroCollateral](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L69-L72) is looping trough all accepted tokens dynamically. Two bad impacts from here:

## Impact

- User vault can be liquidated, without he having fault
- If the position is really liquidatable, the funds won't be transferred to stakers, because again only funds inside `acceptedTokens` are used

## Tools Used

Manual Review

## Recommendations

- Save accepted tokens in storage on vault deployment, if you plan to maintain different `acceptedTokens` and safely remove items

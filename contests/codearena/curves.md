## H1 - Users can steal other users `holderfee` by transferring their shares and claiming fee again

- **Summary**:
    
    **Multiple Claims on Same Token Holdings, because transfer doesn’t adjust `userFeeOffset`  to the current amount of `cumulativeFeePerToken`**. 
    
    **Impact:**
    
    This protocol supports rewards for holding a share of given tokens, which are distributed to [`FeeSplitter`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L89-L94) on user [`buy/sell`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L246-L249). The fee calculation for holders is based on two important params: `data.cumulativeFeePerToken` and  `data.userFeeOffset[account]`. 
    
    `uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;`. 
    
    When user claim his rewards, `data.userFeeOffset[account]` is set to be equal to `data.cumulativeFeePerToken` and this prevent him from claiming the same fee more than once. When a new token purchase is made,  `data.cumulativeFeePerToken` is increased, so the calculation from above will return only the new reward. A problem arrises from the possibility of a user to transfer his tokens to another address and not updating `data.userFeeOffset[account]`, for this new address, which means that he can call [claimFees](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L80-L86) again, which will transfer him the same amount, which is reserved for other holders. This means direct theft of other users holding rewards, which is serious, because nothing stops an exploiter from doing so for all `curvesTokenSubject`s and the whole amount of the holding reward, which can be X100 of what he originally deserves.
    
    **PoC**
    
    https://gist.github.com/NicolaMirchev/ef6c010fc0f264d6902e574e67f12773
    
    **Recomendations**
    
    When calling [`transferCurvesToken`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L296-L299), you should also update `data.userFeeOffset[account]` for the `to` account. The same way you are doing in [`_transferFees`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L247-L248):
```
  function _transfer(address curvesTokenSubject, address from, address to, uint256 amount) internal {
        if (amount > curvesTokenBalance[curvesTokenSubject][from]) revert InsufficientBalance();

        // If transferring from oneself, skip adding to the list
        if (from != to) {
+            feeRedistributor.onBalanceChange(curvesTokenSubject, to);
            _addOwnedCurvesTokenSubject(to, curvesTokenSubject);
```

## H2 - Any `curvesTokenSubject` can stop his follower from selling his token, by implementing malicious `fallback` when receiving fee
**Summary**

When someone [`buy`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L211) or [`sell`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L282),  [`_transferFee`] (https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L218) function is called, which is responsible for sending part of the payed amount by user to ‘protocol’, ‘owner’, or ‘referral’.

**Impact**:

Owner may don’t have benefit from blocking payments to himself, but can do it, when someone wants to sell his tokens. This is easily achievable, as we have `call` with checked return value to both addresses controlled by the `curvesTokenSubject`:

```diff
bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0);
            {
                address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;
                uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
                uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
                (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
                if (!success1) revert CannotSendFunds();
            }
            {
                (bool success2, ) = curvesTokenSubject.call{value: subjectFee}("");
                if (!success2) revert CannotSendFunds();
            }
            {
                (bool success3, ) = referralDefined
                    ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
                    : (true, bytes(""));
                if (!success3) revert CannotSendFunds();
            }
```

The only think the owner has to do is implement a `referral` address with `fallback()`, function which always revert. Then owner can enable referral only when he sees that his followers want to sell his tokens. This will result in revert in any transaction for such actions:

[`if (!success3) revert CannotSendFunds();`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L240-L245).

- The impact is DoS of one of the core functionalities of the protocol: `Anyone being able to sale their tokes any time`

**PoC**

Coded PoC and instructions on how to run it here:

https://gist.github.com/NicolaMirchev/e74b9aff2900e1715d3b4510901d8828

**Recommendations** 

- One solution is to implement [pull over push patter](https://fravoll.github.io/solidity-patterns/pull_over_push.html) for transfers to owner and referral
- Another approach is if the call fails, wrap the ETH amount in WETH and send it to the recipient using IERC20.transfer

## H3 - FeeSplitter::setCurves misses access control on setCurves can result in broken functionality  of whole FeeSplitter
**Impact**

`FeeSplitter` is contract responsible for distributing fees to all holders of tokens. It is bounded to `Curves` contract and uses it's state variables to track balance of each user and so his corresponding rewards. Another important part is that `curves` contract, which is state variable in `FeeSplitter` has the role `manager`, which is responsible for the functions `addFees` and `onBalanceChange`, which are triggered on user buy/sell inside `curves`.

The problem is that `FeeSplitter` has public function `setCurves`, which will change this address, which would break the whole functionality of the protocol. 

- Newly set curves may be malicious and can manipulate all rewards with `setFees` and `onBalanceChange`
- `Curves` protocol would be DoS-ed if holder fees are enabled, because `buyCurvesToken` and `sellCurvesToken` will always revert, as the contract no longer has `manager` role and calling `feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);` will revert

**PoC**

1. Original `Curves`token and `FeeSplitter` are enabled 
2. Some time passes and a lot of traffic is generated by the protocol
3. Some malicious user sees that the `setCurves` function is public and call it with contract managed by him
4. All holders are no longer able to buy/sell tokens

**Recomendations**

- Set `onlyOwner` modifier

## M1 - Holders loose rewards, when when call `buyCurvesToken`, or `sellCurvesToken`, without claiming standing fees
- **Summary:**

This protocol supports rewards for holding a share of given tokens, which are distributed to [`FeeSplitter`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L89-L94) on user [`buy/sell`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L246-L249). The fee calculation for holders is based on two important params: `data.cumulativeFeePerToken` and  `data.userFeeOffset[account]`. 

- **Impact**

An issue may arrises, if this feature is enabled and a user, who haven’t still collected his pending rewards, buys another share of the tokens, because [`onBalanceChange`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L247) is called, which overrides `data.userFeeOffset[account]` for the given user to `data.cumulativeFeePerToken` no matter if he has already claimed his pending fees.

[Here](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L73C1-L78) you can see that when this important param is overriden, the calculation for claiming fees for the given participant will be wrong and next time the user calls `claimFees`, his holding rewards, which were accumulated between his buys, won't be present in the calculation:

```diff
function updateFeeCredit(address token, address account) internal {
        TokenData storage data = tokensData[token];
        uint256 balance = balanceOf(token, account);
        if (balance > 0) {
            uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;
            data.unclaimedFees[account] += owed / PRECISION;
            data.userFeeOffset[account] = data.cumulativeFeePerToken;
        }
    }
```

function updateFeeCredit(address token, address account) internal {
TokenData storage data = tokensData[token];
uint256 balance = balanceOf(token, account);
if (balance > 0) {
uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;
data.unclaimedFees[account] += owed / PRECISION;
data.userFeeOffset[account] = data.cumulativeFeePerToken;
}
}

**Example**:

1. Bob buys 1 share of `EdSheren` and `holderFee` feature is enabled
2. `userFeeOffset` in `FeeSplitter` for Bob for EdSheren is now 1, because `cumulativeFeePerToken` is also 1 for the single share
3. Other users also buy shares. Lets say in total of 50. Now if Bob calls `claimFees` , the calculation for his reward would be `50 {cumulativeFeePerToken} - 1 {Bob's offset} * 1 {Bob's balance} = 49`
4. But Bob isn’t aware of the bug and want to accumulate more fees and then collect them, so he saves money on paying gas
5. He want to buy 1 more token of `EdSheren`, but when he does this, userFeeOffset[bob] is set to 50 and now the calculation from previous point would be 0, which means he has collected the fees, or it is just new holder, but neither is true.
6. Result is lost of holder fee for 49 tokens for Bob :(

 **Note**: The same impact may appear when Bob sell his token

- Also every time he buy/sell token’s address would be added inside `FeeSplitter::userTokens`, which means duplicates of the same token

**Coded PoC**:

- Coded PoC and instructions on how to run it

https://gist.github.com/NicolaMirchev/ecd37fa15e3a9fde60113b4a21aea277

**Recommendations:**

- Inside `FeeSplitter::onBalanceChange` update fee credit  for the user for the given token, before `data.userFeeOffset[account] = data.cumulativeFeePerToken;`. :

```diff
function onBalanceChange(address token, address account) public onlyManager {
        TokenData storage data = tokensData[token];
+        updateFeeCredit(token, account);
        data.userFeeOffset[account] = data.cumulativeFeePerToken;
        if (balanceOf(token, account) > 0) userTokens[account].push(token);
    }
```

- This will update `unclaimedFees[account]` for user and later he would able to claim all standing rewards

## M2 - Malicious use can block deployment of default ERC-20 tokens by creating token with `symbol = _curvesTokenCounter+1`
### Summary
The recent update to the Curves.sol protocol introduces a significant feature: the ability for token holders to convert their curvesTokenSubject into its ERC20 representation. This upgrade allows token owners to deploy ERC20 tokens with custom names and symbols. Additionally, a key functionality is the option for users to deploy tokens with "default" names and symbols, facilitating easy withdrawal.

### Impact
Vulnerability in Default Token Deployment
The vulnerability arises due to the system's check for pre-existing tokens with identical “symbols” and the capacity for any user to deploy an ERC20 token with an arbitrary symbol. This creates a loophole where a user can preemptively deploy a token with the symbol that's designated for the next default deployment, effectively blocking this default functionality.

Critical Code Examination:
if (symbolToSubject[symbol] != address(0)) revert InvalidERC20Metadata();
This line of code in Curves.sol checks if a token with the given symbol already exists. If it does, the deployment of a new token with the same symbol is blocked.

Scenario of Exploitation:
Consider the initial state where _curvesTokenCounter is set to 0, as it is upon deployment. If a user promptly calls buyCurvesTokenWithName with SYMBOL = CURVES1, all subsequent attempts for default ERC20 deployment will fail at this check:

  // If the token's symbol is CURVES, append a counter value
        if (keccak256(bytes(symbol)) == keccak256(bytes(DEFAULT_SYMBOL))) {
            _curvesTokenCounter += 1;
            name = string(abi.encodePacked(name, " ", Strings.toString(_curvesTokenCounter)));
            symbol = string(abi.encodePacked(symbol, Strings.toString(_curvesTokenCounter)));
        }

        if (symbolToSubject[symbol] != address(0)) revert InvalidERC20Metadata();

Resulting Impact:
This vulnerability leads to a Denial of Service (DoS) regarding one of the protocol's main features: the seamless withdrawal of subjectTokens without relying on the token owner to deploy the ERC20 representation.

Proof of Concept
Coded PoC with instructions on how to run it:
https://gist.github.com/NicolaMirchev/017a42825773b031a40485a534fb8025
Tools Used
Manual Review
Hardhat

Recommended Mitigation Steps
Restricting Symbol Parameter: Implement limits on the Symbol parameter when setting a custom name and symbol for a token.
Validation Against “CURVES” Keyword: Include a check to ascertain if the provided Symbol parameter contains “CURVES”. If it does, disallow the user from setting such a symbol to prevent abuse of the default deployment feature.


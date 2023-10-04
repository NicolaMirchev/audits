# M1 - Create pool expect amount greater than intended `baseFee + amount`

## Summary

If `baseFee > 0`, user should always overpay, because of the messed checks inside `_createPool` function:

```
if (baseFee > 0) {
            // To prevent paying the baseFee from the Allo contract's balance
            // If _token is NATIVE, then baseFee + _amount should be >= than msg.value.
            // If _token is not NATIVE, then baseFee should be >= than msg.value.
            // @audit the rule is that msg.value should always be grater than the necessary amount.
            // If base fee is 10 and amount is 100, then msg.value of 110 should be enough, but the transaction will revert and will always
            // expect payment in advance.
            if ((_token == NATIVE && (baseFee + _amount >= msg.value)) || (_token != NATIVE && baseFee >= msg.value)) {
                revert NOT_ENOUGH_FUNDS();
            }

            _transferAmount(NATIVE, treasury, baseFee);
            emit BaseFeePaid(poolId, baseFee);
        }
```

## Details

The problem is here `(baseFee + _amount >= msg.value)`. The comment is misleading, because `baseFee + _amount should be >= than msg.value` is not true, because if it is > than msg.value, than paying baseFee would be done from Allo balance. And inside `if` statement it is presented the same inequality, which is a rule for when to revert, which is okay, but `=` will make the function revert every time user provide exactly the required baseFee, which is not the intended behaviour.

## Impact

Poor UX, unintended contract behaviour. Neccessity to pay more from users account.

## Code Snippet

https://github.com/sherlock-audit/2023-09-Gitcoin-NicolaMirchev/blob/d809326fe078fa4427ab44db013d9f2f1c1aad09/allo-v2/contracts/core/Allo.sol#L473

## Tool used

Manual Review

## Recommendations

Change remove `=` from inside the `if` statement:

```
 if ((_token == NATIVE && (baseFee + _amount > msg.value)) || (_token != NATIVE && baseFee > msg.value)) {
                revert NOT_ENOUGH_FUNDS();
            }
```

# M2 - No check for address(0) on allowedTokens in VotingMerkleStrategy could result in undesired behaviour

## Summary

In `DonationVotingMerkleDistributionBaseStrategy` in the initialization we should provide tokens allowed for voting. If none are provided, it is assumed that all tokens are accepted. But what if allowed tokens are provided and unintentially of is set to be the address(0). This will in having voting strategy with all allowed tokens, because marking `address(0) = true` in the map of the allowed tokens means all tokens are allowed.

## Details

Here are code snippets showing how such user mistake could lead to unintended behaviour - having unlimeted options of voting tokens.

- Imagine one of the tokens in the provided array is address(0)

```solidity
        if (allowedTokensLength == 0) {
            // all tokens
            allowedTokens[address(0)] = true;
        }

        // Loop through the allowed tokens and set them to true
        for (uint256 i; i < allowedTokensLength;) {
            // @audit if inside allowedTokensArray there is a zero address, it will be interpreted as all tokens
            allowedTokens[_initializeData.allowedTokens[i]] = true;
            unchecked {
                i++;
            }
        }
```

Here we can see that if we set `allowedTokens[address(0)] = true`, this will result in passing checks for allowed tokens, which consist in:

```solidity
   // The token must be in the allowed token list and not be native token or zero address
        if (!allowedTokens[token] && !allowedTokens[address(0)]) {
            revert INVALID();
        }
```

I.e we will never revert.

## Impact

- Unintended behaviour
- Possibility to allocate tokens using potentially malicious ERC20

## Recomendation

Check for `address(0)` and revert if it is passed

```solidity
        // Loop through the allowed tokens and set them to true
        for (uint256 i; i < allowedTokensLength;) {
				if(_initializeData.allowedTokens[i] == address(0)){
				revert("ZERO_ADDRESS");
				}
            allowedTokens[_initializeData.allowedTokens[i]] = true;
            unchecked {
                i++;
            }
        }
```

# M3 - Stucked ETH in VotingMerkleDistributionStrategy contract

## Summery

In `DonationVotingMerkleDistributionDirectTransferStrategy` there is a `receive` function to accept native tokens, but there is no way to get those funds out of the contract.

## Details

`DonationVotingMerkleDistributionDirectTransferStrategy` is the parent contract and neither of its child has a functionallity to do something with the contract's balance.
There is a receive function to accept ETH from any address and from any call, where signature does not match existing methods.

```
   /// @notice Contract should be able to receive ETH
    receive() external payable {}
```

But such functionallity is not needed and could only lead to stucked resources.

## Impact

- Stucked native tokens in strategy contract.

## Recomendations

- Remove `receive` function, or create a function to distribute native tokens.

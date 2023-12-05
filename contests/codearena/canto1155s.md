## M1 - Possibility of sandwich attack for user buying/selling shares

### Summary 
`Market.sol` is using a bonding curve, which ensures that the price of the token increases in a linear fashion as more tokens (shares) are minted or purchased. This means that the ones who mint earlier would pay less and would benefit more of the price and fees that next participants will pay.
### Impact



### Impact
The problem here is the well know [Sandwich attack](https://medium.com/coinmonks/defi-sandwich-attack-explain-776f6f43b2fd), which is possible using front-running user transaction, if it is beneficial for another user. The benefit here comes from the formula, which charges minter  greater amount of the underlying token depending on the total minted amount. The same stays for selling options. Whoever sell first, he receives greater amount. If an attacker combine both approaches, he could frontrun a depositor, so he mint large amount of shares, which would give him leverage for his victim's original transaction. The attacker can then immediatelly sell his shares for larger price. The impact is big, because the attacker could use big amount, which would result is larger paid funds for victim. Also it could result in user funds loss, if a depositor with larger share balance frontruns original user `sell` transaction, which would otherwise be beneficial for him (him = victim, original seller, who have benefit, only if another sell does not happen before his).


### PoC
Here is a [gist](https://gist.github.com/NicolaMirchev/971f21e5a4cd94cb9f05dd304a65b085) with a test, which you should paste inside `1155tech-contracts/src/test/Market.t.sol`  and run it using `forge test --match-contract testSandwichLeverage`

### Mitigation

Implement a slippage check when a user is executing a `buy`, or `sell` transaction.
Also a user can provide an count for the tokens, when he initiates the transaction and later revert if this param mismatch the token count inside the execution.
Example:
```
    function buy(uint256 _id, uint256 tokenCountBeforeBuying, uint256 _amount) external {
        require(tokenCountBeforeBuying ==  shareData[_id].tokenCount, "Token count missmatch");
        require(shareData[_id].creator != msg.sender, "Creator cannot buy");
       ... continue original transaction
    }

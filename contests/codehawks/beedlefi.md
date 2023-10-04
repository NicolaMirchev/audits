# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Protocol assumes token will be only with 18 decimals](#H-01)
    - ### [H-02. User can buy loan for much less than it actually costs](#H-02)

- ## Low Risk Findings
    - ### [L-01. Use modifier instead of repeating the same code block](#L-01)
    - ### [L-02. Not emitting event for important state changes](#L-02)
- ## Gas Optimizations / Informationals
    - ### [G-01. Increments can be unchecked in for-loops](#G-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 24th, 2023 - Aug 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 2
  - Gas/Info: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Protocol assumes token will be only with 18 decimals            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L246

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L384

## Summary
Currently the code assumes that all tokens, which are going to be used for collateral in the platform would be with 18 decimals, but this is not mandatory as long as we don't have ERC20 token restrictions for creating a pool.
This could lead to wrong LTV and eventually big misleading afterwards. Example of such a token is USDC that has 6 decimals only.
## Vulnerability Details
- If the collateral decimals are more than 18, we will probably we able to borrow an asset with a very small collateral amount. (Lower than the systems think it is)
- Also if the collateral decimals are less than 18, the collateral provided by the borrower should be a way bigger than originally intended, so the borrow is valid.
## Impact
Alice has a pool which lends DAI for USDC and max LTV is 75%, which means that if Bob wants to borrow 150 DAI, he should collateralize at least 200 USDC. But here is would be the result if we follow the current logic to calculate the LTV:
```
uint256 loanRatio = (debt * 10 ** 18) / collateral;
```
`(150 * 10e18)/ 200 * 10e6` = `750 * 10^9` , instead of the expected `750` (LTV).
This means that if Bob wants to borrow $150 of DAI, he should provide at least around $2,000,000 USDC. 
Is it worth it? 
I don't think so
## Tools Used
Manual Review
## Recommendations
Dynamically calculate the LTV using decimals() of the collateral.
```
uint256 loanRatio = (debt * collateral.decimals()) / collateral;
```
## <a id='H-02'></a>H-02. User can buy loan for much less than it actually costs            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L466-L491

## Summary
The only requirement for a user to buy a loan is to have his pool, but his pool may consist of
`{ loanToken = {doge coin address} (real world value less than 0.1 USD); }` and set a pool balance of 100.
While the actual loan from the selling pool is `{loanToken = {eth address}, collateralToken = {btc address}` and the loan is for value of 1 (eth), which is a way more expensive than 100 doge coin.

## Vulnerability Details
__PoC__

- 1 - __Alice__ creates a pool {loanToken: ETH, collateralToken: BTC} with {maxLoanRatio : 10 * 1e18}
- 2 - __Alice__ fill the pool balance with 10 ETH (\$18 846.10).
- 3 - __Bob__ borrow 10 ETH for collateral of 1 BTC (\$29 000.00) from __Alice's__ pool.
- 4 - __AliceFromProfile2__ creates a pool {loanToken: DOGE, collateral: DOGE} with same other values as the first pool.
- 5 - __AliceFromProfile2__ fill the pool with 100 DOGE (\$8).
- 6 - __Alice__ create an auction for the loan of Bob.
- 7 - __AliceFromProfile2__ buy her own loan of 10 ETH (\$18846.10) for 10 DOGE (\$0.80).
    Now __Alice__ is able to withdraw her 10 ETH (\$18846.10), because the system points that her pool balance is full and don't have outstanding loans.
- 8 - Currently __Bob__ collateral of 1 BTC (\$29 000.00) could be adopted by __AliceFromProfile2__ for the cost of 10 DOGE (\$0.80).

## Impact
There are a lot of problems here. Even if we assume that borrower will always pay back and receive his collateral. Everyone is happy? Not quite still.
Smart **Alice** could reuse one valuable asset, which costs a lot more than the one with which she is buying her loan after that. Imagine she is only using 1 of her ETH and then buy the loan for less than a dollar and her pool balance is back to 1 ETH. Now imagine doing that over and over again, because everything she needs per 1 ETH is 1 DOGE ... This way she could very fast drain Lender.sol pool of the valuable assets and also obtain interests for assets that in reality she didn't possessed.

## Tools Used
Manual Analysis

## Recommendations

- One solution is to make sure one could buy others loans, only if the assets in his pool are the same as those in the seller, or at least loanToken.

```
 function buyLoan(uint256 loanId, bytes32 poolId) public {
        // get the loan info
        Loan memory loan = loans[loanId];
        // validate the loan
        if (loan.auctionStartTimestamp == type(uint256).max)
            revert AuctionNotStarted();
        if (block.timestamp > loan.auctionStartTimestamp + loan.auctionLength)
            revert AuctionEnded();
            // Check if the tokens are the same
        if(loan.loanToken != pools[poolId].loanToken) revert TokenMismatch();
        if(loan.collateralToken != pools[poolId].collateralToken) revert TokenMismatch();
        // calculate the current interest rate
        uint256 timeElapsed = block.timestamp - loan.auctionStartTimestamp;
        uint256 currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) /
            loan.auctionLength;
        // validate the rate
        if (pools[poolId].interestRate > currentAuctionRate)
            revert RateTooHigh();
        // calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(
            loan
        );
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Use modifier instead of repeating the same code block            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L183

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L199

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L211C1-L211C1

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L222

## Summary
In the main contract we have multiple functions, which are intended to be invoked "only by the lender". The same code check is duplicated multiple times.
## Vulnerability Details
- We have 4 functions with same code check whether msg.sender is the lender of the pool. This is redundant since we have "modifiers", which could make the code cleaner, shorter and more gas efficient for deployment.
- There are some other functions, which are making checks whether each element inside an array meets a condition. We should be careful with refactoring those conditions.
## Impact
Redundant code repetition.
## Tools Used
Manual Review
## Recommendations
Where there are no loops create a modifier and use it for those functions: 
```
    modifier onlyLender(bytes32 poolId) {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        _;
    }

    function addToPool(
        bytes32 poolId,
        uint256 amount
    ) external onlyLender(poolId) {
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
        // transfer the loan tokens from the lender to the contract
        IERC20(pools[poolId].loanToken).transferFrom(
            msg.sender,
            address(this),
            amount
        );
    }
```

- Consider doing the same for some other duplicated checks such as `if (amount == 0) revert PoolConfig();`
- For duplicated checks inside the loops. We can extract those in modifiers too and iterate the array and do the checks. However, this may not be the best solution, because if all checks pass, we will iterate the same array two times, which would be more gas inefficient, so we may want to leave it that way, or extract a helper function, which will only do the check and revert and we will call it inside the for loops.

## <a id='L-02'></a>L-02. Not emitting event for important state changes            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L94

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L101

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L86

## Summary
When changing state variables - events are not emitted.
## Vulnerability Details
There are three onlyOwner functions, which are changing important storage variables, which are used for tax calculations. On such important change, we should emit an event.
## Impact
The system does not record historical state changes.
## Tools Used
Manual Review
## Recommendations
For set... functions emit events with old and new values.

# Gas Optimizations / Informationals

## <a id='G/I-01'></a>G/I-01. Increments can be unchecked in for-loops            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L233

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L233

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L293

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L359

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L438

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L549

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L592

## Summary
In Solidity 0.8+, there’s a default overflow check on unsigned integers, which we can skip when iterating over loops and therefore save gas. The risk of overflow is non-existent for uint256 here.
## Vulnerability Details

## Impact
Pointless gas waste.
## Tools Used
Manual Review
## Recommendations
Consider wrapping with an unchecked block  (around 25 gas saved per instance).
Example in `Lender.sol:438`
```
    /// @notice start a refinance auction
    /// can only be called by the lender
    /// @param loanIds the ids of the loans to refinance
    function startAuction(uint256[] calldata loanIds) public {
-        for (uint256 i = 0; i < loanIds.length; i++) {
+        for (uint256 i = 0; i < loanIds.length;) {
            uint256 loanId = loanIds[i];
            // get the loan info
            Loan memory loan = loans[loanId];
            // validate the loan
            if (msg.sender != loan.lender) revert Unauthorized();
            if (loan.auctionStartTimestamp != type(uint256).max)
                revert AuctionStarted();

            // set the auction start timestamp
            loans[loanId].auctionStartTimestamp = block.timestamp;
            emit AuctionStart(
                loan.borrower,
                loan.lender,
                loanId,
                loan.debt,
                loan.collateral,
                block.timestamp,
                loan.auctionLength
            );
        }
+     unchecked { ++i};
    }
```

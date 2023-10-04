# Foundry DeFi Stablecoin CodeHawks Audit Contest - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. 'staleCheckLatestRoundData()' does not check for round completeness](#M-01)
    - ### [M-02. Price feed is assumed to always return value with 8 decimals](#M-02)
- ## Low Risk Findings
    - ### [L-01. _revertIfHealthFactorIsBroken(msg.sender) on burnDsc will revert when user try to make his healthFactor grow](#L-01)
    - ### [L-02. No price feed token validation in the constructor](#L-02)
- ## Gas Optimizations / Informationals
    - ### [G-01. modifier isAllowedToken could be used in more places to prevent underflow revert and provide gas optimization](#G-01)
    - ### [G-02. Increments can be unchecked in for-loops](#G-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Cyfrin

### Dates: Jul 24th, 2023 - Aug 5th, 2023

[See more contest details here](https://www.codehawks.com/contests/cljx3b9390009liqwuedkn0m0)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 2
   - Low: 2
  - Gas/Info: 2


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. 'staleCheckLatestRoundData()' does not check for round completeness            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/libraries/OracleLib.sol#L32

## Summary
Chainlink data feed is maybe the most important component in the project, on which everything depends. If the data from it is not correct, but we don't understand, everything else could go wrong - liquidation, health factors, etc. Therefore we should take all measures to prevent this from happening.

## Vulnerability Details
Currently the code is only checking if the round has been updated more than three hours ago, but in chainlinks [documentation](https://docs.chain.link/data-feeds/historical-data) it is suggested to make two more checks with the returned result.
 
## Impact
The likelihood is low, but it can lead to misleading, false results, if oracle system is not working as it should.

## Tools Used
Manual Review

## Recommendations
Add two more checks using the returned values: 
```
  function staleCheckLatestRoundData(
        AggregatorV3Interface priceFeed
    ) public view returns (uint80, int256, uint256, uint256, uint80) {
        (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();

        uint256 secondsSince = block.timestamp - updatedAt;
        require ( answer > 0, " Chainlink price <= 0");
        require ( updatedAt != 0, " Incomplete round ");
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();

        return (roundId, answer, startedAt, updatedAt, answeredInRound);
    }
```

## <a id='M-02'></a>M-02. Price feed is assumed to always return value with 8 decimals            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L366

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L70

## Summary
I mark it as 'medium' , because the chance is very little, but if it occurs it will break the whole project logic.
`The system is meant to be such that someone could fork this codebase, swap out WETH & WBTC for any basket of assets they like, and the code would work the same.` The following sentence from the project documentation give us reason to be careful no to hardcode WETH & WBTC specific things in the engine. `getUsdValue` is using ChainLink datafeed, which in the current implementation is assuming that the return value will always be with 8 decimal places value. But in https://docs.chain.link/data-feeds/price-feeds/addresses we can see that AMPL/USD returns 18 decimal places result, which could result in larger confusion in the price calculations.

## Vulnerability Details
PoC
We assume that:
`1 AMPL = $100`
Chainlink datafeed for AMPL/USD will then return
`10 000 000 000 0000 000 000 = 100 * 18 decimal places`
The current code formula
```
  function getUsdValue(
        address token,
        uint256 amount
    ) public view returns (uint256) {
        AggregatorV3Interface priceFeed = AggregatorV3Interface(
            s_priceFeeds[token]
        );
        (, int256 price, , , ) = priceFeed.staleCheckLatestRoundData();
        // 1 AMPL = $100
        // The returned value from CL will be 100 * 1e18
        // 100 00000000 00000000 * 00000000 
        
        return
            ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
    }
```
will return a value of `1 AMPL = 10 000 000 000 USD`, which is faaar away from the truth -> `1 AMPL = 100 USD` 

## Impact
Low likelihood with high bad impact on price conversions if it occurs.

## Tools Used
Manual Review

## Recommendations
Calculate `ADDITIONAL_FEED_PRECISION` the following way:
```
  function getUsdValue(
        address token,
        uint256 amount
    ) public view returns (uint256) {
        AggregatorV3Interface priceFeed = AggregatorV3Interface(
            s_priceFeeds[token]
        );
        (, int256 price, , , ) = priceFeed.staleCheckLatestRoundData();
    	
		// If price feed of the token returns 18 decimals 	    (as it is with AMPL/USD),
		// then the additionFeedPRecision would be 0
		uint256 additionalFeedPrecsision = PRECISION - priceFeed.decimals();
		
        return
            ((uint256(price) * additionalFeedPrecsision) * amount) / PRECISION;
    }
```
- The same stands for `getTokenAmountFromUsd()`
The additionalPrecision should be calculated from dataFeed.decimals()

# Low Risk Findings

## <a id='L-01'></a>L-01. _revertIfHealthFactorIsBroken(msg.sender) on burnDsc will revert when user try to make his healthFactor grow            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L214

## Summary
 It is possible to hit this if nobody has liquidated a user and collateral prices have dropped significantlyThere is a possibility for a user to burn dsc and therefore make his health factor better, but still broken.
## Vulnerability Details
We don't want to revert when someone has make their health factor better (even if it is still broken) and even burn gas fee for that.
## Impact
The operation in this location is useless and won't allow user to strengthen the system, when burning DCS, without redeeming collateral.
## Tools Used
Manual Review

## Recommendations
Delete this line `244.        _revertIfHealthFactorIsBroken(msg.sender); // I don't think this would ever hit...`
## <a id='L-02'></a>L-02. No price feed token validation in the constructor            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L114-L120

## Summary
When constructing our DSCEngine we should provide array of token addresses, which could be used as collateral and Chain Link price feed addresses for the exact tokens. However the only check that is currently there is that the number of the tokens is the same as the number of the price feeds. If by any mistake the creator has provided ETH address on first place in addresses array and the corresponding price feed address (ETH / USD) on the second place in price feed array, the engine  will be created if the sizes of the arrays match and this will lead to whole broken contract, because calculating given collateral asset using wrong price feed won't provide real data. 
## Vulnerability Details
It is important to validate such important fields, because this is the logic, which holds the whole application. We have an invariant for the data feed (TOKEN1 / TOKEN2). We want TOKEN2 to always be USD. However there is never check for that, so creator could unintentionally to mismatch price feed address and we won't be aware, until later when a user deposit 1 ETH (\$1 800), expecting to be able to mint 900 DSCE (\$900), but indeed he would be able to mint less than 0.5, if when wrong pricefeed is set to ETH / BTC for example. This immediately breaks the our coin properties stability and being USD pegged.
## Impact
 __The following example is for mismatching the places of the token addresses and their corresponding price when creating the contract. I think this the scenario, which is with highest probability, but the code is also vulnerable of providing data, which does not returned the value compared to USD.__  
PoC
1. 1 - Creator of the contract (Bob) want to make the engine work with ETH and BTC as collaterals. 
2. 2 - Bob unintentionally deploy the contract providing arrays as follows : `tokenAddresses = [ethAddress, btcAddress], priceFeedAddresses = [btcUsdPriceFeed, ethUsdPriceFeed]`.
3. 3 - This will pass the check `  if (tokenAddresses.length != priceFeedAddresses.length) {revert DSCEngine__TokenAddressesAndPriceFeedAddressesMustBeSameLength();}` ,  but there is a big problem, because logic that follows is assuming that index `x` of the tokenAddress array corresponds to the price feed for same token in the price feed array for `x`.
4. 4 - The contract will be created and inside `s_priceFeeds` we will have -> `[{ethTokenAddress} = {btcUsdPriceFeedAddress}, {btcTokenAddress} = {ethUsdPriceFeedAddress}]`.
5. 5 - This will result in DSCE token price very different from the main project idea (1 DSCE = 1 USD).
6. 6 - Because when Alice buy 1 ETH for \$1 800 from an exchange and want to deposit inside our broken contract, she will be pleasantly surprised. Instead of being able to withdraw DSCE 900, because if we want 200% overcollateralized system and token which is being pegged to the USD, this is the expected value, she will be able to mint DSCE 15 000, because the BTC price feed has returned that for 1 (She provived ETH, but we have linked it to BTC price feed) of the collateral token corresponding prize in USD is \$30 000 (For example, the ratios of \$1 800 ETH and \$30 000 is accurate) and to be able to mint half of the collateral value makes 15 000.
7. 7 - Now she doesn't care if the system will take her collateral worth of \$1 800, because she has DSCE worth of \$15 000, or no. Because this broken model will change the corresponding price of DSCE and won't be pegged to USD anymore.

## Tools Used
Manual Review
## Recommendations
Consider checking the ERC20 "symbol()" of the current token and the "description()" of the corresponding price feed while iterating over the array in the constructor: 
```
   error DSCEngine__ProvidedPriceFeedDoesNotMatchProvidedToken(
        address priceFeed,
        address tokenAddress
    );

        for (uint256 i = 0; i < tokenAddresses.length; i++) {
            // Price feed token symbols in format: "ETH / USD"
            string memory priceFeedTokenSymbols = AggregatorV3Interface(
                priceFeedAddresses[i]
            ).description();
            // Current collateral token symbol in format: "ETH"
            string memory currentCollateralToken = ERC20(tokenAddresses[i])
                .symbol();

            if (
                keccak256(abi.encodePacked(priceFeedTokenSymbols)) !=
                keccak256(
                    abi.encodePacked(
                        string.concat(currentCollateralToken, " / USD")
                    )
                )
            ) {
                revert DSCEngine__ProvidedPriceFeedDoesNotMatchProvidedToken(
                    priceFeedAddresses[i],
                    tokenAddresses[i]
                );
            }

            s_priceFeeds[tokenAddresses[i]] = priceFeedAddresses[i];
            s_collateralTokens.push(tokenAddresses[i]);
        }
        i_dsc = DecentralizedStableCoin(dscAddress);
    }
```


# Gas Optimizations / Informationals

## <a id='G/I-01'></a>G/I-01. modifier isAllowedToken could be used in more places to prevent underflow revert and provide gas optimization            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L169

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L229

## Summary
If we pass wrong token address on those functions they will revert in the operation of ` s_collateralDeposited[from][tokenCollateralAddress] -= amountCollateral;` which will underflow the balance of the token address(0), which is a bad place to capture this error. The catch in the modifier will also save gas for the msg.sender.
## Vulnerability Details
Also there is a possibility for unpredictable behavior when we hit 
```
 function getUsdValue(address token, uint256 amount) public view returns (uint256) {
        AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
        (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
```
## Impact
Bad traceability on the actual error.
## Tools Used
Manual Review
## Recommendations
Use `isAllowedToken(address)` on "liquidate" and "redeemCollateral" functions:
```
 function liquidate(address collateral, address user, uint256 debtToCover)
        external
        moreThanZero(debtToCover)
        nonReentrant
        isAllowedToken(collateral){
...}

   function redeemCollateral(address tokenCollateralAddress, uint256 amountCollateral)
        public
        moreThanZero(amountCollateral)
        nonReentrant
        isAllowedToken(tokenCollateralAddress)
    {
...
}
```

## <a id='G/I-02'></a>G/I-02. Increments can be unchecked in for-loops            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L353

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L118

## Summary
In Solidity 0.8+, there’s a default overflow check on unsigned integers, which we can skip when iterating over loops and therefore save gas. The risk of overflow is non-existent for uint256 here.
Here we loop through all the tokens, which are valid for collateral.
## Vulnerability Details
As the project is a template for a stable coin, which other developers could use as a ready solution, we are not sure about the size of the collateral array, so the optimization could save a good amount of gas.
## Impact
Gas usage
## Tools Used
Manual Review
## Recommendations
Consider wrapping with an unchecked block  (around 25 gas saved per instance).
```
 function getAccountCollateralValue(address user) public view returns (uint256 totalCollateralValueInUsd) {
        // loop through each collateral token, get the amount they have deposited, and map it to
        // the price, to get the USD value
        uint256 allTokens = s_collateralTokens.length;
        for (uint256 i = 0; i < allTokens;) {
            address token = s_collateralTokens[i];
            uint256 amount = s_collateralDeposited[user][token];
            totalCollateralValueInUsd += getUsdValue(token, amount);
          unchecked {++i};
        }
        return totalCollateralValueInUsd;
    }
```


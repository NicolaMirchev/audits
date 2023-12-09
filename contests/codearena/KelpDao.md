## Lines of code
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTConfig.sol#L17
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTConfig.sol#L109-L122
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/NodeDelegator.sol#L121-L124

## Vulnerability details

EigenLayer has a deposit limit for strategies and this limit could be reached by any regular user in the blockchain, because the contracts are public. Currently Kelp protocol plans to set strategies implemented by Eigen team, but in some situations as reach max deposit limit for strategy, or implemented another strategy, which fits better for kelp protocol, it is possible to update strategy contract for a given asset. And the term update maybe is misleading, because a user should be responsible for the assets that he had deposited and when and whether to withdraw them. But even if he hasn't and the strategy is updated, the oracle will calculate only the deposited assets in the new strategy when calculating reETH price.

## Impact
Here are the lines, where Oracle calculate the deposits based on the current strategy:
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTOracle.sol#L70
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L47-L51
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L84
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/NodeDelegator.sol#L121-L124

## Proof of Concept
Lets examine the following scenario:

Protocol is initialised with the default strategy for rETH.
Some time passes and there is big demand on using the protocol, so the team decides to introduce new strategy, because there is a better one and also the deposit for the last last will be reached soon
The admin updates the strategy map for rETH to the new strategy and now the function used in the Oracle, which gets the total deposited amount into EigenLayer will return misleading data, which would lead to vulnerable behaviour for minting new rsETH tokens.
If in the prev strategy user has deposited 1000 rsETH and in the new strategy rsETH is currently 0, here is the result of the first depositor for the new strategy: (Assuming r:rsETH:ETH = 1:1:1 for easier calcualtions and that all deposited funds are located in the EigenLayer)
Bob deposits 10 rETH and getRsETHAmountToMint() function, which uses ORTOracle will return either 0 (if rounding error is not handled) or will always revert
Because of the calculation here
Even if the division problem is handled properly, the data from this calculation cannot be trusted and is not accurate
## Tools Used
Manual Review

## Recommended Mitigation Steps
There is a function-getAssetBalances(), which uses EigenLayer StrategyManager.getDeposits, which returns all the deposits for all strategies that Delegator has triggered. Just use this function for Oracle calculations:
```
 function getAssetDistributionData(address asset)
        public
        view
        override
        onlySupportedAsset(asset)
        returns (uint256 assetLyingInDepositPool, uint256 assetLyingInNDCs, uint256 assetStakedInEigenLayer)
    {
        // Question: is here the right place to have this? Could it be in LRTConfig?
        assetLyingInDepositPool = IERC20(asset).balanceOf(address(this));

        uint256 ndcsCount = nodeDelegatorQueue.length;
        for (uint256 i; i < ndcsCount;) {
            assetLyingInNDCs += IERC20(asset).balanceOf(nodeDelegatorQueue[i]);
            (assets[], balances[]) += INodeDelegator(nodeDelegatorQueue[i]).getAssetBalances();
            // TODO: 
            // Implement logic to calculate only requested asset deposited values
            // Maybe using a nested loop (Not optimal solution, but working one)
            unchecked {
                ++i;
            }
        }
    }
```
Or add new function, which uses StrategyManger deposits to query all deposits for the given delegator and all strategies.

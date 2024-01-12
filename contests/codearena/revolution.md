## M1 - `MaxHeeap::extractMax()` doesn't erase valueMapping & possitionMapping

- MaxHeap.sol has a crucial role for the whole protocol as it stores and filters the voting for the pieces and it is very important to work correctly
- However if we take a look at extractMax() we can notice something very suspicious:
- We remove the item with maximum value (first one in the heap), we set the last item on his place and decrement size. We can notice that we actually haven't removed the last item from his previous place inside heap and now we store the same piece id on two places, which is wrong.
Note that: major imapct that comes from this root would araise if we call updateValue with new value, which is smaller than the previous, which would be the case if the protocol has functionality to downvote, which they play to implement in future. However the issue is regarding a data structure implementation, which is whole in the scope of the audit and it is confirmed and valuable for the sponsor:

### PoC
I have created detailed diagram showing the state of the heap after each operation. Diagram is inside the gist:

Here is a coded PoC with detailed comments.
You can paste it inside /packages/revolution/test/max-heap/Updates.t.sol , import import "forge-std/console.sol"; at the beginning of the file and run:
FOUNDRY_PROFILE=dev forge test --match-test "testUpdateValueExtractMaxDoesntEraseLastValue" -vv at ./revolution directory


## M2 - DoS of `AuctionHouse` due to large creators tollerance and external calls gas limit

### Summary 
Observing the main and most important flow of [revolution protocol](https://github.com/code-423n4/2023-12-revolutionprotocol) (mint the winning NFT and start auction) we can immediately see the complexity of the nested functions and actions. Since we work in blockchain gas cost and execution is crucial as it costs money and in some situations it could result in DoS. Currently the flow consist of settling an auction, transferring rewards, minting erc20votes, extracting value from constantly increasing heap and minting new NFT with arbitrary string data. All this should happen in one transaction initiated inside [`settleCurrentAndCreateNewAuction`](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L152). But the worst part is the large tollerance of untrusted addresses, which are called, forwarding 50_000 gas.
### Details
Lets carefully observe the flow of settling an auction:
- [_settleAuction](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L336) function would [iterate over the verb creators](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L383-L395) and [`call`](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L429)
- We can note that creators limit is set to [100](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/CultureIndex.sol#L75)
- So worst case scenario, which is a valid and real one is to have 100 calls, each consuming 50_000 gas = 5_000_000 gas only for transffering rewards, which for current state of mainnet (gas = 45 gwei; ETH = $2250) is ~ 0.225 ETH = ~$500 in gas only for a couple of lines of this complext function flow.
- In the nested function [`ERC20TokenEmitter::buyTokens`](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L152), which is also with great complexity, we again [iterate](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L209-L215) over the creators array.

I have coded a [PoC](https://gist.github.com/NicolaMirchev/2e7868233240e0d0f3d613c823f90c96), where you can examine that the issue is actually bigger, as the whole flow could exceeded Mainnet gas limit, which would result in DoS. Even if admin [`pause`](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L208) and call only [`settleAuction`](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L161C14-L161C27), the gas used for the flow is > 30_000_000 which is mainned average gas limit.
- To run the test paste the provided `MaliciousParticipant.sol` inside `packages/revolution/test/auction/AuctionSettling.t.sol`
- add the provided test function inside the test contract 
- navigate to root foundry directory (./revolution)
- run `FOUNDRY_PROFILE=dev forge test --match-test "testSpendGasAmountOnLargeCreatorsArray" -vv` for faster execution

### Impact
- The impact is big, because griefing creators contract may go unnoticed and reach quorum because of the good art this malicious actor has provided. This is the enough to reach a DoS on Ethereum Mainned, which a sponsor has confirmed that is supported network `Mainnet included`
- This will stop the possibility to mint new nfts and start auction, which could result in bad reputation and loss of participants

### Recomendation
- A maximum of 5000 gas should be enough for ETH transffer purpose
- Consider a way to reduce the complexity of the flow and nested operations. Maybe calculating and minting ERC20 votes should happen in another transaction and an amount from reward would go for the gas, because even for 5 greifing creators acoounts (which could be one acount repeated 5 times) the spent for the flow is 1_000_000 gas = ~ $100 on Mainnet.



## M3 - Gas griefing due to unchecked string values, which are copied in memory
### Details:
We can see that when a pice is created there is a [validation](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/CultureIndex.sol#L160-L168) for the provided `mediaType`, but we should note that `ArtPieceMetadata` has multiple string fields, which are never checked:
```
    struct ArtPieceMetadata {
        string name;
        string description;
        MediaType mediaType;
        string image;
        string text;
        string animationUrl;
    }
```
On a later stage (if a piece has reached quorum and is most voted) all those values are [copied](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/VerbsToken.sol#L299) inside `VerbsToken`. This operation is crucial, because it is part of the flow to close an auction and start another, which is getting the most voted piece.
It is possible for a user malliciously or not to put a large string data when creating a piece, so later the fees are extreamly high and unpayable. In the PoC section I have tested with a large text for description, which is 580 wrods (A normal amount if someone want to include a narrative or story to his art). In this case gas spent for only opening the auction is `3371079`, which for Ethereum Mainnet would cost:`60 (gwei) * 3371079 = 202264740 ~ $400`, which is a lot and can be a lot larger, because currently there is no validation.
Note that also in testing enviroment we have an empty heap, which in real application with some time functionating would add more operations , as extracting and ordering heap is part of the flow

## PoC
Here is a simple [test](https://gist.github.com/NicolaMirchev/1a0f9da7312ed17548bff6f8029374e4), which you can run in any of files inside `packages/revolution/test/auction` using `FOUNDRY_PROFILE=dev forge test --match-test "testGasCostForLargeDescription" -vv` when you are inside `./revolution`

## Recomendations
- Add a validation for the metadata fields to reduce unpleasant consequences
- Example:
```
uint256 public constant MAX_METADATA_FIELDS_L = 1000;

function validateMetadata(ArtPieceMetadata calldata metadata) internal pure {
      	uint256 allFieldsLength = metadata.name.length + metadata.description.length + metadata.image.length + metadata.text.length + metadata.animationUrl.length;
        require( allFieldsLength <= MAX_METADATA_FIELDS_L);

    }
```

## М4 Edge case revert in `RewardSplit::computeTotalReward`  can block auction settlement
- Lets say `entropyRateBps` is set to 80% (80% of creator's shares would be payed in ether)
- - `creatorRateBps` is 5%
- - Winning bit is 1 ether, so 

- There are many parameters in the system, which prevent this scenario, but neither of them is guaranteed and has a check for min/max amount
- The problem here is that in specific case, where `reservePrice` and winning bid are close to zero and `entropyRateBps` is high enough, settling an Auction can hit [`INVALID_ETH_AMOUNT`](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/protocol-rewards/src/abstract/RewardSplits.sol#L41) and block new auction from starting. 

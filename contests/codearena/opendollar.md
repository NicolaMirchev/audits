## M1 Changing SafeManager on 721Vault will cause unexpected behaviour (DoS | Lost funds)
## Summary 
In `Vault721` it is possible to change the implementation of SafeManager, which tightly coupled to the NFV. The function is only callable by the governor, but when it is called it could lead to blocked oppertunity to open new safes(Mint), fund loses, because if transfer(id) is called on the new implementation, safe information would be changed to new owner and then implementation is returned to the old version, to resume the possibility to mint tokens (the ownership of the safes for all transfers on the nft-s, which has occured on the implementation, which stopped minting, won't match the ownership of the nfts, which is a funds loss for one of the parties in the trade that happened) 

## PoC:
Execute PoC:
Inside `test/nft/anvil/NFT.t.sol` import `import {ODSafeManager} from '@contracts/proxies/ODSafeManager.sol';
import {ODProxy} from '@contracts/proxies/ODProxy.sol';` at the top and then place the following [test](https://gist.github.com/NicolaMirchev/7b31536ec888977c9123e29de14177fb)

`forge t --fork-url $ANVIL_RPC --match-contract NFTAnvil --match-test test_setSafeManagerWillDoSFunctionalities -vvvv`

## Reccomendations:
I don't know what is actually the best solution here. Maybe an architecture design change, where safe data is always stored and a proxy implementation, which would be the `ODSafeManager` address to change inside `Vault721` 

---
---

## M2  Many safe functionalities won't work in the current implementation due to the delegateCall of the proxy execute

## Summary

Each user has a proxy, which is the owner of the user corresponding safes. The proxy is used so many transactions could be combined together and so safe gas. The problem here is that if there is no "delegator" between the user's proxy and the `ODSafeManager` all call will revert, because  `safeAllowed(_safe)` modifier compares msg.sender with the ___safe's__ owner, who is the proxy address of the original user address. Note that if the user executes calls via his proxy directly to the `ODSafeManager` __msg.sender__ would be the actual user(because of the `delegatecall`) and not the proxy contract. Furthermore - all changes from this call won't affect the storage of the __manager__ contract.

## Impact
The following functions are not executable in the current protocol context:
- allowSAFE()
- quitSystem()
- enterSystem()
- moveSAFE()
- removeSAFE()
- protectSAFE()

## PoC
Here is a [test](https://gist.github.com/NicolaMirchev/5bc158c78ba0c766c784b08730499b7c) presenting the impossibility to allow another address to manage a safe:
Run it inside `test/nft/anvil/NFT.t.sol` using `forge t --fork-url $ANVIL_RPC --match-contract NFTAnvil --match-test test_setSafeManagerWillDoSFunctionalities -vvvv`
Import `import {ODProxy} from '@contracts/proxies/ODProxy.sol';` if necessary

## Reccomendations:
Add missing functions to the "delegator" `BasicActions.sol`

---
---

## M3 - Collateral could be transffered to an address, which is not `SAFEHandler` managed by the `SAFEManager`

## Summary
In the current implementation each safe has corresponding proxy contract inside `ODSafeManager` and corresponding `SAFEHandler` "proxy" which is deployed for each safe and used as safe's address inside `SAFEEngine`.
The problem is that inside `ODSafeManager` `transferCollateral()` function there is no check wether the provided address is a valid safe and so the collateral could be "transffered" to any ordinary address.
## Impact
This could result in contracts missbehaviour, or lost of funds as it breaks the invariant of "Each safe has a corresponindg SAFEHandler" and so other accounting errors could appear as the new owner of the contract can transffer it where he wants directly calling `SafeEngine.sol` contract, which is also unwanted behavour (To execute safe operations, bypassing `SafeManager.sol`)

## Recommendations
The issue is confirmed by sponsor, who came up with a prevention:
![Photo](https://user-images.githubusercontent.com/78795830/278062588-40354bf1-d164-4073-a33d-f26a7ef05c84.png)

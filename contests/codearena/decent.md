
# H1 - `DcntEth::setRouter` doesn't have access control, which result in uncontrolled mint/burn, when exploited

## Impact
DcntEth represents bridged token for decent bridge, which uses LayerZero OFT. In current implementation, DecentRouterEth is responsible for minting DcentEth pegged to WETH to users, which are providing liquidity in the router.
The problem is there is no access control on DecentBridge::setRouter func, which is sets the address authorised for minting/burning DcntEth tokens. By exploiting the following issue, attacker can mint himself tokens equal to all WETH liquidity in the router for free and then redeem the corresponding WETH, belonging to honest participants.
## PoC
1. Everything works as planed. `[DecentRouterEth](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol#L309)` has accrued liquidity of 100 WETH of honest participants.
2. Malicious actor change the router to a contract, which call `DcntEth::mint` with his address and amount of 100e18
https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DcntEth.sol#L24-L26
3.The same actor return the router control over `DcntEth` and calls `[DecentRouterEth::removeLiquidityWeth](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol#L332-L335)`, which successfully transfer him 100 WETH, which has been stolen from all participants
## Recommendations
Set onlyOwner modifier on `DcntEth::setRouter`

# H2 - `DecentBridge` will always refund to wrong address on destination chain, if transaction is reverted, because encoded from address doesn't correspond to the address provided by the participant
## Impact
There is big issue in combining `UTB` and `DecentBridge` in current implementation. `DecentBridge` implements a logic to refund a sender on dest chain, if the call reverts:
https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentBridgeExecutor.sol#L33-L38
The problem is that `from` address, which is used here will always be wrong, because it will correspond to the address of `DecentBridgeAdapter` on src chain, which on dest chain will be some random address. This is also an issue, if we don't use `UTB`, but directly call `DecentEthRouter::bridgeWithPayload` from a Multisig wallet contract, because on destination chain the same address is not controlled by users of the multisig, so their funds are lost.
Lets roughly examine the execution flow:

When using `UTB::callBridge`, the contract/function flow is as follows:

Source Chain (src):

1. `UTB::callBridge`
2. `DecentBridgeAdapter::bridge`
3. `DecentEthRouter::bridgeWithPayload`
4. `DcntEth::_send`

Destination Chain (dest):
5. `DcntEth::onOFTReceived`
6. `DecentEthRouter::onOFTReceived`
7. `DecentBridgeExecutor::execute`

Other contracts on the destination chain, which are not important in the current scenario, may also be involved.

Inside `[DecentEthRouter::bridgeWithPayload](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol#L197C14-L197C31)` we call `[_getCallParams](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol#L80)`, where we encode `destChain`, `adapterParams` and `payload`. Inside the payload `from` address in set, which on destination chain is used as the refund address:
```
    if (msgType == MT_ETH_TRANSFER) {
            payload = abi.encode(msgType, msg.sender, _toAddress, deliverEth);
        } else {
            payload = abi.encode(
                msgType,
                msg.sender,
                _toAddress,
                deliverEth,
                additionalPayload
            );
        }
```
## PoC
I have provided pretty detailed walk trough in the above section, but lets examine the following scenario:

1. Bob want to use Decent UTB to swap USDC for AVAX and bridge from ARB -> AVAX.
2. He wants to swap 2000 USDC for at lest 75 AVAX tokens, slippage params are configured and call to `UTB::callBridge` is executed.
3. `DecentEthRouter::bridgeWithPayload` encodes `from` address param to `DecentBridgeAdaper` on Arbitrum and flow continues.
4. Flow reaches `UTB::performSwap`, but it reverts, because of the slippage check and `minAmountOut` is not satisfactory.
5. All value of 2000 USDC, gas and fees of user are waisted and lost, because `_from` address is not his.
## Recommendation
## 

- Encode the `refundAddress`, instead of `msg.sender`, inside `DecentEthRouter::_getCallParams`
- Maybe you should decode `additionalPayload` to obtain the `refundAddress` previously provided in `UTB::SwapInstructions`
- If you want `DecentEthRouter::bridgeWithPayload` to be used not only from bridge adapters, you should add a `from` param argument to handle the case of multisig wallets

# M1 - `DecentEthRouter::bridge/bridgeWithPayload` can directly be called, which bypasses UTB main entry point with signature
## Impact
The flow to swapAndBridge/bridgeAndSwap assets is long and starts in UTB contract, where participant provides authenticator signature:
https://github.com/code-423n4/2024-01-decent/blob/07ef78215e3d246d47a410651906287c6acec3ef/src/UTB.sol#L259-L266
The problem is that a malicious actor can directly call DecentEthRouter::bridge with unchecked payload data, which means that it can be anything. For example - giving allowance for some ERC20 token from DescentExecutor to the attacker.
https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentBridgeExecutor.sol#L61
Another example is provided data, which is not checked by the signer, but reaches the following line and continues down the flow inside BridgeAdapter::receiveFromBridge and UTB::receiveFromBridge

Impact is bypassing UTB calldata checks and fee calculations
Possibility of executing malicious actions on the behalf of executor contract.
## PoC
One example is:

Malicious actor encodes "abi.encodeWithSelector(IERC20(WETH).approve.selector,maliciousActor,uint256.max)" and provide it as a payload to bridgeWithPayload
The call is bridged and reaches the executor, which runs the provided calldata.
Now the malicious actor can withdraw WETH from the contract, if by any chance there is some left.
## Recommendation
Add a modifier on DecentEthRouter::bridge/bridgeWithPayload so only BridgeAdapter can access it


# M2 - `DecentEthRouter::_bridgeWithPayload` set `refundAddress` to msg.sender, which would be `DecentBridgeAdapter` on src chain
`DecentEthRouter::bridgeWithPayload` configures payload an LZ params before calling dcntEth.sendAndCall(layer zero send functionality).
```
ICommonOFT.LzCallParams memory callParams = ICommonOFT.LzCallParams({
            // @audit : refundAddress is set to `DecentBridgeAdapter` on src chain?
            refundAddress: payable(msg.sender),
            zroPaymentAddress: address(0x0),
            adapterParams: adapterParams
        });
```
In case of LayerZero transaction being reverted, user won’t receive any refund, but the gas that he has pre-paid would be sent to DecentBridgeAdapter and stucked. Worse thing is that other msg.sender(DecentBridgeAdapter) address on dst chain will have another address on other chains, which means that if a refund is being triggered on the destination chain, funds would be send to unknown address.
## PoC
1. User has prepaid 0.005 eth for his expensive tx from Arbbitrum -> Mainnet
2. If the call fails inside LayerZero contracts, the gas would be repaid to DecentBridgeAdapter, or random address
3. The refund is los
## Recommendation
Configure the refund address to be passed to bridgeWithPayload use the param provided from the users:

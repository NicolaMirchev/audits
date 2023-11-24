# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Pausing contract in between user interactions (deposit) could result in lost funds)](#M-01)
    - ### [M-02. Invariant violation (funds could remain in the vault and a depositor could benefit from it) ](#M-02)
    - ### [M-03. It is possible to reopen a deposit after being closed](#M-03)
- ## Low Risk Findings
    - ### [L-01. Consider erasing cache after completing deposit/withdraw/rebalance/compound operations](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 3
   - Low: 1



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Pausing contract in between user interactions (deposit) could result in lost funds)            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L47

## Summary
| Impact      | Likelihood | Overall  |
|-------------|------------|----------|
| High        | Low        | Medium   |


The contract provide functionality to pause the vault in some emergency situation. The function doesn't check the state of the vault, because it is "emergency" and so it should be possible to trigger it any time. However, this could result in user loses, if it is called at the wrong time (in between user interactions like deposit or withdraw).

## Vulnerability Details
The problem comes from the two way transaction process for GMX vaults and safety checks, which are implemented in the protocol. 
For example if a user initiate a deposit transaction with $1000 USDC on vault X, the vault will add those tokens as liquidity to GMX and put the vault in state of "Deposit".
The problem arise from that it is possible to put the vault in state of "Paused", before the callback from GMX router, which would initiate vault tokens being minted and sent to the user. The callback will revert, because the state of the vault is "Paused":
```
 function processDeposit(
    GMXTypes.Store storage self
  ) external {
    GMXChecks.beforeProcessDepositChecks(self);

  ...
    try GMXProcessDeposit.processDeposit(self) {
```
 So the result is depositor looses his $1000, which benefit other depositors, whose vault token shares now worth more.

## Impact
User's funds loss
## Tools Used
Manual Review
## Recommendations
Maybe consider medium state before "Paused", which would pass the check for `processDeposit` and officially pausing it after proceessDeposit. 
__Note that:__
- This is a solution if you want to maintain the opportunity to pause the protocol in any state and you should carefully examine all other related to `deposit` functions, such as "onCancelation", etc...
- Here I provide just a basic example of how it could be achieved, but you should pay attention to the callbacks of `deposit` on the other states.
- If you don't want to add more complexity, you could just prohibit `pause` to be executed, when the vault is in state "Deposit" and wait until it is safe to execute the function. 

Example:

- `emergencyPause`:
```  
function emergencyPause(
    GMXTypes.Store storage self
  ) external {
   ...code
    if(self.status == GMXTypes.Status.Deposit){
    self.status = GMXTypes.Status.PrePaused;
    } else{
    self.status = GMXTypes.Status.Paused;
    }
    emit EmergencyPause();
  }
```

- `processDeposit`:
```
 function processDeposit(
    GMXTypes.Store storage self
  ) external {
    GMXChecks.beforeProcessDepositChecks(self); // Allow state "PrePaused"

    try GMXProcessDeposit.processDeposit(self) {
      ...code
      if(self.status == GMXTypes.Status.PrePaused){
      self.status = GMXTypes.Status.Paused
      }
      else{
      self.status = GMXTypes.Status.Open;
      }
      emit DepositCompleted(
        self.depositCache.user,
        self.depositCache.sharesToUser,
        self.depositCache.healthParams.equityBefore,
        self.depositCache.healthParams.equityAfter
      );
    } catch (bytes memory reason) {
      self.status = GMXTypes.Status.Deposit_Failed;

      emit DepositFailed(reason);
    }
  }
```



## <a id='M-02'></a>M-02. Invariant violation (funds could remain in the vault and a depositor could benefit from it)             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXCompound.sol#L58

## Summary
There is a violation to one of the main invariants defined by the sponsor - `After every action (deposit/withdraw/rebalance/compound), the vault should be cleared of any token balances. Violation of this could allow a subsequent depositor to benefit from it.` coming from the `compound()` function, which withdraws funds from `trove`, but after that there is a `if`, where the funds would remain in the contract, if the condition is not met.

## Vulnerability Details
`Compound` functionality is used to reinvest "rewards"(in form if airdrops for example) from GMX, given to the vault contract. The problem here is that if the the balance of the vault, for the given `tokenIn` after withdrawing `tokenA/B` from trove is zero, the function terminates with potentially tokens after the withdraw. Another problem is that the function does the check, wether the vault is open and set its state again inside the `if` statement. All those combined makes it possible for a user to benefit from those tokens.
```    // Transfer any tokenA/B from trove to vault
    if (self.tokenA.balanceOf(address(self.trove)) > 0) {
      self.tokenA.safeTransferFrom(
        address(self.trove),
        address(this),
        self.tokenA.balanceOf(address(self.trove))
      );
    }
    if (self.tokenB.balanceOf(address(self.trove)) > 0) {
      self.tokenB.safeTransferFrom(
        address(self.trove),
        address(this),
        self.tokenB.balanceOf(address(self.trove))
      );
    }

    uint256 _tokenInAmt = IERC20(cp.tokenIn).balanceOf(address(this));

    // Only compound if tokenIn amount is more than 0
    if (_tokenInAmt > 0) {
    ...
      _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
      _alp.tokenBAmt = self.tokenB.balanceOf(address(this));
    ...
      GMXChecks.beforeCompoundChecks(self);
      self.status = GMXTypes.Status.Compound;
      self.compoundCache.depositKey = GMXManager.addLiquidity(
        self,
        _alp
      );
    }
```

## Impact
Inside deposit/withdraw functions we send all tokens A/B back to `trove` and that would prevent the problem, but since `compound` doesn't check the state of the vault, this transaction could be frontruned, so the withdraw cache is created. After that the compound would left some funds for one of the tokens and then the transaction triggered from callback `proccessWithdraw` would use this tokens to benefit the depositor.
Imagine the following scenario:
- After some time for the protocol functioning now the trove has gained 1000 USDC and 0 WETH (from airdrop for example)
- The compound bot is running scheduled transactions once with "tokenIn : USDC", once with "tokenIn : WETH" 

__1.__ Bot initiates compound function with "tokenIn : WETH"

__2.__ Eve sees that transaction and frontruns it calling a `withdraw` with "shareAmt : 10, token : USDC"
   - This transaction set the vault status to "Withdraw"

__3.__ Bot enter `compound` and only withdraw 1000 USDC and 0 WETH. The transaction passes, no matter that the state is "Withdraw"

__4.__ Callback function to `proccessWithdraw` of Eve do check wether received amount is at least `minWithdrawTokenAmt`, which is not a problem, because the contract transfer the whole balance of USDC (1000 + {amount corresponding to 10 shares})

## Tools Used
Manual Review
## Recommendations

Consider adding 'else' statement, which would return withdrawn tokens
```
   if (self.tokenA.balanceOf(address(self.trove)) > 0) {
      self.tokenA.safeTransferFrom(
        address(self.trove),
        address(this),
        self.tokenA.balanceOf(address(self.trove))
      );
    }
    if (self.tokenB.balanceOf(address(self.trove)) > 0) {
      self.tokenB.safeTransferFrom(
        address(self.trove),
        address(this),
        self.tokenB.balanceOf(address(self.trove))
      );
    }

    uint256 _tokenInAmt = IERC20(cp.tokenIn).balanceOf(address(this));

    // Only compound if tokenIn amount is more than 0
    if (_tokenInAmt > 0) {
    ...
    }
    else{
     self.tokenA.safeTransfer(trove, self.tokenA.balanceOf(address(this));
     self.tokenB.safeTransfer(trove, self.tokenB.balanceOf(address(this));
    } 
```
## <a id='M-03'></a>M-03. It is possible to reopen a deposit after being closed            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L63C5-L63C5

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L72

## Summary
Close vault should be one way action, which should permanently stop the vault. However, due to lack of checks inside `emergencyPause` the vault could be reopened.
## Vulnerability Details
The vault could be reopened using following actions:
{The vault is being closed} => `emergencyClose()` (Will set vault's state from "Closed" to "Paused") => `emergencyResume()` (Will change state from "Paused" to "Open")
## Impact
Business logic error
## Tools Used
Manual Review
## Recommendations
On `emergencyPause()` check wether the whether the vault is closed:
```
function emergencyPause(
    GMXTypes.Store storage self
  ) external {
    if(self.status ==  GMXTypes.Status.Closed) revert TheVaultIsClosed(self);
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Consider erasing cache after completing deposit/withdraw/rebalance/compound operations            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L148

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L171-L181

## Summary
I would suggest to always erase data, which was for an action already executed. 
## Vulnerability Details
We use a cache to store the arguments for an action, because of the two transactions pattern used by GMX and so in the second transaction we reference the cache from the first. However, best practice is to erase an object once we have finished with it.
## Impact
As I could not find any path that could exploit this, I am rating it as low, but this could be a root cause with something else to abuse old data. And this could be prevented. 
## Tools Used
Manual Review
## Recommendations
After the end of each of the actions that are using cache, delete this cache, so it is impossible to exploit old data in some creative way.



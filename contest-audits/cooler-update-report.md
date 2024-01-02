# Cooler Sherlock Contest - Radev's Findings Report

# Findings Summary

| ID     | Title                                                                                                                                 | Severity |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| [H-01] | `repayLoan()` will fail if the lender is blacklisted and this results in a guaranteed griefing attack on the collateral of a borrower | High     |
| [H-02] | Malicious lender can Front-running `rolLoan()`                                                                                        | High     |
| [M-01] | When a loan can be liquidated, anyone can `claimDefaulted` on behalf of the lender, even if it is not authorized by the lender        | Medium   |
| [M-02] | Potential Stale Exchange Rate for DAI/gOHM in Clearinghouse                                                                           | Medium   |

---

# **Detailed Findings**

# [H-01] `repayLoan()` will fail if the lender is blacklisted and this results in a guaranteed griefing attack on the collateral of a borrower

## Summary

The `repayLoan()` function allows borrowers to repay a loan.

In doing so `repayLoan()` attempts to transfer the debt token back to the lender. However, if the loan token implements a blacklist like the common USDC token, the lender can join the USDC blacklist to prevent the borrower from repaying (the `debt().safeTransferFrom` transfer will be impossible) and thus withdraw the borrower's collateral.
So the borrower can be liquidated.

## Vulnerability Detail

#### Attack Scenario:

Consider we have:

- Borrower: Alice
- Lender: Bob

1. Alice (borrower) request the loan (via `requestLoan()` and transfer her collateral to the contract).
2. Bob (lender) accept the request loan (via `clearRequest()`) and set the `repayDirect_` parameter to true.
3. Bob calls approveTransfer() to approve transfer of loan ownership rights to a new blacklisted address of the token loan (USDC for example).
4. After that Bob immediately calls transferOwnership() to execute loan ownership transfer.
5. Alice wants to repay the loan, but since the new lender owner cannot receive USDC, the transaction fails.
6. After Alice (borrower) defaults, lender can withdraw lender's collateral via `claimDefaulted()` function.

## Impact

Any lender can prevent repayment of a loan and its liquidation. In particular, a lender can wait until a loan is almost completely repaid, transfer the loan to a blacklisted address (even one they do not control) to prevent the loan to be fully repaid / liquidated. The loan will default and borrower will not be able to withdraw their full amount of collateral.

This result in a guaranteed griefing attack on the collateral of a user.

I believe the impact is high since the griefing attack is always possible whenever the lent token implements a blacklist, and results in a guaranteed loss of collateral. This makes the borrower unable to repay the loan, resulting in liquidation.

## Code Snippet

Loan ownership transfering: https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L338-L354

```solidity
    function approveTransfer(address to_, uint256 loanID_) external {
        if (msg.sender != loans[loanID_].lender) revert OnlyApproved();

        // Update transfer approvals.
        approvals[loanID_] = to_;
    }

    /// @notice Execute loan ownership transfer. Must be previously approved by the lender.
    /// @param  loanID_ index of loan in loans[].
    function transferOwnership(uint256 loanID_) external {
        if (msg.sender != approvals[loanID_]) revert OnlyApproved();

        // Update the load lender.
        loans[loanID_].lender = msg.sender;
        // Clear transfer approvals.
        approvals[loanID_] = address(0);
    }
```

Repaying: https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L178

```solidity
    function repayLoan(
        uint256 loanID_,
        uint256 repaid_
    ) external returns (uint256) {
        ...

        // Transfer repaid debt back to the lender and (de)collateral back to the owner.
        debt().safeTransferFrom(msg.sender, repayTo, repaid_);

        ...
    }
```

Liquidation: https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L318-L333

```solidity
    function claimDefaulted(
        uint256 loanID_
    ) external returns (uint256, uint256, uint256) {
        Loan memory loan = loans[loanID_];
        delete loans[loanID_];

        if (block.timestamp <= loan.expiry) revert NoDefault();

        // Transfer defaulted collateral to the lender.
        collateral().safeTransfer(loan.lender, loan.collateral);

        // Log the event.
        factory().newEvent(loanID_, CoolerFactory.Events.DefaultLoan, 0);

        // If necessary, trigger lender callback.
        if (loan.callback)
            CoolerCallback(loan.lender).onDefault(
                loanID_,
                loan.amount,
                loan.collateral
            );
        return (loan.amount, loan.collateral, block.timestamp - loan.expiry);
    }
```

## Tool used

Manual Review

## Recommendation

Consider modifying the logic so that when a loan is repaid or liquidated, the lendingToken is sent to the contract first, allowing the lender to claim it at a later time. This would remove the need for the repayDirect\_ option.

# [H-02] Malicious lender can Front-running `rolLoan()`

## Summary

The protocol offers a mechanism to rollover a loan.

- Initially, the lender sets the new terms for the loan using the `provideNewTermsForRoll()` function. These terms define aspects like the interest rate and duration that the borrower will adhere to.
- Once the lender has established these terms, the borrower can proceed to call the `rollLoan()` function. This action lets borrowers refresh their loans with the new conditions, effectively extending their loan with the updated duration and interest rate. It's essential to note that the lender must determine these new terms in advance.

A vulnerability has been identified in the loan rollover process, allowing a malicious lender to front-run the borrower's transaction. This can result in the borrower unintentionally accepting unfavorable loan terms they never agreed to.

## Vulnerability Detail

The identified vulnerability revolves around the process of loan rollover. Let's say the process begins when a borrower, close to the expiration of their loan, negotiates with the lender to provide new terms for the loan via the provideNewTermsForRoll() function. However, after the lender sets these terms, there's an opportunity for the lender to front-run the borrower's acceptance (via `rollLoan()`) of the terms. The lender can send a transaction with a higher gas price to modify these terms, requiring more collateral, higher interest, and a shorter duration or even worse with zero duration. Consequently, when the borrower tries to accept the terms using the `rollLoan()` function, they inadvertently accept these malicious terms set by the front-running transaction of the lender.

Potential exploit scenario is:

1. A borrower's loan is nearing its expiration.
2. The borrower asks the lender to set new terms for the loan via the `provideNewTermsForRoll()` function. Both parties `agree on specific terms` for the new duration, collateral, and interest.
3. The lender calls provideNewTermsForRoll() with the agreed-upon terms.
4. When the borrower attempts to call the rollLoan() function to accept the new terms, however a malicious lender can `front-run` this transaction by setting different terms (more collateral, higher interest, and shorter duration) by again calling the `provideNewTermsForRoll()` function with a higher gas fee.
5. The borrower's rollLoan() function then gets executed with these unfavorable terms, putting the borrower at a significant disadvantage.

Even worse scenario is if the lender call `provideNewTermsForRoll()` function with very high collateral and zero duration, so in `rollLoan()` loan's expiry `remains unchanged` due to the addition of a zero (`loan.expiry += loan.request.duration`).

## Impact

The borrower can end up locked into unfavorable terms that they never agreed upon. They could find themselves providing more collateral than initially discussed, paying higher interest rates, and having less time or even no time to repay. Such an event can have severe financial implications for the borrower.

## Code Snippet

Procedure for Providing New Terms for a Loan Rollover:
https://github.com/sherlock-audit/2023-08-cooler-radeveth/blob/main/Cooler/src/Cooler.sol#L282-L300

```solidity
    function provideNewTermsForRoll(
        uint256 loanID_,
        uint256 interest_,
        uint256 loanToCollateral_,
        uint256 duration_
    ) external {
        Loan storage loan = loans[loanID_];

        if (msg.sender != loan.lender) revert OnlyApproved();

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

Rolling a loan over with new terms:
https://github.com/sherlock-audit/2023-08-cooler-radeveth/blob/main/Cooler/src/Cooler.sol#L192-L217

```solidity
    function rollLoan(uint256 loanID_) external {
        Loan memory loan = loans[loanID_];

        if (block.timestamp > loan.expiry) revert Default();
        if (!loan.request.active) revert NotRollable();

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

        // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onRoll(loanID_, newDebt, newCollateral);
    }
```

## Tool used

Manual Review

## Recommendation

To prevent this front-running vulnerability, it's recommended to introduce an additional confirmation step or a locking mechanism:

1. Locking Mechanism: Once the lender has provided terms using provideNewTermsForRoll(), restrict any new term changes for a predetermined time period. This buffer ensures the borrower has a safe window to accept terms without interference.

2. Borrower Confirmation: Implement a confirmation mechanism where the borrower must first validate the provided terms before they can be used. Introduce an intermediary function that the borrower must call to confirm their acceptance of the terms. Only post this confirmation can the loan then be rolled over.

By implementing the above recommendations, it would ensure that once terms are decided upon, they cannot be maliciously altered without the explicit confirmation of the borrower, safeguarding against the outlined front-running attack.

# [M-01] When a loan can be liquidated, anyone can `claimDefaulted` on behalf of the lender, even if it is not authorized by the lender

## Summary

When the loan is liquidatable, it can be liquidated by the lender or anyone approved by the lender. However, after the loan becomes liquidatable, anyone can initiate a `claimDefaulted()` on behalf of the lender to liquidate the loan, even if it was not intended by the lender. This means the lender will be forced to call `claimDefaulted()` to `withdraw collateral even if he/she intended to provide new terms for the loan to be rolled over`.

## Vulnerability Detail

`Cooler#claimDefaulted()` would pass if the loan is defaulted (the borrower does not repay in time). In that case, anyone can trigger a claiming of collateral on behalf of the lender, even if the lender would provide new terms for loan to be rolled over.

When `block.timestamp > loan.expiry` is true, this loan will be liquidatable.

```solidity
    function claimDefaulted(
        uint256 loanID_
    ) external returns (uint256, uint256, uint256) {
        Loan memory loan = loans[loanID_];
        delete loans[loanID_];

        if (block.timestamp <= loan.expiry) revert NoDefault();

        // Transfer defaulted collateral to the lender.
        collateral().safeTransfer(loan.lender, loan.collateral);

        ...
    }
```

## Impact

The lack of access control disrupts the normal liquidation flow, which is clearly a high impact.

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler-radeveth/blob/main/Cooler/src/Cooler.sol#L318-L333

## Tool used

Manual Review

## Recommendation

To ensure only the lender can claim the collateral of defaulted loan, you should add a check for the sender's address against the lender's address.

```solidity
    function claimDefaulted(uint256 loanID_) external returns (uint256, uint256, uint256) {
        Loan memory loan = loans[loanID_];
        delete loans[loanID_];

+      if (msg.sender != loan.lender) revert OnlyApproved();
        if (block.timestamp <= loan.expiry) revert NoDefault();

        // Transfer defaulted collateral to the lender.
        collateral().safeTransfer(loan.lender, loan.collateral);

        // Log the event.
        factory().newEvent(loanID_, CoolerFactory.Events.DefaultLoan, 0);

        // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onDefault(loanID_, loan.amount, loan.collateral);
        return (loan.amount, loan.collateral, block.timestamp - loan.expiry);
    }
```

# [M-02] Potential Stale Exchange Rate for DAI/gOHM in Clearinghouse

## Summary

The `Clearinghouse` contract uses a fixed DAI/gOHM exchange rate through the LOAN_TO_COLLATERAL constant. Given the volatile nature of cryptocurrency prices, relying on a fixed exchange rate is problematic. As currencies fluctuate, this constant value will inevitably become obsolete, potentially leading to under-collateralized loans.

## Vulnerability Detail

The `ClearingHouse` contract design permits loan processing as long as the operator approves it. Presumably, this operator could be an automated keeper program. Loan fairness is determined using the `LOAN_TO_COLLATERAL` constant, which represents the DAI/gOHM exchange rate. As of now, the value of gOHM is $2,500, indicating that `LOAN_TO_COLLATERAL` fixed value is already outdated.

This vulnerability's gravity can be likened to a scenario where a Chainlink oracle is utilized, but no mechanism checks if the oracle's provided exchange rate is stale. Such a lapse in real-time rate-checking is considered of medium severity.

While it's uncertain who controls the operator address invoking the clear() method, it's likely designed to be an automated keeper. Such a keeper, without any modification, might approve loans without verification. Even if the operator were a human, the existence of hardcoded checks like `LOAN_TO_COLLATERAL` suggests that the contract aims to safeguard against potential errors, so this particular vulnerability should be addressed.

## Impact

There's a potential risk of approving under-collateralized loans. Astute borrowers could exploit this, taking loans they intend to default on. They'd benefit by using the borrowed amount to acquire more collateral than what they'd forfeit upon defaulting.

## Code Snippet

The loan-to-collateral is hard-coded, rather than being based on an oracle price.
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L55

```solidity
    uint256 public constant LOAN_TO_COLLATERAL = 3000e18;       // 3,000 DAI/gOHM
```

If the gOHM price drops below $3000 to say $2500, a loan for 3000 DAI will be under-collateralized.

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L371-L373

```solidity
    function collateralFor(uint256 amount_, uint256 loanToCollateral_) public view returns (uint256) {
        return (amount_ * (10 ** collateral().decimals())) / loanToCollateral_;
    }
```

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L386-L391

```solidity
    function debtForCollateral(uint256 collateral_) public pure returns (uint256) {
        uint256 interestPercent = (INTEREST_RATE * DURATION) / 365 days;
        uint256 loan = collateral_ * LOAN_TO_COLLATERAL / 1e18;
        uint256 interest = loan * interestPercent / 1e18;
        return loan + interest;
    }
```

## Tool used

Manual Review

## Recommendation

To ensure loans remain adequately collateralized, implement a dynamic pricing model using Chainlink oracles. This will continuously fetch the most recent DAI/gOHM exchange rate, reducing the potential for under-collateralized loans.
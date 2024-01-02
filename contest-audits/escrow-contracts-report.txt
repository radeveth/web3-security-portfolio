# Escrow Contracts Codehawks Contest - Radev's Findings Report

# Findings Summary

| ID     | Title                                                                   | Severity |
| ------ | ----------------------------------------------------------------------- | -------- |
| [M-01] | The buyer of the Escrow contract can obtain all tokens contract balance | Medium   |
| [M-02] | Contracts do not work with fee-on-transfer tokens                       | Medium   |

---

# **Detailed Findings**

# [M-01] The buyer of the Escrow contract can obtain all tokens contract balance

## Severity

Medium Risk

## Relevant GitHub Links

https://github.com/Cyfrin/2023-07-escrow/blob/main/src/Escrow.sol#L109-L129

## Summary

The buyer of the Escrow contract can obtain/get all tokens contract balance in the Escrow#resolveDispute() function. When the new Escrow is created in the EscrowFactory contract the msg.sneder is actually the buyer of the Escrow contract. This allows the msg.sender (buyer) to set the arbiter address to their own.

## Vulnerability Details

Let's consider the following example:

1. The buyer creates a new Escrow contract and sets the arbiter address to their own.
2. Buyer call Escrow#initiateDispute() function to set the state to Disputed.
3. The arbiter (who is actually the buyer) calls the Escrow#resolveDispute() function and passes the buyerAward parameter with the value of i_tokenContract.balanceOf(address(this)).
4. The buyer obtains all of the tokens in the contract balance.

## Impact

Medium

## Tools Used

Manual Code Review

## Recommendations

One possible solution is to ensure that msg.sender != arbiter during the creation of a new Escrow contract.

```solidity
+       if (arbiter == msg.sender) {
+           revert Improper_Action();
+       }

        Escrow escrow = new Escrow{salt: salt}(
            price,
            tokenContract,
            msg.sender,
            seller,
            arbiter,
            arbiterFee
        );
```

# [M-02] Contracts do not work with fee-on-transfer tokens

## Severity

Medium

## Relevant GitHub Links

- https://github.com/Cyfrin/2023-07-escrow/blob/main/src/Escrow.sol#L119-L128
- https://github.com/Cyfrin/2023-07-escrow/blob/main/src/Escrow.sol#L98
- https://github.com/Cyfrin/2023-07-escrow/blob/main/src/EscrowFactory.sol#L39

## Summary

Some tokens take a transfer fee (e.g. STA, PAXG). According to my knowledge, the Escrow contract will use the WETH token. However in the past, some tokens like USDT and USDC did not charge a fee, but after a while, they started to take a fee on transfers.

## Impact

- Potential Loss of Funds: In the worst-case scenario, tokens may become trapped in a contract if it is not equipped to handle the fee deduction. If the contract expects a certain amount and receives less, it may not function correctly, leading to a loss of funds.
- User Experience: From a user's perspective, these fees can create confusion and unexpected costs, particularly if they are not clearly disclosed or if the user is unaware that the token includes transfer fees.

## Tools Used

Manual Code Review

## Recommendations

Use the balance before and after the transfer to calculate the received amount instead of assuming that it would be equal to the amount passed as a parameter.
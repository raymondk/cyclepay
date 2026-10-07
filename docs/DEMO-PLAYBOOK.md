# Demo playbook

A speaker's outline for a recorded walkthrough of the payment flow. Setup is already
done on whatever deployment you show, so it is narrated, not performed.

## Before recording

- ⚠️ **No number is written in this file.** Read them off the screen:
  ```bash
  icp canister call backend pricing_status '()' -e ic   # divisor: 1 is live, >1 simulation
  icp canister call backend reserve_status '()' -e ic   # availableToSell, totalOrders
  ```
- ⚠️ `unset STRIPE_API_KEY STRIPE_WEBHOOK_SECRET`, and keep `scripts/.local-dev.env` off
  screen. The canister cannot leak them; your terminal can.
- ⚠️ **One open order per principal.** Cancel every rehearsal order, or the next one is
  refused until Stripe expires the session (~35 min).

## The pitch

- Let a customer **buy cycles with a credit card**, from a card to spendable cycles,
  without solving crypto onboarding first. No wallet, no exchange account, no ICP.
- Runs entirely on ICP: one Motoko backend canister, one certified-assets frontend, no
  server, both on a confidential SEV-SNP subnet. No external dependency but Stripe;
  pricing reads two canisters, not an HTTP oracle.

## Roles

- **Operator** funds the cycles reserve (the canister's own cycles-ledger account, not
  its gas), provisions the two Stripe secrets sealed with vetKeys, and allow-lists
  customers while the gateway takes sandbox payments.
- **Customer** creates an order, pays on Stripe's hosted page, and receives cycles on
  the cycles ledger at the principal the signed-in page shows.
- **Backend canister** quotes and locks the cycle quantity, reserves it so a paid order
  can always be delivered, verifies the webhook, delivers with one `icrc1_transfer`, and
  writes an audit line for every Stripe event it acts on.

## The demo

Open the page, show the balance and the completed-order count, then narrate one live
purchase:

1. **Pick the amount.** The quote shows the cycles and the processing fee before
   committing. USD/ICP comes from the Exchange Rate Canister and XDR/ICP from the CMC.
2. **Create the order.** Locks the quote, reserves the cycles, and makes an HTTPS
   outcall to Stripe for a Checkout Session. Stripe's deadline is ~35 minutes.
3. **Pay.** Card details go to Stripe's page and never touch this system.
4. **Stripe calls the webhook**: `http_request_update`, an update call, so it goes
   through consensus.
5. **The canister verifies the HMAC** against the signing secret, then transfers the
   cycles to the customer's principal.

Three points worth making out loud:

- **Step 5 moves cycles, never fiat.** The money stays at Stripe.
- **The HMAC is the entire trust root.** The endpoint is public and the caller is
  anonymous, so whoever holds the signing secret gets cycles for free. That is why the
  reserve balance is the blast radius and why the secret is sealed rather than pasted.
- **ICP cancels out of the price.** Both rates are per ICP, so a cycle is XDR-pegged and
  the gateway holds no ICP. Cross-checkable against the IMF's published SDR rate.

## Considerations

- **Running out of cycles.** The gate refuses the order before the customer pays, and a
  promise is held from creation, so a paid order is always deliverable.
- **An expired session.** Stripe tells the canister, the order is marked expired, and
  the reserved cycles go back to what is sellable.
- **A dispute.** One audit line and no automated response, deliberately: the cycles are
  delivered and irreversible while the card network pulls the fiat back. A refund is
  different: it files an obligation on the order.
- **This is a demo.** You must be allow-listed, because sandbox payments are free and an
  open list on a funded gateway would be a faucet. Cycles are divided by the divisor,
  quote and delivery alike.

## Try it

Send me the principal the signed-in page shows you and I will allow-list it. Internet
Identity derives one per origin, so a principal copied from another app is a different
account. The rest is in the repo: [`docs/ARCHITECTURE.md`](./ARCHITECTURE.md) for the
diagram, [`docs/VERIFY.md`](./VERIFY.md) for what anyone can check.

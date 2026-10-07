# Operations runbook

Day-2 operations for the cycles gateway: provisioning, money levers, triage, incident
response. Build and release procedure is `RELEASE.md`; first-time setup for all three
modes is `docs/OPERATE.md`.

**`§N` always means a `docs/DESIGN.md` section**, the same thing the `§N` comments in
the code point at. This file's own sections are referred to by name.

## Enter here: what you are looking at

| what you are seeing | go to |
|---|---|
| the rail refuses every purchase, `refusingNow.railClosed` | **3. Presets, the API key**: the rail opens when both secrets are provisioned |
| `refusingNow.reserveShort`, or `availableToSell` is 0 while the ledger holds cycles | **5. The cycles reserve**: a floor that only rises by observation |
| `refusingNow.stripeApiFailing` latched | **8. Monitoring** P1 rows: one lever, rotate the key |
| an order sits at `created` past its own `expiresAtNs` | **5b. Order expiry**: the missed-event case, and why nothing sweeps it |
| buyers report that cancelling does nothing | **8. Monitoring** P2: run `expire_order` once, read `order.expireRaced` |
| a buyer paid and was never credited | **6. Obligations**, the `#paidNotCredited` row. Resend first, always |
| `pricing_status.lastAttempt.ok` is false, or a quote returns no cycles | **4. Pricing rates** |
| a delivery is stuck, or you must establish where the money is | **6. Obligations** and **7. Recovery timer** |
| you suspect the webhook secret leaked | **2. Webhook secret** |
| an unexpected principal can or cannot buy | **5a. Admission gate** |
| you are about to upgrade the canister | **10. Upgrades & releases**, then `RELEASE.md` |
| you are setting a deployment up for the first time | not here: `docs/OPERATE.md` |

## 0. Operating model

**The admin allowlist IS the canister controller set** (§7): every admin method runs
`requireAdmin`, which accepts exactly `caller ∈ controllers` (anonymous always rejected)
and traps otherwise. All controllers are equal: any one can upgrade, withdraw, rotate the
secret, resolve errors, change every config. The hardening path is a multisig canister as
sole controller (IC controllers are OR-semantics) or SNS (§11).

```bash
icp canister settings update backend --add-controller <principal> -e ic
icp canister status backend -e ic        # lists current controllers
```

**Calling convention** for everything below (admin calls must use a controller identity):

```bash
icp canister call backend <method> '(<candid args>)' -e ic --identity <operator>
```

⚠️ **Always pass an explicit `'()'` for zero-argument methods.** Omitting the argument
makes `icp canister call` ask *"Do you want to send this message?"* and read stdin, which
hangs any script or CI step.

**For reads, open the console first.** Every read-only command in this runbook is also a
panel at `#/admin`; the `Admin` header link appears for a controller or a granted admin.

| panel | what it shows |
|---|---|
| **Now** (default) | the summary split into what needs a person and what clears itself, the reserve, and refusals since deploy |
| **Worklists** | orphans · problems · delayed · pending, as sortable tables |
| **Orders** | order history, paged · look up one order by id |
| **Diagnostics** | `health` · queue depths · the recovery sweep with its count drift · the audit trail |
| **Configuration** | the config groups, the write commands, and this browser's identity |

The CLI is still the answer for anything that **writes** (the console prints the command,
it does not send it), reading when the frontend is down or not yet deployed, and scripted
or monitored reads. Opening the console spends no audit entries: `admin_order`,
`admin_receipt` and `delivery_journal` are updates so the read is recorded, and no panel
calls them on open.

Public queries (`reserve_status`, `pricing_status`, `recovery_status`, `card_tiers`,
`lifecycle_config`, `can_purchase`, `cycles_status`, `orphan_depth`, `health`) work from
any identity and are the monitoring surface (§8).

**Units:** money is US cents (`usdCents`); durations are nanoseconds (1 h =
`3_600_000_000_000`, 24 h = `86_400_000_000_000`, 72 h = `259_200_000_000_000`); cycle
prices are XDR-pegged (1 XDR = 1 T cycles).

## 2. Webhook secret — provisioning & rotation (§7)

HMAC is symmetric, so verify = forge: anyone holding this secret can forge "paid"
webhooks and drain the entire reserve, one order at a time. **The reserve balance is the
blast-radius bound, so size it to what you can afford to lose in one window.** It is
stored plaintext by design (`Secret.mo`; the confidential-subnet checklist is section 9).

**Provision / rotate. The value is sealed, never typed into a call:**

```bash
# Reads STRIPE_WEBHOOK_SECRET from the environment (or scripts/.local-dev.env, locally).
STRIPE_WEBHOOK_SECRET='whsec_…' scripts/seal-secret.sh webhook-secret ic
```

- ⚠️ **`set_webhook_secret` takes a `blob`, not the string** (§7.3): an IBE ciphertext
  sealed to this canister's vetKD public key. Calling it by hand with a quoted `whsec_…`
  returns `#notCiphertext`.
- ⚠️ **The trailing `ic` selects the MAINNET master key, and it is not optional.** Mainnet
  and a local network both have a vetKD key called `key_1` backed by different master
  keys. Omit it and the script defaults to `local`, producing a ciphertext this canister
  can never open (`#notSealedToThisCanister`).
- Pass the full `whsec_…` string, prefix included; the whole string is the HMAC key.
- The 16-byte floor applies to the decrypted value (`#tooShort`), and the working secret
  is left untouched on any rejection.
- `webhook_secret_status` returns `{isSet; generation; setAtNs}`. `generation` increments
  per successful set, which is how you confirm a rotation landed; there is no read-back
  path, not even for controllers.
- ⚠️ **Put the values in `scripts/.local-dev.env` or export them, never on the command
  line.** A secret typed as an argument lands in shell history, `ps` output and CI logs.
  The examples show the variable inline for brevity.

**Rotation** (Stripe-side overlap makes it zero-downtime):

1. In the Stripe Dashboard, roll the endpoint's secret with an overlap window. During
   overlap Stripe signs each delivery with one `v1=` per active secret, and the verifier
   accepts any matching `v1`.
2. `scripts/seal-secret.sh webhook-secret ic` with the new `whsec_…`; confirm
   `generation` bumped.
3. Expire the old secret in Stripe after confirming deliveries succeed.

**Suspected leak**, in this order:

1. **Know what you can and cannot do.** A forged webhook drains at most what the reserve
   holds, and there is no cap or pause lever behind it. ⚠️ **`withdraw_reserve` will
   REFUSE during an incident, by design**: it is guarded on there being no promise-holder
   at all, and a forged drain means forged orders are open. What you can do immediately
   is stop new orders: the rail is live only while both Stripe secrets are provisioned,
   so rotating the webhook secret (step 2) closes it until the new one is set. Forged
   orders already in flight will deliver.

   The evacuation path is three steps, and step 2 cannot be forced: rotate the webhook
   secret; clear every promise-holder until `reserve_status.promiseHolders` reads zero;
   `withdraw_reserve`. ⚠️ **Clearing is a wait, not a lever, for anything with a delivery
   in flight.** `abandon_order` refuses a paid order whose delivery is outstanding, so a
   forged order already being delivered has to settle, or reach `needsReview` at the
   ~24 h dedup window. `process_order` re-drives a stalled delivery; `pending_deliveries`
   is the live view. There is deliberately no lever that releases a promise over an
   unresolved delivery. The standing control is therefore a sizing decision made before
   an incident: `reserve_status.availableToSell` is the figure.
2. Roll the secret in Stripe + `set_webhook_secret` (above).
3. Reconcile: compare `audit_log` / the order store against the Stripe Dashboard's event
   log; forged "payments" have no matching Stripe payment_intent. Refund nothing that
   has no real charge.
4. Refund the reserve to its sized level with `icp cycles transfer`, then
   `refresh_reserve`. There is no "resume held orders" step: an order refused for
   `#reserveShort` was never created, so legitimate buyers retry and succeed.

⚠️ **The order of steps 2–4 is itself the control.** Refunding the reserve before the
secret is rolled hands the attacker a freshly funded account. Rotate, then reconcile,
then refund.

## 3. Presets, the API key, and the settings that must stay off (§3, §6.1)

The canister creates a **Checkout Session per order** through the Stripe API, with inline
`price_data`. There are no Products, Prices, Payment Links or Dashboard objects to create.

### Provisioning, in order

```bash
# 1. The API key. RESTRICTED (rk_). Permission: Checkout Sessions = WRITE, everything
#    else None. Write also grants the read the recovery sweep needs: it retrieves a
#    session to settle an order whose expiry event never arrived. A key without it
#    401s on every retrieve and stranded capacity is never released (monitoring:
#    refusingNow.stripeApiFailing). SEALED (§7.3); the trailing `ic` selects the
#    MAINNET master key.
STRIPE_API_KEY='rk_...' scripts/seal-secret.sh api-key ic

# 2. Where Stripe returns the buyer. Validated: https, no query, no fragment.
#    Not a secret, so it is set directly.
icp canister call backend set_stripe_origin '("https://<your-origin>")' -e ic --identity <operator>

# 3. The webhook signing secret (section 2). Sealed the same way.
STRIPE_WEBHOOK_SECRET='whsec_...' scripts/seal-secret.sh webhook-secret ic

# 4. The price tiles. ⚠️ REQUIRED for a usable page: with an empty list the buy view
#    renders no tiles AND no custom field (`docs/OPERATE.md`, Mode 3, step 4). Do not
#    register a $100 preset; that is the ceiling and the custom field's job.
icp canister call backend set_card_tiers \
  '(vec { record { id = "t10"; usdCents = 1_000 : nat } })' \
  -e ic --identity <operator>
```

- ⚠️ **Use a restricted key, not an `sk_`.** A leaked write-sessions key can create
  sessions that pay *you*; one that can issue refunds is a materially worse thing to
  leak. Stripe's IP and ASN allowlists are unusable here: a subnet's replicas have many
  changing addresses.
- **Neither secret can be read back out, even by a controller.** `stripe_api_key_status`
  and `webhook_secret_status` report a generation counter and a set timestamp;
  `seal-secret.sh` prints the relevant status after each successful set.
- **Provisioning the two secrets is what OPENS the rail**, so do them last. Rotating
  either closes it until both are valid again, which is a deliberate ordering property:
  no API key means no payable session, no webhook secret means a buyer can pay and
  cannot be credited.
- ⚠️ **Changing the origin later is a user-visible migration, not a config tweak**:
  Internet Identity derives a principal per origin.
- **Record the Stripe API version the account is on, and treat changing it as a code
  change.** Webhook payload shapes follow the account default, so an account-level
  upgrade silently changes what `Json.mo` parses, which no test here can see.

### Tier registration

Validation is atomic: non-empty unique ids, non-zero amounts, every amount within
`[minPurchaseUsdCents, maxPurchaseUsdCents]`, or the whole call rejects and the live
tier list is untouched. `card_tiers` is the public query the frontend renders. A tier's
cycle quantity is locked per order at creation from the cached rate pair, so changing
tier prices never reprices existing orders.

### The settings that must stay off

The whole model rests on one invariant: **the session's `amount_total` equals the
order's `usdCents`.** The canister reads `data.object.amount_total`, which Stripe defines
as the total after discounts and taxes, and refuses anything else. Per-order sessions
removed most of the ways that can break: the canister sends `price_data` inline,
`payment_method_types[]=card`, no promo codes, no adjustable quantity, and
`adaptive_pricing[enabled]=false`. The authoritative list is in the code next to
`Session.createBody`. Two things remain account-level and are yours to keep off:

| Setting | Must be | If enabled |
|---|---|---|
| **Automatic tax** (account default) | **off** | raises `amount_total`; the payment is refused as a mismatch, so nothing is delivered, but every order fails until it is turned off |
| **Adaptive pricing** (Dashboard toggle) | pinned off by the request | currently harmless to `amount_total` for this shape; the request pins it anyway |

Plus **USD** (any other currency is refused as `#unattributed`) and **card-only**.

**A mismatch is not silent.** The canister delivers nothing and files an `#unattributed`
whose detail names both figures, so an amount-moving setting shows up as a queue entry on
the first order, not as a slow drift in what buyers receive.

⚠️ **Test-mode and live-mode keys are different objects.** Going live means a live-mode
restricted key and a live-mode webhook secret, both re-provisioned. Once
`set_expected_livemode '(opt true)'` is set, a stray test-mode event is refused and
tagged `stripe.livemodeMismatch`, so getting it wrong means nobody can buy, not lost
money.

### What the app does with no presets

**An empty preset list is NOT the rail's off switch.** A buyer can order any amount
between the floor and the ceiling without a preset; `create_order` still answers
`#unknownTier` for a `#tier` id that is not registered. **The switch is both Stripe
secrets being provisioned**, derived from capability rather than declared. `railsLive`
also gates the rate-refresh timer, so a gateway with no API key does not pay for XRC
calls it cannot use. To take the rail down deliberately, there is no lever short of
rotating a secret to a value Stripe rejects; `can_purchase` and the `#sessionUnavailable`
refusal are what an operator has instead.

## 4. Pricing rates (§3.1)

Two rates, both read from on-chain canisters on the same timer tick: the **XRC**
(`uf6dk-hyaaa-aaaaq-qaaaq-cai`) for USD/ICP and the **CMC** for XDR/ICP. There is no
HTTPS outcall and no settable rate source.

```bash
icp canister call backend pricing_status '()' -e ic   # public: both rates, config, last refresh
icp canister call backend quote_previews '(vec { 500 : nat })' -e ic  # public: what an amount buys
icp canister call backend refresh_rates '()' -e ic --identity <operator>   # force a tick now
icp canister call backend set_pricing_config \
  '(record { feeBps = 290 : nat; feeFixedCents = 30 : nat; maxAgeNs = 300_000_000_000 : nat; maxRateDeltaBps = 5_000 : nat; minRateSources = 2 : nat })' \
  -e ic --identity <operator>
```

Defaults: 290 bps + 30¢ (Stripe's fee, recovered net-of-fees per §3), a 5-min staleness
window, a 50% delta bound, and a 2-source minimum. Setting the config re-arms the refresh
timer immediately.

`quote_previews` is the fastest "is the rail actually quoting?" check: it runs the same
pricing code `create_order` runs, so a `cycles = null` there is exactly what a buyer
would hit. `pricing_status` is the one command to run first for *why*: `rates` carries
both values plus `fetchedAtNs` and the XRC `quality`; `lastAttempt` carries
`{atNs; ok; detail}`, and **`detail` names the rejecting guard**.

### `maxAgeNs` is a security control, not a tuning knob

Validation caps it at 1 h (`#maxAgeTooLong`). Timers are deactivated by any Wasm change,
and the only thing that makes a dead timer safe is that a stale cache refuses to price.
Widen it to ride out an outage only with that trade understood.

### Diagnosing a stale rate

`create_order` never refreshes; it reads the cache and fails closed. So persistent
`rateUnavailable` is always one of:

| `lastAttempt.detail` | Meaning | Action |
|---|---|---|
| `NotEnoughCycles` | fewer than 1 B cycles could be attached | check `cycles_status`; top up. `Gate.minCanisterCycles` must stay well above 1 B or pricing stops before the gate does |
| `RateLimited` | XRC throttling | wait; backoff already widens the interval |
| `Pending` | XRC is still collecting | resolves on its own; alert only if it persists across ticks |
| `InconsistentRatesReceived` | XRC's sources disagree beyond its own tolerance | wait it out. Never work around it |
| `CryptoBaseAssetNotFound` / `StablecoinRate*` | XRC cannot price ICP/USD right now | wait; nothing local to fix |
| `too few sources` | fewer than `minRateSources` answered | a thin market. The 2-source minimum exists because the XRC's own `InconsistentRatesReceived` cannot fire for a single source; do not lower it to 1 |
| `implausible rate` | outside $0.10–$10,000/ICP | a bad upstream print. Rejected as if down |
| `delta` | moved more than `maxRateDeltaBps` since the last good value | a genuine 50%+ move needs `maxRateDeltaBps` raised once, deliberately; otherwise it is source disagreement |
| `implied XDR/USD` | `P × 10⁸ / U` fell outside 0.5–1.2 | the two sources disagree about reality. XDR/USD has sat in ~0.6–0.9 for decades |
| `cmc stale` | CMC rate older than 15 min | check the CMC; nothing local to fix |

A rejected refresh keeps the previous rate serving until it goes stale, so a single bad
tick is invisible to buyers. The plausibility band and the implied cross-check are not
configurable.

**A rate outage never strands a paid order.** Money-out reads no rate at all; it
transfers a figure fixed when the order was created. An outage means no new orders,
never a stuck buyer.

## 5. The cycles reserve

### Funding the reserve

⚠️ **On a gateway that accepts Stripe TEST payments, populate the buyer allow-list BEFORE
you fund the reserve.** Test payments are free and unlimited, so test mode plus an empty
allow-list plus a funded reserve is a cycles faucet, and the gateway refuses every buyer
in that state (`refusal_counts.refusingNow.unboundedGiveaway`).

```bash
# Who may buy while test payments are accepted. Controller only.
icp canister call backend add_allowed_buyer '(principal "<tester>")'
icp canister call backend allowed_buyers '()'
```

An unfunded reserve refuses every order at `Gate.solvent` anyway, before a Stripe session
is created, so a sandbox deployment is safe to explore before either exists.

⚠️ **`icp` defaults to the LOCAL environment**, so every command in this section needs
`-n ic` (the cycles ledger) or `-e ic` (a canister) to act on mainnet. Omitted, they
succeed against a local network and report a balance that has nothing to do with the one
being funded.

```bash
# The reserve IS the gateway's own cycles-ledger account.
icp cycles transfer 100t <backend-principal>          # -n ic on mainnet

# ⚠️ REQUIRED after every top-up. Without it the balance is real and unsellable.
icp canister call backend refresh_reserve '()'

# What the gateway will actually sell: floor - promised = availableToSell.
icp canister call backend reserve_status '()'

# Read the truth from the ledger — anyone can, including the frontend.
icp canister call um5iw-rqaaa-aaaaq-qaaba-cai icrc1_balance_of \
  '(record { owner = principal "<backend-principal>"; subaccount = null })'
```

⚠️ **`reserveFloor` is a maintained lower bound, not the balance** (§5.4). It rises on
`refresh_reserve` and on the hourly sweep; it falls when the gateway itself transfers
out. **The ledger reading 100 T while `availableToSell` reads 0 is the expected
appearance of a top-up nobody observed**, and `reserveObservedAtNs` is how you tell that
from a genuinely spent reserve. An observation is adopted only across a quiet window (no
delivery in flight), so a reconcile during a busy sweep is skipped, audited as
`reserve.reconcileSkipped`, and retried: a delay, never a loss.

Delivery is one `icrc1_transfer` out of that account, and the buyer receives
`lockedCycles − fee`, where the fee is the stored one (the ledger reports its own fee on
`#BadFee`, so the copy self-corrects). Nothing writes that stored fee but the ledger
itself; there is deliberately no admin lever for it. If the ledger's fee ever exceeds an
order's locked quantity, delivery stalls loudly on `delivery.feeExceedsOrder` and the
answer is a redeploy. The ledger charges its fee on top of the amount, so a delivery
moves the reserve by exactly `lockedCycles`, which is why the promise tally has no
separate fee term.

⚠️ **Nothing creates cycles here.** Refills are `icp cycles transfer` from outside.

**`withdraw_reserve`** exists because a funded mainnet reserve is real money in a ledger
account. It is controller-only and refused while any promise-holder exists, so nothing
can be owed to a buyer when it runs. A decommissioning lever, not an incident one (the
three-step evacuation in section 2).

⚠️ **THREE balances, and confusing them is the most common local-setup failure.**

| balance | what it is | read it with | fund it with |
|---|---|---|---|
| **gas** | what the canister spends to *run*, gated by `minCanisterCycles` | `icp canister status backend` | `icp canister top-up` |
| **the reserve** (stock) | what the canister *sells*: its cycles-ledger account | `reserve_status`, or `icp cycles balance --of-principal <backend-id>` | `icp cycles transfer <amt> <backend-id>` |
| **your own account** | what funds both | `icp cycles balance` | mint from ICP |

An unfunded reserve looks like orders that pay and then never deliver.

- ⚠️ **A failed top-up reports the SENDER's balance.** `icp cycles transfer` answers
  *"insufficient funds. balance: N"* where `N` is your cycles-ledger balance, not the
  reserve's.
- ⚠️ **The reserve survives a canister reinstall; the floor does not.** A
  `--mode reinstall` leaves the stock intact and resets `reserveFloor` to 0. The fix is
  `refresh_reserve`, not a transfer.

## 5a. Admission gate: who is allowed to start an order

Order creation is refused before any quote when fulfilment is already impossible, so the
customer is refused before paying Stripe.

```bash
icp canister call backend lifecycle_config '()' -e ic     # public: gate AND delivery bounds
icp canister call backend can_purchase '(500 : nat)' -e ic  # public: would this be admitted?
# ⚠️ Read the CURRENT config and change one field. The record is whole-value: every
# field you type replaces the live one. This example restates the defaults below.
icp canister call backend set_gate_config \
  '(record { maxOpenOrdersPerPrincipal = 1 : nat; minCanisterCycles = 5_000_000_000_000 : nat;
             maxPurchaseUsdCents = 10_000 : nat; minPurchaseUsdCents = 1_000 : nat })' \
  -e ic --identity <operator>
```

| Lever | Default | What it protects | Sizing |
|---|---|---|---|
| `maxOpenOrdersPerPrincipal` | **1** | Unbounded state growth. Abandoned orders are the only thing a user can create for free. Nothing sweeps them away (order expiry, below): a slot frees when Stripe expires the session, when the buyer cancels, or via `expire_order`. | **1 is a product choice and it is felt.** A buyer who abandons a checkout cannot start another until that session expires (~35 min), including whoever is demoing this. Must be > 0. |
| `minCanisterCycles` | 5 T | **This canister's own gas.** Below it the gate stops admitting new orders. It does not gate delivery, cancellation or the webhook. | Sized against a gas drain, not against freezing: freezing is ~149x further down (~34 B), so at 5 T sales close with over a year of runway. It is the only bound on order flooding from rotating principals, and on a revoked Stripe key retrying its session outcall at ~220 M a try. `0` disables the check. |
| `maxPurchaseUsdCents` | **10 000 (\$100)** | Operator typo in a tier, and the webhook's upward repricing path. **It IS the per-order reserve exposure.** | Set just above your largest tier. `set_card_tiers` rejects any tier above it, and the webhook refuses to deliver against a payment above it. |
| `minPurchaseUsdCents` | **1 000 (\$10)** | A purchase too small to be worth an outcall and a reserve hold, and one that does not buy what a buyer came for. | Two independent floors hold it at \$10: the 30¢ fixed fee is 9.0% of \$5 against 5.9% of \$10, and \$5 buys 3.313 T against the 4.0 T two default-funded canisters need. `docs/BUYER-COST-MODEL.md` carries the model, and `test/buyer-cost.test.mo` pins it. |

All four default to non-zero, unlike the tier list: a limit where 0 would brick the
canister rather than protect it has to ship armed. **The Default column is pinned to the
code** by `test/gate.test.mo`, which fails with the lever that moved.

`can_purchase` returns the same decision `create_order` would make, so it is both the
frontend's button-gating call and the operator's "would a purchase go through right
now?" check. ⚠️ **It does NOT cover solvency, and cannot**: it is a query, and reading
the reserve is what the gate does synchronously inside `create_order`. A green
`can_purchase` alongside `reserve_status.availableToSell = 0` is the split working, and
an unobserved top-up is the most likely reason the rail goes quiet. Check both.

## 5b. Order expiry — Stripe owns the clock

There is no retention config, no TTL and no sweep. An order's deadline is its Checkout
Session's `expires_at` (~35 min, above Stripe's 30-minute floor), stored on the order,
and the only *event* that moves an order to `expired` is Stripe's
`checkout.session.expired`.

| | reaches `expired` | when it does not |
|---|---|---|
| `checkout.session.expired` | the normal path; releases the promise | never sent if the event is not subscribed (`docs/OPERATE.md`, Mode 2) |
| `cancel_order` (owner) | produces `cancelled`, also releasing | expires the session at Stripe first, so a paid race wins |
| `expire_order` (admin) | asks Stripe to expire, then settles | refuses `#sessionNotOpen` once the session has *already* expired or completed at Stripe, which is exactly the missed-event case below |

`expire_order` is therefore the manual release for an order whose session is still open
at Stripe, and for the residue class with no `stripeSessionId` at all. It is not the
remedy for a missed expiry event.

| Status | Payable? |
|---|---|
| `created` | yes, until the session's own `expiresAtNs` |
| `expired` | **no** |
| `cancelled` | **no** |

⚠️ **A missed `checkout.session.expired` leaves the order visibly `created` past its
`expiresAtNs`, and that is deliberate.** A sweep as a backstop would flip the order to
`expired` while its reserve promise stayed held, so a broken order would look like a
correctly expired one and the reserve would leak silently. The stuck order IS the
detection signal. ⚠️ **And it costs reserve capacity**: a `created` order holds its
promise from the moment it exists, and only `checkout.session.expired` and the buyer's
own `cancel_order` release one (`abandon_order` refuses a `created` order, since no money
was taken). `reserve_status.promisedTotal` climbing while `openOrders` also climbs is
what it looks like.

**The lever for this case is off-chain: resend `checkout.session.expired` from the Stripe
Dashboard** (monitoring, P2 row). The session has genuinely expired there, so
`expire_order` refuses it. Nothing on-chain surfaces the stranded order: order reads are
owner-scoped and `reserve_status` carries only counts, so the admin order listing is what
closes it.

⚠️ **A payment arriving against an expired or cancelled order cannot be converted**:
there is no `#expired → #paid` edge. It answers 200, the status does not move, and an
`#unattributed` entry is filed carrying the payment intent. **Refund it in Stripe.**

Nothing deletes an order. The record and its `client_reference_id` survive forever, which
is what keeps a late payment attributable, and therefore refundable rather than a
mystery charge.

### The outcall cost, and the one field that moves it

`create_order` spends the canister's own cycles on an HTTPS outcall, so
`minCanisterCycles` is the floor that closes the rail before the gas runs out. The cost
is computed exactly by `ic0.cost_http_request` (`Call.httpRequest` attaches precisely
that and never a buffer, because attached cycles are reserved for the call's duration).
What decides the number is `max_response_bytes` (`Session.maxResponseBytes`):

| Call | `max_response_bytes` | n = 13 (application subnets, and the local network) | n = 7 (the confidential subnet) |
|---|---|---|---|
| create a session (`create_order`) | 16,384 | **≈ 220 M cycles** (~$0.0003) | **≈ 118 M cycles** |
| expire a session (`cancel_order`, `expire_order`) | 16,384 | ≈ 220 M | ≈ 118 M |
| **retrieve a session** (the recovery sweep) | **32,768** | **≈ 390 M cycles** | **≈ 207 M cycles** |

At 20 T gas with a 5 T floor, the headroom depends on whether webhooks are working:

| | cost per abandoned order | creations before the rail closes (n=13) | (n=7) |
|---|---|---|---|
| **Webhooks healthy** | one create, ≈222 M | **≈68,000** | ≈128,000 |
| **Webhooks failing** (a missed expiry event per order) | create + retrieve, ≈612 M | **≈24,000** | ≈46,000 |

The retrieve is not on the abuse path in normal operation: Stripe fires
`checkout.session.expired` within seconds of the deadline, so an abandoned order is
already `#expired` before the sweep would look at it (`expiryCheckDue` tests the status
first). The second row needs the webhook path broken as well, which is a different
incident with its own P1 rows. Either way this is availability, not memory: it closes the
rail via `minCanisterCycles` rather than freezing the canister, and it takes hours of
sustained paid-for abuse. Recompute these figures from the formula if a cap changes.

**The retrieve's cap is deliberately double the others'.** A completed session carries
`customer_details`, a resolved `payment_intent` and `total_details` that a fresh one does
not, and an over-cap response fails the call outright rather than truncating. The sweep
only retrieves for orders that are already stranded, and the stranded population is
correlated (one unprovisioned webhook secret strands every order in its window at once),
which is why the sweep caps retrieves per pass (`Recovery.maxRetrievesPerPass`).

**The cap counts response headers, not just the body, and it is checked twice**: on the
raw response and on the transform's Candid-encoded output. Three distinct rejects:
`Header size exceeds specified response size limit`, `Http body exceeds size limit of
<N>` (prints the full cap, not the remainder after headers, so the body that failed can
be well under `<N>`), and `Transformed http response exceeds limit`. Raising the cap fixes
all three; stripping headers in the transform fixes only the last.

⚠️ **`No consensus could be reached` means the transform, not Stripe.** It is the
signature of a per-request value not being stripped, it takes the whole rail down, and
no test suite in this repo can catch it. `Session.classifyFailure` labels it in the
audit log for that reason.

### Growth

Growth is bounded at its source: `maxOpenOrdersPerPrincipal` bounds what a user can
create for free, and the reserve bounds legitimate volume. An order is a few hundred
bytes, so a million orders is a few hundred MB and millions of dollars of volume. If
store size ever binds, archive to a separate canister; deleting a financial record is
not the answer. Monitor `reserve_status.openOrders`: climbing while `delivered` does not
is the signature of order-creation abuse. `totalOrders` and `paidIntentsIndexed` should
grow together.

## 6. Obligations — triage (§4.1)

```bash
icp canister call backend orphans '(null, 50 : nat)' -e ic --identity <operator>
icp canister call backend resolve_orphan '(42)' -e ic --identity <operator>
icp canister call backend delivery_journal '("<orderId>")' -e ic --identity <operator>
```

The queue is the operator worklist: resolving an entry lives on the entry and never
transitions the order. The order's own status says whether anything is still owed:

| Status | Meaning | The order's promise |
|---|---|---|
| `NeedsReview` | a money position nobody knows the outcome of. **Check the ledger.** | **still held** |
| `Abandoned` | you ended it, having refunded by hand. Terminal. | **released** |

**How an order gets to `NeedsReview`**, since the triage depends on it:

| Route | Money position | What you do |
|---|---|---|
| the intent aged past the ledger's ~24 h dedup window, or the ledger answered `#TooOld` | **unknown**: a replay is no longer protected | establish the fate on the ledger; the order id is in the transfer's **memo** |
| §5.3's 72 h max-wait, on an order where **nothing was ever sent** | **certain**: fiat in, nothing moved | refund in the Stripe Dashboard |
| `journalInconsistent` | unreachable guard | if this ever fires, `lockedCycles` acquired a second writer: a much bigger problem than one order |

`delivery_journal(orderId)` and the entry's `detail` say which; `terminationFor` derives
it from the journal because the status cannot tell these apart. Reaching the unknown
case at all takes a ~day-long cycles-ledger outage; treat it as expected-never.

`NeedsReview` has exactly two exits, and both are your finding rather than the gateway's:

| You established, on the ledger | Call | Result |
|---|---|---|
| the transfer **did** land | `record_delivered '("<orderId>", <blockIndex>)'` then `resolve_orphan '(<entryId>)'` | `Delivered`, with the block recorded in the journal |
| it did **not**, and you refunded the fiat by hand | `abandon_order '("<orderId>", "<reason>")'` then `resolve_orphan '(<entryId>)'` | `Abandoned`, reason in the audit trail |

- ⚠️ **Neither lever closes the queue entry; `resolve_orphan` is the last step, always.**
  Moving the order does not resolve the entry, so a finished order can sit behind an
  open worklist item if you stop after the first command.
- ⚠️ **You cannot `abandon_order` a `Paid` order whose delivery is still outstanding.**
  The lever refuses and names `pending_deliveries`, because abandoning an unknown
  position releases the promise and files a refund while the transfer may already have
  landed. It is a wait: the ~24 h fuse moves such an order to `NeedsReview`.
- `record_delivered` requires the block index: it is the evidence that you looked, and
  the order id is in the transfer's memo, so finding it is a ledger search. Nothing
  automatic reaches `Delivered` from `NeedsReview`, because re-driving an unknown money
  position is the double-delivery this status prevents.
- ⚠️ **Never treat `NeedsReview` as finished.** It is the status that still owes cycles.
- **Only a full `charge.refunded` auto-resolves an entry.** A partial refund leaves the
  entry open and audits `stripe.refundPartial`.

**Nothing is ever evicted.** An unresolved entry is an open obligation, so the list
grows rather than dropping one, and `orphan_depth` is a real alarm: a depth climbing
past ~1,000 means unresolved work is accumulating faster than it is being cleared.

```bash
icp canister call backend orphan_depth '()' -e ic          # public: {total; unresolved}
icp canister call backend orphans_unresolved '(null, 50)' -e ic --identity <operator>
icp canister call backend orphans '(opt (120 : nat), 50)' -e ic --identity <operator>
icp canister call backend resolve_orphan '(137 : nat)' -e ic --identity <operator>
```

`orphans_unresolved` is the worklist; pass the last id returned as `afterId` to page
forward. Page size is capped at 200.

**Two columns decide everything: the money position, and whether a refund can settle it
on its own** (`Orphans.refundResolvable`, true only where the remedy is exactly "refund
the fiat", so the `charge.refunded` webhook can close the entry without a human).

| Kind | Refund settles it? | Money position | Action |
|---|---|---|---|
| `#duplicate {orderId; paymentRef}` | ✅ **yes**, automatically | Fiat in twice for one order; the second payment delivered nothing | Refund `paymentRef` in the Stripe Dashboard (search by payment_intent). The `charge.refunded` webhook auto-resolves the entry; `resolve_orphan` is the fallback. |
| `#unattributed {claimedRef; paymentRef}` | ✅ **yes**, automatically | Fiat in, and no order that can accept it: a bad or missing `client_reference_id`, an owner/rail/currency mismatch, **a paid amount that is not the one the order asked Stripe for**, or a payment against a `cancelled` or `expired` order (the common producer). The entry's `detail` says which | Inspect the session in Stripe by `paymentRef`, then **refund** → auto-resolve. This is the only remedy, whatever the order's status. If the detail says the amount is not the quoted one, something in the Stripe configuration moved the total: check the forbidden-settings list in section 3 before the next order, because it will recur. |
| `#deliveryStuck {stage}` (on the order) | ❌ **no**; see the stage table below | **Depends entirely on `stage`.** One of them means the buyer may already hold their cycles, so a blind refund pays twice | Read `stage` first, then follow its row |
| `#refundAfterDelivery {paymentRef; cycles; refundedCents; fullRefund}` (on the order) | ❌ **no, and never** | **A loss, not a recoverable position**: the fiat was refunded or charged back *after* the cycles were credited | Nothing to recover on-chain. Reconcile in the Dashboard by `paymentRef` to see whether this was your own refund or a dispute. For repeated disputes, tighten Stripe Radar and lower the per-purchase ceiling. **Nothing auto-resolves it, deliberately: the refund is the event that created it** |
| `#paidNotCredited {orderId; paymentRef; sessionId}` | ❌ **no, and a refund alone makes it worse** | **The buyer paid and this gateway never credited them.** Found by the recovery sweep asking Stripe about a `#created` order past its deadline, once Stripe has had longer than its ~3-day redelivery window. The order is still `Created`, so it still holds reserve capacity, correctly | **RESEND FIRST, ALWAYS.** Find the event in the Stripe Dashboard (search by `paymentRef`) and resend `checkout.session.completed`. That credits the order through the normal path, delivers the cycles, and closes this entry automatically. <br><br>**Refunding instead does not settle it, and leaves no way out**: the order stays in `Created` holding capacity, a complete session never fires `checkout.session.expired`, `expire_order` refuses (session not open), and `abandon_order` cannot act on `Created`. If you have already refunded: resend anyway to move the order to `Paid`, then `abandon_order`. The buyer keeps cycles they were refunded for; that is the cost of refunding first. The entry does not auto-resolve on `charge.refunded`, deliberately |
| `#unprocessable {eventId; field}` | ❌ no; the position is unknown | A verified Stripe event was missing a required field, so the canister could not tell whether money moved | Look the `eventId` up in the Dashboard. **Paid** → refund. **Not paid** → nothing happened; `resolve_orphan`. Then find the configuration that produced it: the canister controls every field it sends, so a missing one points at an account-level API-version change and will recur until fixed |

### `#deliveryStuck`'s stages — read `stage`, never the kind

**`stage` is the money position**, derived from the journal, not the status. These are the
whole vocabulary `Delivery.terminationFor` and the escalate route can emit, pinned by
`test/delivery.test.mo`, so a stage without a row here is a bug in the code.

| `stage` | Money position | Action |
|---|---|---|
| `staleIntent` | **UNKNOWN.** A transfer was issued, no block was recorded, and the intent is past the ledger's ~24 h dedup window, so re-sending could pay twice | **Establish its fate on the cycles ledger**, matching the order id in the transfer's **memo**, the `created_at_time` and the amount from `delivery_journal(orderId)`. **Executed** → `record_delivered '("<orderId>", <blockIndex>)'`, then `resolve_orphan`. **Not executed** → refund in Stripe, `abandon_order`, then `resolve_orphan`. **Never rebuild the intent**: past the window a rebuilt one pays twice |
| `landedNotRecorded` | **Certain, and in the buyer's favour**: the transfer landed and the order never moved to delivered. **The buyer HAS their cycles** | Confirm the block on the cycles ledger, then `record_delivered '("<orderId>", <blockIndex>)'` → `resolve_orphan`. **Do NOT re-send.** Should be unreachable (the block and the transition commit in one synchronous block), so file it as a bug too |
| `deliveryWaitExceeded` | **Certain**: fiat in, nothing was ever sent before the 72 h bound | Refund in the Stripe Dashboard, `abandon_order '("<orderId>", "<reason>")'`, then `resolve_orphan` |
| `transferRejected` | The cycles ledger refused the call definitively, so nothing moved | Read the `detail` for the ledger's reason. `#InsufficientFunds` → the reserve is short: top it up and `refresh_reserve`, then re-drive with `process_order`. Refunding is always a valid resolution |
| `journalInconsistent` | **An invariant breach, not a money position**: the intent's amount exceeds the order's locked quantity | **File it as a bug**: `lockedCycles` has acquired a second writer. Establish the transfer's fate on the ledger before re-sending anything |
| `missingJournal` / `notInFlight` | **Also invariant breaches.** The order's status implies money-out work the journal cannot support | Reconstruct from `audit_log` and the cycles ledger; treat as a bug and file it |

### An unattributed payment has exactly one remedy: refund

⚠️ **Read the entry's `detail`, not its kind.** `#unattributed` covers two situations:

| | What it means | How common |
|---|---|---|
| Genuinely unattributable | no order can be named: missing, malformed or unresolvable `client_reference_id`, wrong currency | **should not happen**: the canister sets that field itself through the API, so treat one as a bug to find (or a session someone created outside the app) |
| Attributable but unpayable | the entry names the order; we refuse to credit it: a lowered ceiling, an amount that is not the quoted one, a cancelled or expired order | the normal producer |

The second kind needs no hunting: the order id is in the detail. What it needs is fixing
the cause, or the next order fails the same way. There is no path that turns an
unattributed payment into cycles for that buyer, whether or not we know which order it
was for; the app does not model refunds, and a refund in the Dashboard auto-resolves the
entry.

### Closing an order-bound problem

Four of the six kinds live on the order rather than in the orphan list, and are closed
with `resolve_problem` rather than `resolve_orphan`:

```sh
# Read ONE order, whoever owns it. Every such read is audited, hit or miss.
icp canister call backend admin_order '("<orderId>")'

# See what is outstanding, and on which orders
icp canister call backend problem_depth '()'                 # the number to alert on
icp canister call backend admin_orders '(record { status = null; owner = null; createdFromNs = null; createdToNs = null; withUnresolvedProblems = true }, null, 50)'

# Read ONE order's receipt, whoever owns it. Audited, like admin_order.
icp canister call backend admin_receipt '("<orderId>")'

# Close one. The second argument is a VARIANT, not a string. The third selects
# WHICH problem, by payment reference.
icp canister call backend resolve_problem '("<orderId>", variant { duplicate }, opt "pi_...")'

# `deliveryStuck` can only ever have one per order, so null is always right:
icp canister call backend resolve_problem '("<orderId>", variant { deliveryStuck }, null)'
```

⚠️ **`admin_orders` is ordered by order id, NOT by time, and the last argument is a page
size.** Order ids are `raw_rand` hex, so the first page is not the most recent orders: to
look at recent orders, set `createdFromNs`. `null` in the second position means "from the
beginning"; pass the returned `nextCursor` back to continue. Sorting by time is
deliberately not offered: it would materialise the whole filtered set, an unbounded scan.

⚠️ **Passing `null` when the order has several problems of that kind is REFUSED**
(`#ambiguous`, with a `candidates` list). A buyer who pays three times files three
`#duplicate` problems, and closing "the duplicate" would mark settled a payment you have
not refunded.

A resolved problem stays on the order; the worklist filter stops listing it while its
history remains readable.

| kind tag | ref needed? | what closing it means |
|---|---|---|
| `duplicate` | **yes** when several exist | you refunded that specific second payment in Stripe |
| `deliveryStuck` | no; one per order | you established the money position and acted (the stage table) |
| `refundAfterDelivery` | **yes** when several exist | you reconciled the recorded loss; the cycles are not recoverable |
| `paidNotCredited` | **yes** when several exist | normally closes itself on the resend; closing it by hand says you gave up on crediting the buyer |

## 7. Recovery timer & manual kicks (§5.2)

The recurring sweep is the backstop for every detached delivery kick that dies: it
re-drives every order in `paid`. It re-arms automatically on every upgrade (transient
initializer).

```bash
icp canister call backend recovery_status '()' -e ic            # public
icp canister call backend set_recovery_interval '(3_600_000_000_000)' -e ic --identity <operator>
icp canister call backend process_order '("<orderId>")' -e ic --identity <operator>
```

- Interval validation pins cadence ≤ 6 h (ledger dedup window ÷ 4: a stuck transfer
  must get several replay attempts while its intent still dedups). Default 15 min
  (`Recovery.defaultIntervalNs`); re-arms immediately on change.
- `recovery_status.lastSweep` not advancing past ~2 intervals means the timer is wedged.
  An upgrade re-arms it, but investigate first.
- The sweep **reconciles the per-status tallies once a day** and reports on
  `recovery_status.lastCountReconcile`. The tallies are maintained incrementally so the
  admission-gate queries stay O(1); the reconcile checks they still match, and audits
  only when something moved. `recount_orders` runs the same pass on demand. Its cost is
  bounded by open orders, not lifetime sales: `lastCountReconcile.ordersRead` should
  stay flat as sales accumulate.
- ⚠️ **`drift` and `refused` demand opposite readings.** `drift` was raised to the
  recount, so those tallies are correct again. `refused` came out below the maintained
  tally and was not adopted, so those tallies are still suspect: a recount lower than
  the tally is indistinguishable from an index missing a member, and adopting it is the
  only way a bookkeeping bug could lower `promised` and oversell the reserve. There is
  deliberately no force flag.
- **One check cannot be daily.** Whether anything outside an index satisfies the index's
  predicate needs every order, so it runs as a rotating per-order scan, one chunk per
  sweep. `recovery_status.indexScan` is its coverage: `lastCompletedCycle` is the only
  thing that licenses reading a clean scan as evidence about the whole store. Three
  states: an audit line is *verified and disagreed*, silence with a recent
  `completedAtNs` is *verified clean*, and silence without one is *unverified*. The
  cycle grows linearly in stored orders (`⌈storedOrders ÷ chunkSize⌉ × sweep interval`;
  `indexScan.expectedFullCycleNs` is the live value), and `set_recovery_interval` is a
  lever on that latency: coarsening to the 6 h ceiling takes 24× longer. The finding it
  delays is `orders.unindexedHolders`, which is P1.
- The sweep also **reconciles the reserve floor against the cycles ledger once an hour**,
  which is how a top-up becomes sellable without an operator call.
  `recovery_status.lastReserveReconcileAttemptNs` is the attempt clock and
  `reserve_status.reserveObservedAtNs` the success one; the two diverging means the read
  is failing or every attempt landed while a delivery was in flight. Both under-sell.
- `process_order` is the safe-to-spam manual kick for one order (per-order
  single-flight; `#inFlight` means it is already being driven). It is admin *or* the
  order's own owner, so a buyer's page refresh heals their own stuck delivery in seconds.
  The sweep is still the guarantee; this is the latency fix.

## 8. Monitoring plan

Every safety mechanism in this system is a number someone has to look at. An alert
nobody receives is not an alert, so wire this before taking money.

### The whole alerting layer needs no credentials

These are public queries, so a monitor can poll them anonymously:

<!-- surface:public -->

`can_purchase` · `card_tiers` · `cycles_status` · `delivery_stats` · `expected_livemode` ·
`health` ·
`admin_status` · `lifecycle_config` · `operator_summary` · `orphan_depth` ·
`pricing_status` · `problem_depth` ·
`quote_previews` · `recovery_status` · `refusal_counts` · `reserve_status` ·
`stripe_origin`

<!-- /surface -->

The ones worth a monitor: `health`, `cycles_status`, `reserve_status`, `pricing_status`,
`recovery_status`, `orphan_depth`, `problem_depth` and `refusal_counts`.

The anonymous principal owns no orders, so `tooManyOpenOrders` can never trip for it:
**anonymous `can_purchase '(<smallest tier cents>)'` is a pure global-health probe**,
and every reason it can give is actionable. ⚠️ It cannot see solvency, so pair it with
`reserve_status.availableToSell`.

### The reserve is the one metric anyone can poll

The reserve is an account on the cycles ledger, so `icrc1_balance_of` answers for free
from any identity. `reserve_status` adds `reserveFloor − promisedTotal =
availableToSell`, and the three figures together separate the causes of a refusal:

| Reading | Means |
|---|---|
| ledger balance high, `reserveFloor` low | a top-up nobody observed → `refresh_reserve` |
| `reserveFloor` fine, `promisedTotal` high | genuinely committed to live orders → wait or raise the reserve |
| `availableToSell` 0 with both low | the reserve is spent → fund it |

### Metric table

Severity: **P1** = wake someone; **P2** = same working day; **P3** = review weekly.

| Metric | Alert when | Sev | Action |
|---|---|---|---|
| `pricing_status.lastAttempt.ok` | false on two consecutive ticks | **P1** | section 4: order creation stops once the cache passes `maxAgeNs`. `detail` names the failing guard |
| `pricing_status.rates.fetchedAtNs` | older than `maxAgeNs` | **P1** | the rail has stopped selling. Timer dead or every tick rejected |
| `cycles_status.balance` | below 3× `minCanisterCycles` | **P1** | top up. At zero the canister is uninstalled and money-bearing state is lost. The XRC needs 1 B attached per refresh, so pricing dies before the gate does |
| anonymous `can_purchase` | returns `#err` | **P1** | the rail is refusing sales; the reason says which lever |
| `orphan_depth.unresolved` | `> 0` | **P2** | triage. Depth climbing past 1,000 means work is accumulating faster than it clears |
| `reserve_status.promisedTotal` | climbing while deliveries do not complete | **P2** | money in, nothing delivered; those orders are on the clock toward the 72 h bound. `pending_deliveries` says which and why |
| `recovery_status.lastSweep.atNs` | older than 2 intervals | **P2** | the sweep timer is not running; nothing recovers while it is dead |
| `recovery_status.lastCountReconcile.drift` | non-empty | **P2** | a per-status tally had diverged and was raised to the recount. The counts are correct again; the bookkeeping bug that moved them is not fixed. An under-counted `Paid` reads as zero to the recovery sweep and money-out silently stops |
| `recovery_status.lastCountReconcile.refused` | non-empty | **P2** | the recount came out below the maintained tally, which is indistinguishable from the non-terminal index missing a member, so it was refused and the tallies are still suspect. Find the writer: either the index lost a member or a tally gained an adjustment. `orders.unindexedHolders` from the rotating scan tells you which |
| `orders.unindexedHolders` in the audit log | any occurrence | **P1** | ⚠️ **The one bookkeeping error nothing else can see, and it is on the money side.** An order held a promise and was missing from the non-terminal index, so `reserve_status.availableToSell` read higher than the truth: the reserve was oversellable. The scan added the order and the next reconcile raises `promised`; what does not close is the writer that set a status outside `Orders.create` and `Orders.commitTransition`. Find it, then check `availableToSell` against the ledger before selling more |
| `orders.staleHolders` in the audit log | any occurrence | **P2** | the reverse: a terminal order was still in the non-terminal index, so `promised` may have been holding released cycles. Over-refusing, not overselling. Same writer to find |
| `orders.problemIndexDrift` / `orders.unindexedProblems` in the audit log | any occurrence | **P2** | **A code problem.** The unresolved-problems index is maintained by exactly `Orders.fileProblem` and `Orders.resolveProblems`; either tag means something else wrote `order.problems` directly. `unindexedProblems` is the worse direction: an order carried an unresolved obligation the worklist did not show |
| `orders.expiredWentBackwards` in the audit log | any occurrence | **P2** | the `Expired` tally fell, which the transition matrix makes impossible. A bookkeeping breach in `Orders.bump`, reported once per decrease. `Expired` is an operator metric that nothing decides on, so this is a bug report rather than an exposure |
| `orders.expiredOverflow` in the audit log | any occurrence | **P2** | the `Expired` tally plus the non-terminal count exceeds the orders in the store, which is arithmetically impossible. Same class as the row above |
| `recovery_status.indexScan.lastCompletedCycle` | empty, or `completedAtNs` older than a small multiple of `indexScan.expectedFullCycleNs` | **P2** | without this the clean-scan rows above mean nothing. Compare against `expectedFullCycleNs`, not a remembered number: `set_recovery_interval` can move it by 24×. Empty long past that, with `inFlightCycle.ordersRead` frozen, means a chunk is trapping |
| `recovery_status.lastCountReconcile.atNs` | older than ~48 h while `lastSweep` advances | **P3** | the daily reconcile is failing. It runs in its own message, so money-out is unaffected, but the tallies are now unverified. `recount_orders` will show the same failure if it is a real one |
| `reserve_status.openOrders` | climbing while `delivered` does not | **P3** | order-creation abuse; lever is `maxOpenOrdersPerPrincipal` (section 5a) |
| `refusal_counts.refusingNow.reserveShort` | true | **P1** | the rail is refusing sales for want of reserve. One `gate.startedRefusing` line marks when it began. Fund the reserve (section 5); it clears on the next successful admission |
| `refusal_counts.refusingNow.canisterCyclesLow` | true | **P1** | the gate is refusing on its own gas: top up the canister (`cycles_status`). Do not expect a corroborating `pricing_status` alert: at the 5 T default the gate closes while the balance is still ~5000x what a rate call must attach. Sales are closed but delivery is not |
| `refusal_counts.refusingNow.railClosed` | true | **P1** | ⚠️ **The rail is not provisioned, so every purchase is refused before the gate is consulted.** Expected on a fresh deployment and an incident at any other time. `stripe_api_key_status().isSet` and `stripe_origin()` say which half is missing |
| `refusal_counts.refusingNow.stripeApiFailing` | true | **P1** | ⚠️ **The key is present but Stripe is refusing it: rotated or revoked without updating the canister.** Every purchase reaches the outcall, 401s, and is refused. ⚠️ Each attempt also commits an order and expires it, and the open-order cap does not bound that, so the order table grows while this is true. Set a fresh restricted key (section 3); it clears on the next successful session |
| `refusal_counts.counts.amountBelowMin` | climbing | **P3** | either the buy UI is offering an amount the gate refuses (a bug) or someone is probing the cheapest free refusal there is. Compare against `reserve_status.openOrders`: climbing refusals with flat order creation is probing |
| `refusal_counts.counts.amountAboveMax` | climbing | **P3** | buyers are asking for more than the per-order ceiling allows. Raising it raises per-order reserve exposure, so treat it as a pricing decision |
| `refusal_counts.counts.tooManyOpenOrders` | climbing | **P3** | buyers hitting the one-open-order cap. Expected in normal use, so alert on the rate, not the total |
| `refusal_counts.counts.railClosed` | climbing while `refusingNow.railClosed` is **false** | **P2** | should be unreachable: `sessionConfig` can only fail with a missing key or origin, both of which latch. Re-read `Main.mo`'s `railClosureCondition` against the current `Session.Error` |
| `reserve_status.availableToSell` | 0, or far below what you expect | **P2** | three causes and the same query separates them: the reserve is spent (`reserveFloor` low), it is committed to live orders (`promisedTotal` high), or the floor has not observed a top-up (`reserveObservedAtNs` old). The last is the common one and the lever is `refresh_reserve` |
| `reserve_status.reserveObservedAtNs` | materially older than `recovery_status.lastReserveReconcileAttemptNs` | **P3** | the hourly reserve reconcile is attempting and not adopting: the ledger read is failing (`reserve.observeFailed`) or every attempt lands while a delivery is in flight (`reserve.reconcileSkipped`). Under-sells; not a loss |
| `delivery.feeChanged` in the audit log | on **every** delivery rather than once | **P3** | the stored cycles-ledger fee is not sticking (an upgrade reverting it, or the ledger's fee moving repeatedly). This is the only detector for that, since the persistence itself is untested (`docs/TEST-COVERAGE.md`). Each occurrence costs one rejected call, never a wrong debit. If it repeats, redeploy; there is no lever, deliberately |
| `refusal_counts.refusingNow.stripeApiFailing` true, with `gate.startedRefusing` naming **retrieve REFUSED** | any occurrence | **P1** | the restricted key cannot read Checkout Sessions, so stranded capacity can never be released automatically. Fix the key's permission (Checkout Sessions = **Write**) and rotate it in. Until then `expire_order` is the manual release. A 401 on create, expire or retrieve is one incident with one lever |
| buyers report that cancelling does nothing, or `admin_orders` shows no order has ever reached `cancelled` | any occurrence | **P2** | `cancel_order`'s "Stripe would not close the payment session" answer has three causes and the buyer-facing arm records none of them. Two settle themselves (the payment won the race, or the session had already expired); the third is a malformed expire request from this canister, which leaves the order `#created` and payable while every cancel fails. One manual `expire_order '("<a live created order>")'` takes the identical path and audits Stripe's body verbatim as `order.expireRaced`. A session-state refusal there means the two normal causes; anything else is ours to fix |
| `stripe.retrieveFailed` in the audit log | repeatedly for the same order | **P3** | Stripe is unreachable or answering non-200 for the session read. Distinct from the row above: "Stripe refused the read" and "Stripe is down" are different actions. A persistent one means the outcall path is broken, so check `pricing_status` (the same egress) before suspecting the key |
| `stripe.paidAwaitingEvent` in the audit log | any occurrence | **P2** | a buyer paid and Stripe has not delivered `checkout.session.completed`. Not yet an obligation (Stripe redelivers for ~3 days), but it is the support signal: the buyer's page renders expired from `expiresAtNs`. One order: resend the event from the Dashboard now. Many: the webhook endpoint is broken; check the secret and the subscribed event list |
| `stripe.paidNotCredited` in the audit log | any occurrence | **P1** | a buyer paid, Stripe has given up redelivering, and the obligation is filed on the order. Follow the `#paidNotCredited` triage row: **resend first, always** |
| `stripe.retrieveUnreadable` in the audit log | any occurrence | **P3** | Stripe answered the session read in a shape the classifier does not recognise, so the sweep did nothing (fail-safe). Most likely an API-version change |
| `reserve.unexplainedShortfall` in the audit log | any occurrence | **P1** | the ledger holds LESS than the floor's lower bound, which the design says is impossible. Treat as a bookkeeping breach: stop selling (`set_gate_config` with a high `minPurchaseUsdCents`), reconcile the journal against the ledger, and find the outflow before funding anything |
| an order still `created` past its own `expiresAtNs` | any | **P2** | a `checkout.session.expired` was missed. Nothing sweeps it (section 5b): resend the event from the Stripe Dashboard. Query with `admin_orders '(record { status = opt variant { created }; owner = null; createdFromNs = null; createdToNs = null; withUnresolvedProblems = false }, null, 200)'` and compare each `expiresAtNs` against now |
| `pricing_status.xrcCanisterId` | anything other than `uf6dk-hyaaa-aaaaq-qaaaq-cai` | **P1** | on mainnet this must be the real Exchange Rate Canister. The id is resolved from a `PUBLIC_CANISTER_ID:xrc` canister environment variable so a local network can point at a mock; only a controller can inject it, so any other id is a misconfigured deploy or a compromised controller. To stop new orders while investigating, `set_gate_config` with `minCanisterCycles` above the canister's current balance. **`null` is not a pass**: it means no refresh has reached the XRC call yet (expected for seconds after an install or upgrade, since the value is transient). Re-read until `lastAttempt.atNs` post-dates the deploy |
| `pricing_status.rates.quality.receivedRates` | drops to `minRateSources` | **P3** | thin market |
| `health` | unreachable | **P1** | canister stopped, frozen, or out of cycles |

"An order still `created` past its own `expiresAtNs`" needs a controller key and a
comparison you make yourself: the predicate is a comparison against the clock, so there
is no field to alert on. The public interim signal is `reserve_status.openOrders`
staying non-zero and static well past ~40 minutes (the session lifetime is ~35),
cross-checked against expired sessions in Stripe. `get_order` / `list_orders` / `receipt`
stay owner-scoped; `admin_order` / `admin_orders` / `admin_receipt` are the controller-side
reads.

### Needs a controller key

- **`audit_log` tags** worth alerting on: `stripe.livemodeMismatch` (real money may be
  landing in the wrong Stripe account), `stripe.creditedElsewhere`,
  `stripe.unprocessable` / `stripe.unhandledType` (a Dashboard config producing events
  this gateway cannot use), `orders.unindexedHolders` (**P1**), `stripe.refundPartial`
  (an obligation deliberately left open), and `delivery.stuck`.
- **`orphans_unresolved`** for the entries themselves. `orphan_depth` is public, so alert
  on the public depth and only fetch details when it fires.
- The audit log drops nothing, so a gap in `seq` is not a signal. The log is telemetry:
  the order store, delivery journal and orphan list are the records of money.

### Off-chain

- **Stripe Dashboard → event deliveries.** A run of failures means the secret is out of
  sync or the gateway is unhealthy. Stripe retries non-2xx for ~3 days; a permanent 4xx
  can get the endpoint disabled, which is why verified-but-unprocessable events are
  acked 200.
- **Stripe payouts** have no on-chain signal. **Disputes do**: `charge.dispute.created` is
  audited as `stripe.disputeCreated` (`rails/Card.mo`), carrying the intent and the
  amount. Audit-only; the Dashboard is still where it is resolved.

### Do you need a dashboard?

Alerting first, dashboard second. The failure modes here are slow (a 2 h alert window, a
72 h terminate bound), so what you need is something that reaches a human at 03:00. A
cron job polling the public queries and posting to Slack/PagerDuty covers the entire
table above except the audit tags, and needs no credentials. A dashboard earns its place
afterwards, for reserve drawdown over time, order volume and rate quality; it is a
convenience, not a control.

## 9. Confidential-subnet checklist (§7, §11.1)

The webhook secret is plaintext canister state, and SEV-SNP is the intended
confidentiality layer. A forged webhook delivers from the reserve, so **the reserve
balance is the blast radius** and sizing it is the always-on control. Launch does not
block on SEV. Before relying on a confidential subnet for the secret, verify, hardest
first:

- [x] **Checkpoint/state-sync confidentiality**: confirmed encrypted on the target subnet.
  SEV-SNP protects RAM; canister state is also checkpointed to disk and state-synced
  between nodes, and both paths are confidential here too. Either one in the clear would
  have leaked the plaintext secret and SEV would have bought nothing.
- [ ] **Attestation coverage**: every replica in the subnet runs attested SEV-SNP (one
  unattested node = one node provider who can read the secret).
- [ ] **Production readiness**: the subnet is GA, not a beta, and check the current AMD
  SEV-SNP CVE list.
- [ ] **Migration**: moving subnets is a canister migration; re-verify the module hash
  after (`RELEASE.md` gate) and rotate the secret.

Until all boxes tick, the protections are exactly (a) the reserve sized to what a leak
could cost and (b) accountable node providers. That is the documented, accepted §7
posture.

Any new rail (§11.1) lands with its own runbook section, its own dedup set, and its own
entry in `docs/OPERATE.md`'s Mode 3.

## 10. Upgrades & releases

`RELEASE.md` end to end: reproducible container build → publish `MODULE-HASHES.txt` →
install → gate on `icp canister status` matching the published hash. Operational notes
the release doc does not cover:

- ⚠️ **Stop the canister before upgrading.** The IC rejects an upgrade while the canister
  has outstanding message callbacks:

  ```
  canister_pre_upgrade attempted with outstanding message callbacks
  (try stopping the canister before upgrade)
  ```

  ```bash
  icp canister stop backend -e ic --identity <operator>
  icp deploy -e ic --mode upgrade
  icp canister start backend -e ic --identity <operator>
  ```

  `icp deploy --mode upgrade` sets the `wasm_memory_persistence = keep` option that
  enhanced orthogonal persistence requires; a hand-rolled `install_code` without it is
  rejected, and `replace` would discard every order, journal, and dedup set.

- **Stopping drains in-flight calls, it does not drop them.** The canister enters
  `Stopping`, the IC delivers the replies to its outstanding calls, and only then does
  it reach `Stopped`. So an in-flight delivery completes before the upgrade happens, and
  a controlled upgrade cannot strand money (integration scenario 12). Consequently
  `staleIntent` is not reachable through a controlled upgrade; it covers genuine faults.

- **A call that never replies blocks the stop, and therefore the upgrade.** If
  `icp canister stop` hangs, check `recovery_status.sweepInFlight` and the audit log for
  a stage that keeps retrying; the money path is journalled at every step, so waiting is
  safe.

- **Locally, a stable-shape change is a reinstall, not a migration** (`docs/OPERATE.md`,
  Mode 1). On mainnet, `reinstall` discards every order, journal and dedup set, so a
  shape change needs the mops migration chain (`docs/OPERATE.md`, Mode 3, "What is not a
  command", item 1).

- In-flight deliveries resume from the persisted journal via the re-armed timer (§5.1).
  Persistent state (orders, their problems, journals, dedup sets, configs, secrets)
  survives upgrades. Transient knobs reset on upgrade: the HTTP body cap (64 KiB), the
  single-flight guards, and the index-scan cursor's in-flight cycle, deliberately, so a
  guard stuck by an upgrade cannot deadlock anything.

- After every upgrade: `health`, `recovery_status` (timer re-armed),
  `webhook_secret_status.generation` unchanged, and one test order end to end if the
  change touched money paths.

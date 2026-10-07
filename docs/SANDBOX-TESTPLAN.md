# Stripe sandbox test plan (manual, human-in-the-browser)

Most Stripe payloads in the automated suites are hand-crafted JSON written from the API
docs, so those suites prove the canister behaves as designed against *our* idea of
Stripe. This plan closes that gap and captures real fixtures while doing it. **Read
"What this cannot tell you" at the end before treating a green run as go-live
approval.** It is not one.

## Status: run against reserve delivery on 2026-08-28

A row here is a record of what a run on a given day observed, and the mechanism under it
may have changed since. The 2026-08-28 run, all of it against `icrc1_transfer` out of the
reserve:

| Flow | Result |
|---|---|
| $20 → paid | delivered |
| $13, custom amount → paid | delivered |
| $10 → cancelled by the buyer | released; the open-order cap of 1 refused a second order while one was open |
| $10 → left to expire | expired by Stripe (`stripe.sessionExpired`), promise released exactly: `promisedTotal` 7,238,461,538,461 → 0 |
| full refund of a delivered order | `#refundAfterDelivery`, `refundedCents = 1300`, `fullRefund = true`, correctly not auto-resolved |

Stripe fired `checkout.session.expired` within ~2 s of `expires_at`. The session TTL is
35 minutes (`Session.requestedLifetimeSeconds = 2_100`), not 30: Stripe's floor is
evaluated against *its* clock on arrival, so sitting exactly on it fails every session
creation.

The 2026-08-13 run (kept only for the Stripe-side figures; its delivery half ran a mint
path that no longer exists) credited two purchases to the buyer's cycles-ledger account:
`icp cycles balance --of-principal <buyer>` → 18,207,492,307,692.

**What is owed before go-live:**

| Gap | Where |
|---|---|
| ⚠️ **The buyer's cycles-ledger balance, read from the LEDGER, on a reserve delivery.** The 08-28 run confirmed the order view said `delivered` and an audit line recorded a block index, which is the canister's own account of the transfer, not an independent observation. One command: `icp cycles balance --of-principal <buyer>` before and after | group H |
| The CLI handoff: `icp identity link web` was never run, so "the cycles are reachable from the CLI" is unproven | group H, H4 |
| Fixture capture: real payloads committed; `test/integration/src/fixtures.spec.ts` compares them against the crafted builder | group I |
| Async payment methods | not applicable: `payment_method_types[]=card` is pinned, so a delayed method is unreachable |
| Disputes | group G |
| Anything live-mode | not local-testable by construction |

---

## Two places to run it, and both do the whole good path

| | PocketIC suite | local `icp network` |
|---|---|---|
| Cycles ledger + CMC (rate only) | ✅ real wasms | ✅ seeded |
| XRC | ✅ pinned `xrc_mock` | ✅ the `xrc` canister in `icp.yaml` |
| Fresh CMC rate | ✅ built in | ✅ via the PocketIC API (below) |
| Real Stripe events | ✅ `makeLive()` + `stripe listen` | ✅ `stripe listen` at the gateway |
| Time travel / outage injection | ✅ first-class | ✅ via the PocketIC API |
| **Frontend in a browser** | ❌ no `ic_env` cookie | ✅ **only here** |
| Repeatable, scripted, in CI | ✅ | ❌ manual |

Pick the PocketIC suite for anything you want to keep. Pick a local network when you need
the real UI in a browser. `icp network start` runs PocketIC, and its control API is
reachable on a second port, so a local network has arbitrary sender impersonation and
time control. ⚠️ **That is not a supported `icp` interface**: it depends on the launcher
being PocketIC and on that port being open. The committed suites do not rely on it.

### The whole flow, in order, in a browser

This is **the** procedure. What you have to supply: a Stripe sandbox account with the
Stripe CLI logged in to it, and a restricted API key (`rk_...`) with Checkout Sessions =
Write and everything else None. Write also grants the read the recovery sweep needs,
measured: a `rk_` key scoped to Checkout Sessions = Write returns HTTP 200 on
`GET /v1/checkout/sessions/{id}`. That matters because PocketIC answers the sweep's
outcall itself, so a key that could create sessions but not read them would 401 in
production with every scenario green (`stripe.retrieveUnauthorized`).

```sh
# 0. Prerequisites, once.
brew install stripe/stripe-cli/stripe jq
stripe login                                    # a SANDBOX account, never live
npm --prefix test/integration run fetch:wasm    # the sha256-pinned xrc mock

# 1. A sellable local gateway. Put STRIPE_API_KEY=rk_... in scripts/.local-dev.env
#    (gitignored, sourced by the seed) so it never lands in your shell history.
#    Without it the seed sets a placeholder: everything except paying works.
#    ⚠️ Keep it in that file rather than `export`ing it: the Stripe CLI prefers
#    STRIPE_API_KEY over its own login and a restricted key cannot open a CLI session
#    (403 more_permissions_required). This repo's own scripts wrap every `stripe`
#    invocation in `env -u STRIPE_API_KEY`; a command you type yourself is exposed.
icp network start -d
icp deploy
scripts/local-dev-seed.sh

# 2. Wire Stripe, in its own terminal. It asserts the gateway can price before it
#    starts, and leaves the forwarder running until Ctrl-C.
scripts/stripe-dev.sh
```

⚠️ **The CMC rate has a 15-minute fuse, and it is the single most likely thing to derail
this run.** `Pricing.mo` refuses a CMC rate older than 15 minutes by design, and the
local CMC's rate is stamped 2021 until something sets it. Fifteen minutes after seeding,
`create_order` is refused, and the symptom is not a rate error: `scripts/stripe-dev.sh`
fails its preflight with "the gateway cannot price a $10 purchase". The fix needs no
reinstall:

```sh
scripts/local-dev-seed.sh --rate-only
```

So re-arm immediately before you pay. An order that already exists keeps its locked
quantity forever; a stale rate only refuses new orders, so a run that sits in the browser
for half an hour fails at create, not at pay.

Then, in the browser at the frontend URL `icp deploy` printed
(`http://frontend.local.localhost:8000/` with the default gateway port):

1. **Click "Get cycles".** The form asks nothing about where the cycles go: they go to
   the account of the principal you sign in as, and the gateway refuses any other
   destination.
2. **Sign in.** You get local Internet Identity automatically (`http://id.ai.localhost:8000`,
   deployed by `ii: true` in `icp.yaml`, chosen by `auth.ts` because the origin ends in
   `.localhost`). Register a passkey; it is throwaway and wiped by `icp network stop`.
3. **Pick an amount and create the order.** The rate is locked here, not at payment.
4. **Pay.** "Pay with card ↗" opens the order's own Checkout Session. Card
   `4242 4242 4242 4242`, any future expiry, any CVC. You have 35 minutes, enforced by
   Stripe; the button disappears at the deadline.
5. **Watch the page.** Leave the order tab open. The webhook arrives at the forwarder,
   delivery runs, and the page reaches **delivered** on its own 3 s poll. It should never
   need a reload. If nothing happens, work these three in order; each has produced a
   silent stall in a real run:

   | Check | Command | Wrong looks like |
   |---|---|---|
   | the secret is provisioned | `icp canister call backend webhook_secret_status '()'` | `isSet = false`: every event is dropped unverified, with no audit line and no queue entry |
   | the forwarder is up | `pgrep -fl "stripe listen"` | nothing: Stripe delivered to a closed door, and a resend is delivered once, so resending before the forwarder is up wastes it |
   | the CMC rate is fresh | `icp canister call backend audit_log_recent '(null, 25 : nat)' \| grep rates.refresh` | `rates.refreshFailed: cmc rate is stale or zero`: re-arm with `--rate-only`. A stale rate fails at order creation, not at delivery |

   To replay a payment the canister missed, resend the `checkout.session.completed`
   event, not `charge.updated`, `charge.succeeded` or `payment_intent.succeeded`, none
   of which this canister acts on (a resent event of the wrong type logs
   `stripe.unhandledType` and changes nothing):

   ```sh
   stripe events list --limit 25          # find the checkout.session.completed
   stripe events resend <evt_...>
   ```

   `audit_log` separates a verified-but-unactionable event, a bad signature and an event
   that never arrived, which all look identical from the UI.
6. **Follow the CLI page** at `#/cli`, from the delivered order's next-step link. Five
   numbered steps, and the order is load-bearing:

   ```sh
   # Step 1 is a SETTING, not a command: turn CLI access on at https://id.ai →
   # settings. Without it the next command's sign-in page says "CLI access not
   # enabled" and never returns an identity.
   icp identity link web cyclepay-id --app <host>   # 2 — bare domain, port included
   icp identity default cyclepay-id                 # 3 — what makes the rest act as it
   icp identity principal                           # 4 — must equal the page's principal
   icp cycles balance                               # 4 — must equal the page's figure
   icp deploy -e ic                                 # 5 — the buyer's own project
   ```

   The principal must match. A mismatch means the `--app` value did not match this
   origin. ⚠️ No `--identity` flag on the two verify commands: step 3 made it the
   default, and passing the flag would verify an identity that `icp deploy` will not use.
7. **Prove the cycles exist.**

   ```sh
   icp cycles balance --of-principal <the principal from step 6>  # from any identity
   ```

   This is the end of the flow the product promises, and the one thing no suite in this
   repo proves.

### Every lever the seed script pulls, spelled out

Reference, not a second procedure: `scripts/local-dev-seed.sh` does all of this and
verifies the outcome of each step. Read it when a step fails, or when configuring
something that is not a local network. `icp deploy` leaves a gateway that is fail-closed
on several axes at once: no tiers, no CMC rate, no Stripe secrets, an empty cycles
reserve, and `minCanisterCycles` at 5 T while `icp deploy` creates the canister with
less, so the admission gate refuses every purchase with "temporarily unavailable while
the gateway is topped up". That reads as a reserve problem and is really about the
canister's own gas.

```sh
npm --prefix test/integration run fetch:wasm    # the pinned xrc_mock
icp network start -d && icp deploy              # backend + xrc + frontend

# The admission gate checks the canister's OWN cycles before pricing.
icp canister top-up backend --amount 20t

# Config. The webhook secret is SEALED (§7.3): `set_webhook_secret` takes a ciphertext
# blob. No trailing environment argument here: the default is `local`, which is the
# PocketIC master key this network uses. The 16-byte minimum applies to the decrypted value.
STRIPE_WEBHOOK_SECRET='whsec_local_test_1234567890' scripts/seal-secret.sh webhook-secret
icp canister call backend set_card_tiers \
  '(vec { record { id = "t10"; usdCents = 1_000 : nat } })'
icp canister call backend set_delivery_config \
  '(record { alertAfterNs = 7_200_000_000_000 : int; maxHoldNs = 259_200_000_000_000 : int })'
icp canister call backend set_expected_livemode '(opt false)'

# Fund the reserve: the cycles the gateway will SELL, in its own cycles-ledger
# account. ⚠️ This is not the canister's gas. Nothing creates these cycles.
BACKEND_ID=$(icp canister status backend --json | jq -r '.id')
icp cycles transfer 100t "$BACKEND_ID"
# Then let the gateway observe what arrived. Until it does, the admission gate
# refuses with `#reserveShort`.
icp canister call backend refresh_reserve '()'
icp canister call backend reserve_status '()'   # availableToSell > 0

# Give the CMC a current rate (next section), then:
icp canister call backend refresh_rates '()'
icp canister call backend pricing_status '()' --query   # expect ok = true
```

Create an order and open the `stripeSessionUrl` on the returned order; that is the
payment page. Pay with `4242 4242 4242 4242` and watch `get_order` reach `delivered`.
`process_order` kicks the delivery without waiting for the sweep, and
`pending_deliveries` (admin) shows anything still outstanding. `delivery_journal` and
`receipt` then carry the real cycles-ledger block index and the delivered quantity.

The post-payment redirect lands: the seed points the origin at
`http://frontend.local.localhost:<port>`, which `Session.validateOrigin` accepts because
the host is loopback. A caller-supplied `success_url` remains deliberately impossible; it
would be an open redirect Stripe renders after a real payment.

#### Giving the local CMC a current rate

PocketIC has no API for setting the conversion rate. It solves the problem by pinning the
instance clock to 10 May 2021, the smallest value strictly larger than the timestamp
hard-coded in the CMC state, so the CMC's built-in rate is fresh. A local `icp network`
runs at real wall-clock time, so against the 15-minute staleness guard the rate is
permanently stale and pricing reports `"cmc rate is stale or zero"`. Move the rate
forward (below), which keeps real time so real Stripe signatures verify. Moving the clock
back to 2021 instead trades the CMC problem for the Stripe problem: every real signature
is rejected against the ±300 s tolerance.

Impersonation is a first-class PocketIC feature (the committed suite does the same thing
through `gw.cmcAsGovernance`). The CMC requires a strictly greater timestamp than the one
it holds, so a second call inside the same second fails. The PocketIC port is dynamic
even with a fixed gateway port; discover it:

```sh
PID=$(pgrep -f 'pocket-ic --ttl' | head -1)
lsof -nP -iTCP -sTCP:LISTEN -a -p "$PID" | awk 'NR>1{print $9}' | grep -v ':8000$'
```

```js
// run from test/integration/ so the imports resolve
import { Principal } from '@icp-sdk/core/principal';
import { IDL } from '@icp-sdk/core/candid';

const PIC = 'http://127.0.0.1:<pic-port>/instances/0';
const b64 = (u8) => Buffer.from(u8).toString('base64');
const Arg = IDL.Record({
  data_source: IDL.Text, timestamp_seconds: IDL.Nat64, xdr_permyriad_per_icp: IDL.Nat64,
});
const t = await fetch(`${PIC}/read/get_time`).then((r) => r.json());
const payload = new Uint8Array(IDL.encode([Arg], [{
  data_source: 'local-dev',
  timestamp_seconds: BigInt(Math.floor(Number(t.nanos_since_epoch) / 1e9)),
  xdr_permyriad_per_icp: 35_000n,
}]));
await fetch(`${PIC}/update/submit_ingress_message`, {
  method: 'POST', headers: { 'content-type': 'application/json' },
  body: JSON.stringify({
    sender: b64(Principal.fromText('rrkah-fqaaa-aaaaa-aaaaq-cai').toUint8Array()),
    canister_id: b64(Principal.fromText('rkp4c-7iaaa-aaaaa-aaaca-cai').toUint8Array()),
    method: 'set_icp_xdr_conversion_rate',
    payload: b64(payload),
    effective_principal: { None: null },
  }),
});
```

`/update/set_time`, `/update/set_certified_time` and `/update/tick` on the same instance
give time travel, so the 2 h alert and 72 h terminate are reachable locally too.

### One command: `npm --prefix test/integration run sandbox`

Boots PocketIC with everything, aligns the clock, bootstraps dev config, goes live, prints
the webhook URL, and stays up until Ctrl-C.

```sh
stripe login                                  # a SANDBOX account, never live
STRIPE_WEBHOOK_SECRET="$(stripe listen --print-secret)" \
STRIPE_API_KEY=rk_...                         \
  npm --prefix test/integration run sandbox
# then, second terminal:
stripe listen --forward-to '<the URL it prints>'
```

- Both keys are prefixed onto one command, not exported, so the Stripe CLI in the second
  terminal never sees the restricted key.
- `STRIPE_API_KEY` is what makes an order possible, not just a payment: `create_order`
  itself calls Stripe. Without it the harness boots and says so.
- It lowers `minPurchaseUsdCents` to $1, because its numeric vector is the $5 at-cost
  case; that is a dev value like the 2-minute alert, not a mainnet one.

If you are writing your own spec:

```ts
await pic.setTime(new Date());            // so real Stripe signatures verify (±300 s)
await pic.setCertifiedTime(new Date());
await setXrcRate(gw); await setCmcRate(gw);   // working price inputs
const port = await pic.makeLive();        // real HTTP gateway
```

`src/live-gateway.spec.ts` already does the canister half and logs the exact URL. ⚠️
`makeLive` enables auto-progress, which is incompatible with `advanceTime`, so call
`pic.stopLive()` before any time travel. And the clock alignment is not optional: a
PocketIC instance starts years away from now.

### What still needs somewhere else

| Need | Where | Why |
|---|---|---|
| Completing a **hosted Checkout page** | a browser | Stripe has no headless path. `stripe trigger --override checkout_session:client_reference_id=<ref>` gets a real signed event with the right reference, which covers attribution without the UI |
| **Frontend click-through** (group H) | a **local network**, the walkthrough above | the asset canister serves the `ic_env` cookie the page needs; PocketIC does not |
| Live-mode behaviour: Radar, 3DS, payouts, disputes, account restrictions | mainnet + Stripe **live**, tight caps | unmockable |

Nothing in this plan requires a mainnet deploy. Everything else (signatures, attribution,
amounts, dedup, refunds, event types, livemode, and full delivery) runs in PocketIC.

### Verification commands used throughout

```sh
icp canister call backend audit_log_recent '(null, 25 : nat)'
icp canister call backend orphans_unresolved '(null, 50)'
icp canister call backend get_order '("<orderId>")'
icp canister call backend order_for_payment '("pi_...")'
icp canister call backend delivery_journal '("<orderId>")'
icp canister call backend receipt '("<orderId>")'           # owner identity only
```

---

## A. Signature and transport

| # | Scenario | How | Expect |
|---|---|---|---|
| A1 | Valid signature accepted | any real forwarded event | `200`; event appears in `audit_log` |
| A2 | Tampered body rejected | `stripe listen` + edit the body in a replayed `curl` with the original signature | `400`, nothing in state |
| A3 | Missing signature header | `curl` the route with no `Stripe-Signature` | `400` |
| A4 | **Unprovisioned secret → Stripe retries and later succeeds** | deploy fresh, do not set the secret, pay; then set the secret and wait for Stripe's retry | first delivery `503`; the retry delivers |
| A5 | Secret rotation overlap | `set_webhook_secret` with a new value while a delivery is in flight | no lost event; `webhook_secret_status.generation` increments |
| A6 | Clock drift rejected | skew the host clock >5 min, deliver | `400`. Restore the clock afterwards |

## B. Attribution (claimed, not trusted)

| # | Scenario | How | Expect |
|---|---|---|---|
| B1 | Happy path | open the order's `stripeSessionUrl`, card `4242 4242 4242 4242` | order → `#paid` → `#delivered` |
| B2 | No reference | `stripe trigger checkout.session.completed` | `200`; `#unattributed`, `claimedRef` empty |
| B3 | Forged owner | hand-edit the ref to another principal, same order id | `#unattributed`: "claimed owner does not match" |
| B4 | Malformed reference | ref = `garbage` | `#unattributed`: "malformed" |
| B5 | Payment for an **expired** order | open the order's session URL, expire that session in the Stripe Dashboard so `checkout.session.expired` arrives, then pay a previously opened copy of the page | `200`; order stays `Expired`, `#unattributed` whose detail says "cannot be paid". Refund it in Stripe |
| B6 | Payment for a **cancelled** order | `cancel_order`, then pay a page you opened before cancelling | `200`; order stays `Cancelled`, the same refundable obligation. ⚠️ Hard to reach on purpose: cancel expires the session on Stripe first |
| B7 | There is no rescue lever | — | for B5 and B6 the only remedy is a refund in Stripe, which auto-resolves the entry |

## C. Amount honouring

The canister sets the amount on the session, so C2, C3 and C5 cannot be produced through
the app. To exercise the mismatch branch, create a session outside the app (a hand-made
`POST /v1/checkout/sessions` at a different `unit_amount`, carrying an order's
`client_reference_id`): worth doing once, because it is the branch that used to deliver
silently. `test/integration/src/fixtures.spec.ts` compares a recorded real session
against the crafted builder on `amount_total`, `currency`, `payment_status` and
`payment_intent`, so the shape question is automated; C1 and C4 are the required rows.

| # | Scenario | How | Expect |
|---|---|---|---|
| C1 | Exact quoted amount | pay the order's own session | `lockedCycles` verbatim; `paidUsdCents == pricing.usdCents` |
| C2 | **Different amount** | hand-make a session at another `unit_amount` with the order's reference | `200`; nothing delivered, order stays `Created`, a refundable obligation naming both figures |
| C3 | Below the fee floor | same, at e.g. $0.31 | the same obligation as C2; not a separate outcome |
| C4 | Above the per-purchase ceiling | lower `maxPurchaseUsdCents` below an existing order's amount, then pay that order's session | a refundable obligation, nothing delivered. The ceiling's one reachable case, and it needs no tampering |
| C5 | Wrong currency | hand-make a EUR session | `#unattributed`: "unexpected currency" |

## D. Dedup and replay

| # | Scenario | How | Expect |
|---|---|---|---|
| D1 | Resend one event | `stripe events resend <evt_...>`. ⚠️ Not the Dashboard's own "Resend": that acts on a delivery attempt to a registered endpoint, and this plan's transport is `stripe listen`, a live subscription with no endpoint | `200 duplicate event`; no second credit |
| D2 | Two genuine payments | pay the same order twice (two intents) | second → `#duplicate` |
| D3 | Same intent, new event id | resend after >7 days if you can arrange it, else trust D1 | `200 already credited`, `stripe.replayedAfterPruning` |
| D4 | Credited elsewhere | deliver a hand-made `completed` for order Y carrying an intent already credited to order X | nothing delivered; `stripe.creditedElsewhere` + a `#duplicate` naming both |

## E. Refunds — the highest-value group

Use real Stripe refunds, not crafted events: the point is confirming Stripe's
`amount`/`amount_refunded` semantics match what the code assumes.

| # | Scenario | How | Expect |
|---|---|---|---|
| E1 | Full refund of an unattributed payment | B2, then refund it fully in the Dashboard | the entry auto-resolves |
| E2 | **Partial refund** | refund e.g. $1 of a $5 charge | entry stays open; `stripe.refundPartial`; no `refundUnmatched` line |
| E3 | Partial then completed | refund the remaining $4 | entry now resolves |
| E4 | Two partials summing to full | $2 then $3 | resolves on the second: `amount_refunded` is cumulative |
| E5 | Refund **after delivery** | deliver an order, then refund | `#refundAfterDelivery` with `refundedCents` and `fullRefund` set; never auto-resolves |
| E6 | Refund of an escalated order | force an escalation, then refund | `stripe.refundOfEscalated` |

## F. Async / delayed payment methods

Not reachable while `payment_method_types[]=card` is pinned. If the pin is ever removed,
verify Stripe really sends `checkout.session.completed` with `payment_status != "paid"`
followed by `checkout.session.async_payment_succeeded`:

| # | Scenario | How | Expect |
|---|---|---|---|
| F1 | Delayed method settles | enable a delayed method on a test session and pay | first event `200 ignored`, order stays `#created`, `stripe.unpaidSession`; on settlement → `#paid` |
| F2 | Delayed method fails | trigger `async_payment_failed` | order stays payable; intent not consumed |
| F3 | Out-of-order arrival | force settlement before `completed` | still delivers exactly once |

## G. Event types and configuration

| # | Scenario | How | Expect |
|---|---|---|---|
| G1 | Unhandled type | subscribe `payment_intent.succeeded`, trigger it | `200 ignored` + `stripe.unhandledType`. Never 4xx. Needs a type the dispatcher genuinely does not know; `charge.dispute.created` is handled |
| G2 | **No `payment_intent`** | a 100%-off promo code, or a subscription-mode session | `200` + `#unprocessable`; a resend does not duplicate it |
| G3 | Livemode mismatch | point a test secret at a canister set to `opt true` | nothing delivered; `stripe.livemodeMismatch`; no obligation filed |
| G4 | Live-on-test | the reverse | nothing delivered, but an obligation is filed, keeping the real reference |
| G5 | Mode unset | `set_expected_livemode '(null)'`, pay | delivers, plus `stripe.livemodeUnset` on every payment |

## H. Frontend — only what a machine cannot do

Most of this group is automated: `main.test.ts` runs a jsdom suite against the real
`index.html` body, and `test/browser/` drives a production build in Chromium with
committed screenshot baselines (tier estimates, the fee disclosure, the absent
destination question, the quote-moved confirmation, cancel visibility, the receipt, the
delivered view, the CLI page's five commands, and paint). Do not repeat those by hand.
Still needs a human:

| # | Scenario | Expect | 2026-08-13 |
|---|---|---|---|
| H1 | **Real sign-in**, local Internet Identity at `http://id.ai.localhost:8000` | a passkey registers and the header shows a shortened principal | ✅ |
| H2 | **The deployed asset canister**, not a static build | the page reads its backend id and root key from the real `ic_env` cookie and prices from the real canister | ✅ |
| H3 | **The real Stripe hosted Checkout page** | Stripe has no headless path | ✅ |
| H4 | **The CLI page's commands actually work** | five steps at `#/cli`. Run them in order and stop at the first that does not do what the page says (step 6 of the walkthrough above) | ❌ **not run** |
| H5 | **The cycles are really there** | `icp cycles balance --of-principal <that principal>` shows the delivered quantity | ✅ 18.2 T for two orders |
| H6 | Order history across a real sign-out and sign-in | the table repopulates; a reopened order still shows its timeline | not run |
| H7 | Typography, hierarchy, and the italic rule | `brand-lint.sh` covers banned characters, vocabulary and hardcoded colour. The rest needs eyes | not run |

**H4 is the one that still matters most.** H5 proves the cycles exist at the buyer's
principal; H4 proves a buyer can become that principal from the CLI and spend them. Two
of its steps are unverifiable by any suite: the id.ai CLI-access switch is a setting in
someone else's product, and the delegation the link command returns needs a real browser
sign-in.

## I. Fixture capture — do this while you are in there

Save the raw request bodies (Dashboard → event → the JSON) and commit them as integration
fixtures (`scripts/capture-stripe-fixtures.sh`; `--status` lists what is missing):

- `checkout.session.completed`, paid, with a reference
- `checkout.session.completed`, `payment_status: unpaid`
- `checkout.session.async_payment_succeeded`
- `charge.refunded`, full and partial
- a `payment_intent: null` session (G2)
- `charge.dispute.created`

---

## How this relates to `RUNBOOK.md`

This plan asks *does this build behave correctly?*, once before the rail carries money
and again after material changes. `RUNBOOK.md` asks *how do I operate a live system?*
Where this plan states an expected outcome (a filed obligation, a `#unprocessable`, a
`stripe.refundPartial`), `RUNBOOK.md` section 6 states what to do about it, so a run is
also a check on the runbook: a scenario whose triage row is missing or unfollowable means
the runbook is the thing to fix. The plan is a precondition to `docs/OPERATE.md`'s Mode 3.

`scripts/stripe-dev.sh` sets only the two Stripe-side values (`expected_livemode =
false` and the forwarding session's signing secret); `scripts/local-dev-seed.sh` owns the
money levers and sets the mainnet delivery timeline (2 h alert, 72 h max hold), so on a
local network the delay path needs a hand-called `set_delivery_config`. Only the PocketIC
harness shortens it, to a 2-minute alert.

## What this cannot tell you

A green run closes the biggest unknown, not the remaining known gaps:

1. **Sandbox ≠ live.** No real Radar rules, 3DS challenges, payout mechanics or
   account-restriction behaviour. Only a mainnet deploy in live mode shows those.
2. **Disputes produce only an audit line.** A lost chargeback cannot be reversed
   on-chain; it is managed in the Dashboard.
3. **SEV-SNP.** Checkpoint-to-disk and state-sync are confidential on the target subnet;
   attestation coverage of every replica is still open (`RUNBOOK.md` section 9).
4. **Flat controller allowlist.** Any one controller can upgrade-then-drain.
5. **No monitoring exists.** `RUNBOOK.md` section 8 owns the plan. Wire it before taking
   money.
6. **No external security audit.**

Groups A and B green means the Stripe integration works as coded, and money-out works
against real NNS canisters. That is the engineering bar. Production also needs items 3
and 5 resolved, plus a deliberate decision to accept 1, 2, 4 and 6.

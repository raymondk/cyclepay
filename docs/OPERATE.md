# Operating the gateway

Setup, in every mode it runs in. `RUNBOOK.md` is the other half: day-2 operations,
entered by symptom, once something is already running.

## The three modes, and the fourth cell

Two settings decide the mode, and they are independent:

| `expected_livemode` | `divisor` | mode | what bounds a giveaway |
|---|---|---|---|
| `?false` | 1 | **local plain**, what `scripts/local-dev-seed.sh` sets | nothing needed; the cycles are local |
| `?false` | >1 | **mainnet simulation** | the buyer allow-list |
| `?true` | 1 | **mainnet production** | real money |
| `?true` | >1 | **refused by design**: `#simulationDivisorSet` / `#divisorNeedsSandbox` | — |

⚠️ **Row 1 is representable on MAINNET.** `?false` with `divisor = 1` on mainnet means
free Stripe-sandbox payments delivering full-value cycles out of a funded reserve. The
`#unboundedGiveaway` guard keys on *accepting test payments + an empty allow-list + a
funded reserve* and never looks at the divisor, so the only thing between that
configuration and giving real cycles away is who is on the allow-list.

⚠️ **The divisor must be set BEFORE the first order.** `set_pricing_config` refuses a
divisor change once any order is stored (`#divisorChangeWithOrders`), because every
earlier receipt would recompute against the new value. The only way back is
`icp deploy --mode reinstall`.

**Reinstall is available on mainnet too; what makes it unacceptable is real orders, not
the network.** On a simulation gateway the stored orders are test orders, so
reinstalling to `divisor = 1` is the ordinary route to production, and the canister id
does not change:

| survives a reinstall | because |
|---|---|
| the Stripe webhook URL | it names the canister id |
| both sealed secrets' **ciphertexts** | the seal derives from the master key, the canister id and a fixed context, so re-send the same blobs |
| the reserve's cycles | the account belongs to the canister's principal and lives on the cycles ledger |

What has to be redone: `refresh_reserve` (the floor resets to 0 while the stock is
intact), the buyer allow-list, and re-sending the two sealed blobs. Orders, receipts, the
audit log, the journals and the dedup sets are gone.

## Prerequisites

- Node.js ≥ 22
- `mops`: `npm i -g ic-mops` (the Motoko compiler version is pinned in
  `mops.toml [toolchain]`)
- `icp` CLI: `npm i -g @icp-sdk/icp-cli @icp-sdk/ic-wasm`. This project uses `icp-cli`,
  never `dfx`.

⚠️ **Clone with submodules.** The backend decrypts its sealed secrets using a BLS12-381
implementation pinned as a git submodule, resolved by `mops` as a path dependency, so
without it nothing compiles and the error names a missing package:

```sh
git clone --recurse-submodules https://github.com/marc0olo/cyclepay
git submodule update --init --recursive     # if already cloned
```

That code is experimental and unaudited; `docs/DESIGN.md` §7.3 explains what it is
trusted with and the deletion criterion.

## Mode 1 — local

### Run the app locally, from nothing

Steps 1–4 need nothing from Stripe; 5–6 are for clicking through a real payment.

```sh
# 1. dependencies and a local replica
git submodule update --init --recursive   # the pinned crypto, first time only
mops install
icp network start -d                    # PocketIC, gateway on :8000

# 2. deploy (backend, frontend, local XRC mock)
icp deploy

# 3. make the gateway sellable — NOT optional, see below
scripts/local-dev-seed.sh

# 4. allow-list yourself as a buyer
#    Open http://frontend.local.localhost:8000/ , sign in with Internet Identity,
#    copy the principal the page shows, then:
icp canister call backend add_allowed_buyer '(principal "<your-principal>")'

# 5. one-time: your restricted Stripe key (see below for the required scope)
cat > scripts/.local-dev.env <<'ENV'
STRIPE_API_KEY=rk_test_...
ENV
scripts/local-dev-seed.sh               # re-run: it provisions the key

# 6. in a SECOND terminal: webhook secret + forwarder, and leave it running
scripts/stripe-dev.sh
```

Then buy: pick an amount, pay with `4242 4242 4242 4242`, and the order walks
`created → paid → delivered`.

**Which script owns what**, because re-running the wrong one fixes nothing:

| | owns | after a `--mode reinstall` |
|---|---|---|
| `local-dev-seed.sh` | tiers, the cycles reserve, the CMC + XRC rates, the delivery timeline, the canister's own gas, the Stripe **API key** | re-run it |
| step 4 | the buyer allow-list | redo it |
| `stripe-dev.sh` | expected livemode, the **webhook signing secret**, forwarding | re-run it |

- ⚠️ **Run the seed before `stripe-dev.sh`.** The latter refuses to start if the gateway
  cannot price.
- **No browser where the CLI runs?** `docs/STRIPE.md` §15 has the pairing-code login
  and the Linux install; the checkout URL itself can be paid from any device.
- ⚠️ **Step 4 is the one nobody guesses.** With an empty allow-list every purchase
  refuses with `unboundedGiveaway`, the faucet guard. The seed prints the exact command.
- ⚠️ **Step 5 needs a restricted key** (`rk_...`) with **Checkout Sessions = Write** and
  everything else None. Write is the level that also grants the read the recovery sweep
  needs.
- ⚠️ **Do NOT `export STRIPE_API_KEY`** into a shell where you run the Stripe CLI. The CLI
  prefers it over your `stripe login` credential, and a restricted key cannot open a CLI
  session (`more_permissions_required`). Our scripts strip the variable before calling
  the CLI; a command you type yourself is not protected.

#### Why the seed is not optional

A freshly deployed gateway is fail-closed on five axes at once, which looks like a
broken app rather than a safe one:

| What you see | Why |
|---|---|
| "No amounts are configured yet" | No presets registered. Not a paused rail; the seed registers the tiles |
| "No exchange rate available yet" | Pricing needs the CMC rate, which only NNS governance can set; the seed reaches it through the local PocketIC control API |
| "temporarily unavailable while the gateway is topped up" | `minCanisterCycles` is 5 T and `icp deploy` creates the canister with less. This is the canister's own **gas**, not the cycles it sells; the seed tops up rather than lowering the floor |
| Orders paid but never delivered | The **cycles reserve** is empty |
| `unboundedGiveaway` on every purchase | The buyer allow-list is empty (step 4) |

The seed verifies a $10 purchase is admitted before reporting success, and queries the
CMC for what it actually stored rather than trusting the `Ok` reply.

⚠️ **The CMC rate goes stale in 15 minutes**, and a stale rate shows as `cycles = null`
in `quote_previews`, at any divisor. `scripts/local-dev-seed.sh --rate-only` re-arms it
without redoing the rest.

If the local identity runs out of cycles (`Insufficient cycles` from a top-up):

```sh
icp cycles mint --icp 5                 # ~17.5 T; ICP is pre-seeded on local principals
icp network stop && icp network start -d   # worst case: reseeds principals, wipes state
```

### Verify the deployment wiring

```sh
scripts/e2e-local.sh                      # 20 checks against a real local network
```

Covers what the canister suites structurally cannot: the deploy pipeline, the
`PUBLIC_CANISTER_ID:xrc` override, the `ic_env` cookie, local Internet Identity, a
signed webhook through the real gateway, and that `icp.yaml`'s `ic` environment excludes
the local-only mock.

### Backend iteration loop

```sh
mops check -- -Werror        # typecheck + lint; -Werror makes M0145 a build failure
mops build                   # compile, and regenerate the committed .did
mops test                    # the Motoko unit suites
```

The three IC brand typefaces are vendored into the bundle rather than linked from Google
Fonts: a page that takes card details should not send every visitor's IP to a third
party. `scripts/fetch-fonts.sh` re-vendors them.

### Frontend iteration with hot reload

Needs the network up and the backend deployed: the Vite dev server shells out to `icp`
to simulate the `ic_env` cookie the asset canister sets in production.

```sh
icp network start -d && icp deploy
npm --prefix src/frontend run dev
```

TypeScript bindings are regenerated from the committed `src/backend/dist/backend.did` by
the `icpBindgen` Vite plugin on every dev/build run. Change the backend API, run
`mops build`, and the frontend picks it up or fails to typecheck.

### After a change to the stable shape

A new field on `Order`, a removed variant tag, a changed config record: the upgrade
traps in `register_stable_type` because enhanced orthogonal persistence refuses to
reinterpret the existing memory. Locally that is a two-command problem:

```sh
icp deploy --mode reinstall --yes
./scripts/local-dev-seed.sh
```

`scripts/e2e-local.sh` detects the trap and does this for you. Do not add a mops
migration file to avoid it: the app holds no data anyone needs, and every migration
replays forever on a fresh install. The chain is a go-live prerequisite (Mode 3, "What
is not a command", item 1).

⚠️ **A reinstall also wipes the Stripe webhook secret, and `local-dev-seed.sh` does not
put it back**; only `scripts/stripe-dev.sh` does. An unprovisioned secret makes the
canister drop every event it cannot verify, so a real payment produces no order
movement, no obligation and no audit line. Check it in one call:

```sh
icp canister call backend webhook_secret_status '()'    # isSet must be true
```

### Stopping

```sh
icp network stop
```

## Mode 2 — mainnet simulation

⚠️ **This is not Mode 3 with different values.** This deploys a gateway that takes real
charges in Stripe's sandbox and delivers a divisor'd fraction of the cycles. `divisor > 1`
requires `expected_livemode == ?false` exactly, and the two guards are mutual (see "The
simulation arithmetic" below).

Target of this procedure: `https://cyclepay.raymondk.co`, on the confidential subnet
`re2t4-faa75-v3vhk-kdmdr-uyrkl-aik2l-ixd6u-p3fyr-zlfkc-6c5af-zae`, divisor 1000.

### What needs no configuration

| | why |
|---|---|
| the Exchange Rate Canister | `icp.yaml`'s `ic` environment lists `[backend, frontend]` only, so the local XRC mock is never created on mainnet and the backend falls back to the real XRC. Verify after a rate call: `pricing_status.xrcCanisterId` must read `uf6dk-hyaaa-aaaaq-qaaaq-cai` |
| the Cycles Minting Canister | `rkp4c-7iaaa-aaaaa-aaaca-cai` is compiled in and is the same principal on mainnet and PocketIC |
| a webhook forwarder | `scripts/stripe-dev.sh` exists only because Stripe cannot reach localhost |
| the CSP | `connect-src` already admits `https://icp-api.io`, which is where a custom-domain page sends its canister calls |

### The order is one-way in four places

```
expected_livemode = ?false        <- ?false EXACTLY; null is the fresh-install default and is refused
        |
set_pricing_config divisor=1000   <- refused once ANY order is stored. Reinstall is the only way back
        |
add_allowed_buyer <tester>        <- MUST precede funding the reserve
        |
icp cycles transfer -> refresh_reserve
```

And the fifth, outside the canister: **the derivation origin decides who every buyer
is.** It is pinned to the frontend canister's own origin (`src/frontend/src/config.ts`),
which is what makes this test domain and whatever production domain is chosen yield the
same principals. Changing that pin after the first sign-in strands every account.

### Cycles

```bash
# ⚠️ `-n ic` on BOTH. Without it these act on the local network. Exactly one of
# --icp / --cycles is required.
icp cycles mint --icp 5 -n ic     # or --cycles 8600b, which solves for the ICP needed
icp cycles balance -n ic
```

Budget, every row cash to mint. ⚠️ **The creation fee is a flat 500 B per canister and
does not scale with subnet size**; the deployment consumed 505.9 B each.

```
  4.000 T   icp deploy, two canisters at the 2 T default
              of which creation fees         1.012 T   consumed
              of which lands as balance      2.988 T   1.494 T per canister
+ 3.506 T   top the backend up to its own-gas floor
              5 T floor minus the 1.494 T it actually holds. COMPUTE THIS, see step 1
+ 0.200 T   the sellable reserve, a SEPARATE pot on the cycles ledger
              at divisor 1000 a $10 purchase locks ~7.24 G, so ~27 test purchases
+ 1.000 T   slack for one reinstall, because the divisor is one-way once an order exists
  ────────
  8.706 T   to mint
```

| threshold | value | what happens below it |
|---|---|---|
| `Gate.Config.minCanisterCycles` | 5 T | the gate admits no orders. Compared against the raw `Cycles.balance()`, so the freezing threshold does not eat into it |
| the reserve floor | anything | `#reserveShort`, and it stays 0 until `refresh_reserve` observes a top-up |

⚠️ **The shortfall is invisible until the first order**: the canisters get created, the
frontend serves, and every `create_order` is refused because the backend sits under its
own-gas floor. Check `icp cycles balance -n ic` before starting.

⚠️ **`pricing_status` reads `xrcCanisterId = null`, `rates = null`, `lastAttempt = null`
on a fresh deploy, and all three are correct.** `xrcCanisterId` is null until an XRC call
has resolved it, deliberately: defaulting it to the mainnet id would make the one signal
that detects a mainnet deploy wrongly pointed at a mock read all-clear. No XRC call has
happened because the rate timer returns early while no rail is selling, and the rail is
not live until both Stripe secrets and the return origin are set. `refresh_rates`
bypasses that guard, so force a tick to verify early:

```bash
icp canister call backend refresh_rates '()' -e ic --identity <operator>
icp canister call backend pricing_status '()' -e ic   # xrcCanisterId = uf6dk-hyaaa-aaaaq-qaaaq-cai
```

`xrcCanisterId` is transient: it is null again after every upgrade until the next rate
call. A null reading after a redeploy means "not yet asked", never "misconfigured".

### 1. Create and install on the confidential subnet

```bash
# From a green main. The -Werror gate runs on `mops check`, not on `icp deploy` (see Mode 3).
bash scripts/test-all.sh

icp deploy -e ic \
  --subnet re2t4-faa75-v3vhk-kdmdr-uyrkl-aik2l-ixd6u-p3fyr-zlfkc-6c5af-zae

icp canister status backend -e ic -i     # note both ids
icp canister status frontend -e ic -i

# The backend's own gas, to the floor. This is NOT the reserve.
# ⚠️ Read the balance and compute the difference; do not paste a figure. A pasted
# `--amount 3400b` landed at 4.894 T against a 5 T floor, refusing every order.
icp canister status backend -e ic | grep -i cycles      # e.g. 1_494_093_400_599
#   top-up = 5_000_000_000_000 - that, rounded up. For the figure above: 3506b.
icp canister top-up backend --amount 3506b -e ic
icp canister status backend -e ic | grep -i cycles      # must now read >= 5_000_000_000_000
```

⚠️ **`--subnet` on the deploy, not on a later create.** A canister already created on the
default application subnet cannot be moved; on mainnet, starting over means new canister
ids, a new derivation origin and a new principal for anyone who signed in.

Raise the freezing threshold once the deployment is real; 30 days is thin for
money-bearing state:

```bash
icp canister settings update backend --freezing-threshold 7776000 -e ic   # 90 days
```

### 2. The custom domain

`src/frontend/public/.well-known/ic-domains` and `ii-alternative-origins` ship in the
frontend already. What remains is DNS and the registration call.

| record | host | value |
|---|---|---|
| CNAME | `cyclepay.raymondk.co` | `cyclepay.raymondk.co.icp1.io` |
| TXT | `_canister-id.cyclepay.raymondk.co` | the **frontend** canister id |
| CNAME | `_acme-challenge.cyclepay.raymondk.co` | `_acme-challenge.cyclepay.raymondk.co.icp2.io` |

⚠️ **Turn off the DNS provider's own TLS.** Cloudflare's Universal SSL and equivalents
interfere with the ACME challenge the boundary nodes run, and can leave stale
`_acme-challenge` TXT records that do not appear in the dashboard. Check with
`dig TXT _acme-challenge.cyclepay.raymondk.co`: there should be only the CNAME.

```bash
curl -sL "https://icp.net/custom-domains/v1/cyclepay.raymondk.co/validate" | jq
curl -sL -X POST "https://icp.net/custom-domains/v1/cyclepay.raymondk.co" | jq
curl -sL "https://icp.net/custom-domains/v1/cyclepay.raymondk.co" | jq '.data.registration_status'
```

Poll until `registered`, then give the gateways a few minutes.

⚠️ **`.well-known/ii-alternative-origins` is what makes the domain usable at all.**
Internet Identity fetches it from the derivation origin and refuses to derive for a
serving origin the file does not list. Adding a second serving domain means editing that
file and redeploying before the domain goes live.

### 3. Declare the mode, then the divisor

```bash
# ?false EXACTLY. `null` accepts either mode and is what a fresh install has.
icp canister call backend set_expected_livemode '(opt false)' -e ic --identity <operator>
icp canister call backend expected_livemode '()' -e ic

# Simulation. MUST come before the first order.
icp canister call backend set_pricing_config '(record {
  feeBps = 290 : nat; feeFixedCents = 30 : nat; maxAgeNs = 300_000_000_000 : int;
  maxRateDeltaBps = 5_000 : nat; minRateSources = 2 : nat; divisor = 1_000 : nat })' \
  -e ic --identity <operator>

icp canister call backend pricing_status '()' -e ic   # divisor = 1_000, xrcCanisterId = uf6dk-...
```

### 4. The secrets, sealed, without any local script

Both Stripe secrets are encrypted to the canister before they are sent, so the plaintext
never appears in an ingress message, a shell history or a CI log. The trailing `ic`
selects the **mainnet vetKD master key**: mainnet and a local network both call their
key `key_1`, and sealing against the wrong one produces a ciphertext nobody can open
(`#notSealedToThisCanister`).

Each `read` waits for you to paste the value:

```bash
# `read -rs`: -s so nothing echoes, -r so a backslash stays a backslash. The printf is
# the prompt. Typed into `read`, the only thing shell history records is the command.
# ⚠️ `read -rsp` is the bash idiom and FAILS in zsh (-p means "read from the coprocess").
printf 'Stripe restricted key (rk_...): '; read -rs STRIPE_API_KEY
printf '\n%s chars captured\n' "${#STRIPE_API_KEY}"
export STRIPE_API_KEY
scripts/seal-secret.sh api-key ic

printf 'Stripe webhook signing secret (whsec_...): '; read -rs STRIPE_WEBHOOK_SECRET
printf '\n%s chars captured\n' "${#STRIPE_WEBHOOK_SECRET}"
export STRIPE_WEBHOOK_SECRET
scripts/seal-secret.sh webhook-secret ic

# ⚠️ Not optional. An exported key stays readable by every later child process of this
# shell, including the Stripe CLI, which prefers STRIPE_API_KEY over its own session.
unset STRIPE_API_KEY STRIPE_WEBHOOK_SECRET
```

- The length echo is the confirmation: `read -rs` shows nothing as you type, so check
  the number against the key you hold before running the seal.
- `scripts/.local-dev.env` is read **only for a local environment**. The script refuses a
  set-but-empty value outright and never reads the file for a named environment, so an
  empty paste cannot seal the sandbox key to the mainnet canister.
- ⚠️ **Use a SANDBOX restricted key (`rk_test_...`), Checkout Sessions = Write, everything
  else None.** Never an unrestricted `sk_`.

By hand, if you want each step visible:

```bash
BACKEND=$(icp canister status backend -e ic -i)

# 1. Seal, OFFLINE. The public key is computed from a master key shipped in
#    @icp-sdk/vetkeys plus the canister id: no network call, no identity.
CYCLEPAY_SEAL_SECRET="$STRIPE_API_KEY" npm --prefix scripts/seal run --silent seal -- \
  --canister "$BACKEND" --source mainnet --out /tmp/sealed.arg

# 2. Send it. This is the only step that involves your identity.
icp canister call backend set_stripe_api_key --args-file /tmp/sealed.arg -e ic
rm -f /tmp/sealed.arg

icp canister call backend stripe_api_key_status '()' -e ic   # isSet = true, generation = 1
```

`--source mainnet` has no default, on purpose.

### 5. Stripe: where events arrive, and where the buyer comes back

```bash
# Where the BUYER returns after paying. Validated: https, no query, no fragment.
# success_url becomes `<origin>/#/paid/<id>` (cancel_url `#/unpaid/<id>`), and the app
# routes on the hash.

icp canister call backend set_stripe_origin '("https://cyclepay.raymondk.co")' \
  -e ic --identity <operator>
icp canister call backend stripe_origin '()' -e ic
```

In the Stripe **sandbox** dashboard, add a webhook destination pointing at the **backend
canister**, not the domain:

```
https://<backend-canister-id>.icp.net/webhook/stripe
```

| event | what it does here | omitting it |
|---|---|---|
| `checkout.session.completed` | the delivery path | nothing is ever delivered |
| **`checkout.session.expired`** | the only event that moves an order to `#expired` and releases its reserve promise | ⚠️ orders sit `#created` past their deadline forever, holding reserve capacity. `expire_order` does not recover it: it asks Stripe first and refuses with `#sessionNotOpen` once the session has really expired there. Subscribe, then resend the event from the Stripe Dashboard |
| `charge.refunded` | resolves an `#unattributed` obligation, and files one for a refund after delivery | a refund settles the money and leaves the worklist item open |
| `charge.dispute.created` | one audit line: reconcile in Stripe; cycles cannot be recovered | the dispute leaves no trace in the trail |
| `checkout.session.async_payment_succeeded` | the delivery path, for a delayed method | cannot fire today (`createBody` pins `payment_method_types[]=card`). Subscribe anyway: if that pin is ever removed, an unsubscribed success is fiat in with nothing delivered |
| `checkout.session.async_payment_failed` | one audit line | same reason to subscribe |

⚠️ **Subscribe to all six.** The signing secret that destination shows you is the one
step 4 provisions. Until it is set the route answers 503 and Stripe retries.

### 6. Allow-list, then fund the reserve

⚠️ **This order.** Sandbox payments are free and unlimited, so test mode plus an empty
allow-list plus a funded reserve is a cycles faucet, and the gateway refuses every buyer
in that state (`#unboundedGiveaway`).

```bash
# Sign in at https://cyclepay.raymondk.co, copy the principal the page shows.
# ⚠️ A principal copied from any OTHER deployment of this app is a different one.
icp canister call backend add_allowed_buyer '(principal "<tester>")' -e ic --identity <operator>
icp canister call backend allowed_buyers '()' -e ic

icp cycles transfer 200b <backend-principal> -n ic
icp canister call backend refresh_reserve '()' -e ic --identity <operator>
icp canister call backend reserve_status '()' -e ic     # availableToSell > 0
```

⚠️ **`refresh_reserve` is not optional and its absence is silent**: the ledger holds the
cycles, `availableToSell` stays 0, and every buyer is refused with `#reserveShort`.

### 6a. The price tiles

```bash
icp canister call backend set_card_tiers \
  '(vec { record { id = "t10"; usdCents = 1_000 : nat };
          record { id = "t20"; usdCents = 2_000 : nat };
          record { id = "t50"; usdCents = 5_000 : nat } })' \
  -e ic --identity <operator>
icp canister call backend card_tiers '()' -e ic          # three tiles
```

⚠️ **With an empty list the buy page offers no way to buy** (Mode 3, step 4). Every tier
must sit inside the gate's bounds or the whole vector is refused. No `$100` preset: that
is the ceiling, and it is what the custom field is for.

### 7. Verify before spending a card

```bash
icp canister call backend health '()' -e ic                       # true
icp canister call backend pricing_status '()' -e ic               # ok, divisor 1_000, real XRC id
icp canister call backend quote_previews '(vec { 1_000 : nat })' -e ic
icp canister call backend refusal_counts '()' -e ic               # every refusingNow flag false
icp canister call backend expected_livemode '()' -e ic            # opt false
curl -sI https://cyclepay.raymondk.co                             # HTTP/2 200
curl -sL https://cyclepay.raymondk.co/.well-known/ic-domains
```

⚠️ **Verify identity too; it is the only one-way item in this list.** Two checks, before
step 6 allow-lists a principal and funds a reserve against it:

```bash
# 1. Internet Identity reads this CROSS-ORIGIN from the derivation origin. Without the
#    CORS header it refuses to derive, which presents as sign-in failing only on the
#    custom domain.
curl -sI https://<frontend-canister-id>.icp.net/.well-known/ii-alternative-origins
#    expect: content-type: application/json  AND  access-control-allow-origin: *
```

2. Sign in at `https://cyclepay.raymondk.co` and at
   `https://<frontend-canister-id>.icp.net`, and confirm the page shows the same
   principal. If they differ, stop: anything allow-listed from here is allow-listed for
   an identity that will not come back.

Two readings that are correct and look wrong: `availableToSell` stays in real cycles
while quotes are scaled, so it can read 200 G while $10 buys 7.24 G; and a stale rate
shows as `cycles = null` in `quote_previews` at any divisor, so read `pricing_status` and
force a tick with `refresh_rates` before concluding the divisor is wrong.

### The simulation arithmetic

**One number is the whole switch: `pricing_status().config.divisor`.** `1` is production;
anything greater is simulation, and the mode signal, the banner and the receipt's extra
terms all key off that value. There is deliberately no second boolean.

| what | where | why there |
|---|---|---|
| the scale | `Pricing.quote` | the single derivation of a cycle quantity, so the quote, `lockedCycles`, the promise tally, the floor decrement and the transfer are all the same scaled number |
| who may buy | the buyer allow-list | the only bound on the **total** given away; the divisor bounds only the per-order loss |
| the ceiling | `Pricing.quote` again | the cycles-ledger deposit fee is flat, so an over-scaled order cannot clear it |

⚠️ **Do NOT scale at delivery.** The reserve floor decrements at issue by the full locked
amount (§5.4 rule 2), so a scaled transfer against an unscaled decrement would surface
as an unexplained shortfall on every reconcile.

The Stripe fee is taken before the divisor and the ledger fee after it, because each
third party really charges what it charges. Where the division sits does not matter for
correctness (`floor(floor(a/b)/d) == floor(a/(b*d))` for positive integers), so it is
one division after the rate conversion, and `checkReceipt` divides at the same point.

**The four guards:**

| guard | prevents |
|---|---|
| `divisor > 1` requires `expected_livemode == ?false` **exactly** | real money in, scaled cycles out. `null` means *either mode*, accepts live payments, and is the default |
| `set_expected_livemode` refuses anything but `?false` while `divisor > 1` | the same state reached from the other direction |
| a divisor **change** is refused while any order is stored | earlier receipts recomputing against the new value. Reinstall to change it |
| a scaled quote must clear the ledger fee **ten times over** | the one delivery state with no recovery lever: a fee above a whole order's locked quantity means no `#BadFee` ever arrives to correct the stored copy |

⚠️ **The divisor's ceiling scales with `minPurchaseUsdCents`, not with the amount being
bought.** At the shipped $10 floor a divisor of 1,000 leaves 7.24 G (72x the 100 M fee);
at a $1 floor the same divisor leaves 515 M and is refused
(`#simulationScaleTooSmall`). An operator who lowers the floor for a demo and then cannot
set the divisor is seeing the guard work. That refusal is a separate variant from
`#tierBelowFees`, which names the Stripe fee.

**The faucet rule.** The canister cannot enforce the allow-list at funding time: the
reserve arrives as an ICRC transfer to its ledger account, which it cannot refuse. It
enforces it at the sale instead. An empty list therefore means two things: it does not
filter while the floor is zero (nothing can be sold anyway), and it refuses everyone the
moment the floor is not.

**What the buyer sees.** A sentence that a real charge happens in Stripe's sandbox and a
fraction of the cycles is delivered. The receipt shows both legs: `checkReceipt`'s
`recomputed` stays the unscaled quantity, so a simulation receipt states what production
would have locked, the divisor, the locked quantity, and the ledger fee.

## Mode 3 — mainnet production

⚠️ **Before any of this**, work `docs/SANDBOX-TESTPLAN.md` to green. Every Stripe payload
in the automated suites is hand-crafted; that plan is the only thing that verifies the
real wire format.

⚠️ **Deploy only from a green `main`.** The `-Werror` gate that makes a non-exhaustive
match (M0145) a build failure runs on `mops check` in `scripts/test-all.sh` and in CI,
not on `mops build` or `icp deploy`. A direct-deploy hotfix bypasses it: a non-exhaustive
match would ship and trap at runtime, and on the webhook path that is a 5xx Stripe
retries for ~3 days.

⚠️ **A simulation gateway is not promoted in place; it is reinstalled.** The divisor is
refused once any order is stored, and `set_expected_livemode '(opt true)'` is refused
while the divisor is above 1. Reinstall breaks that deadlock and keeps the canister id,
so the webhook URL, both sealed ciphertexts and the reserve's cycles survive. Buyer
identity is unaffected: principals derive from the frontend canister id.

### What is not a command

Six prerequisites that no step below can perform:

1. **The migration chain, before there is data worth keeping.** `Main.mo` is a
   `persistent actor` with inline initializers and there is no
   `src/backend/migrations/`, so an incompatible stable-shape change has exactly one
   remedy, `icp deploy --mode reinstall`, and on a canister holding real orders that is
   not a remedy. Three facts to write it against: write it after the last
   schema-affecting change, since the init migration must enumerate every stable field;
   adding a stable `var` to the actor needs no migration, while a field on an existing
   stable record does; and ⚠️ **the baseline only helps while it is current**, so
   promote `deployed/backend.most` with `mops deployed` after every deploy
   (`AGENTS.md`). Read the `migrating-motoko-actors` skill first. No
   `preupgrade`/`postupgrade`, no `(with migration = ...)`.
2. **An alert someone actually receives.** `RUNBOOK.md` §8 is a complete monitoring
   plan and nothing runs it. The whole P1 set polls public queries, so the alerting
   layer needs no key. It is done when those metrics reach a human out of hours and
   someone has tripped one deliberately and watched it arrive.
3. **The claim, and the legal surface of being official.** *"At cost"* must hold net of
   card processing or become a visible fee line. Beside it: imprint, terms, privacy and
   contact; invoices a developer can expense, and the VAT position; a refund procedure,
   because the canister models no refunds; and a staffed rotation for the obligation
   queue. The serving domain is not irreversible: the derivation origin is pinned to the
   frontend canister id, so a test domain and a production domain yield the same
   principals.
4. **`stripe_origin` must be https and non-loopback once livemode is `?true`.**
   `Session.validateOrigin` accepts `http://` for loopback hosts, and nothing refuses the
   pair `expected_livemode = ?true` with `stripe_origin = http://localhost:8000`. Read
   `stripe_origin` back after setting either one (steps 5 and 8). The permanent fix is a
   refusal on both setters, but ⚠️ **key each refusal on the bad value, not on the
   pair**, so a gateway already in the bad pair can always set a good origin and walk
   out. The divisor's mutual refusal is the shape to copy and not the constraint. Do not
   add a second lockout to a money-handling setter.
5. **Attestation coverage of the confidential subnet** (`RUNBOOK.md` §9). Checkpoints
   and state-sync are confirmed confidential on the target subnet. Attestation coverage
   is the box still open: one unattested replica is one node provider who can read the
   webhook secret.
6. **Where the repo lives, before the "check the code" link is published.**
   `RELEASE.md`'s trust story is *verify the deployed module hash against a tagged
   commit*, so the repository URL is a user-facing artifact. Renaming or moving is free
   now and costs redirect debt once that link is on a money page.

### The steps

⚠️ **Two of these are knowingly not done on the live simulation deployment**: step 2's
freezing threshold and step 10's backup controller. Read
`icp canister status backend -e ic` before assuming either has been done.

1. **Deploy and verify** per `RELEASE.md`: reproducible build, published module hash,
   and `icp canister status` gated on matching it.

2. **The canister's own gas, and its freezing threshold.** Read the balance and compute
   the difference; never paste a figure. Then raise the freezing threshold: losing this
   canister to a cycle drain destroys the order store, the journals and the dedup sets.

   ```bash
   icp canister status backend -e ic                 # read the balance first
   icp canister top-up backend --amount <diff> -e ic  # --amount, not --cycles
   icp canister settings update backend --freezing-threshold 7776000 -e ic  # 90 days
   ```

3. **Fund the cycles reserve, then tell the gateway to look** (`RUNBOOK.md` §5).
   Delivery transfers out of the gateway's own cycles-ledger account, a different pot
   from step 2; `icp canister top-up` does not touch it.

   ```bash
   icp cycles transfer <amount> <backend-principal> -n ic
   icp canister call backend refresh_reserve '()' -e ic
   ```

   ⚠️ **The second call is not optional.** Solvency is decided against a maintained lower
   bound that starts at zero and rises only by observation.

   ⚠️ **How much is a security decision before it is a working-capital one.** A forged
   webhook delivers from the reserve and nothing caps that, so the reserve balance is
   the blast radius (`RUNBOOK.md` §2 and §9). Size it to what you are willing to lose
   between a leak and its detection, and top up on a cadence. `maxPurchaseUsdCents` is
   the per-order exposure inside it.

4. **Register card tiers** (`RUNBOOK.md` §3):

   ```bash
   icp canister call backend set_card_tiers \
     '(vec { record { id = "t10"; usdCents = 1_000 : nat };
             record { id = "t20"; usdCents = 2_000 : nat };
             record { id = "t50"; usdCents = 5_000 : nat } })' \
     -e ic --identity <operator>
   ```

   ⚠️ **With an empty list the buy view offers NO WAY TO BUY.** The backend accepts any
   amount within the gate's bounds (`Amount` is `variant { custom : nat; tier : text }`),
   but `renderTiers` returns early on an empty list and the custom-amount tile is built
   after that return, so the page shows no tiles and no custom field. The whole-vector
   setter replaces the list; `'(vec {})'` clears it. No `$100` preset: that is the
   ceiling.

5. **Declare the Stripe mode:**

   ```bash
   icp canister call backend set_expected_livemode '(opt true)' -e ic --identity <operator>
   ```

   Until this is set, a test-mode webhook secret would deliver real cycles for payments
   that never happened. Verify with `expected_livemode`, and re-read `stripe_origin`
   (item 4 above).

6. **Configure the Stripe webhook destination**: `https://<backend-canister-id>.icp.net/webhook/stripe`,
   subscribed to all six events in Mode 2's table. An unsubscribed event is never sent,
   so its handler never runs.

7. **Provision the webhook signing secret**, sealed (Mode 2, step 4). Until it is set the
   webhook route answers 503 and Stripe retries.

8. **The origin, then the live API key: the pair that opens the rail.**

   ```bash
   icp canister call backend set_stripe_origin '("https://<your-origin>")' -e ic --identity <operator>
   # then set_stripe_api_key, sealed, per Mode 2's step 4
   ```

   The key is **LIVE, restricted (`rk_...`), Checkout Sessions = Write, everything else
   None**. A sandbox key cannot be reused. With either missing, `create_order` refuses,
   so do this last; rotating either closes the rail until both are valid again. There are
   no Payment Links, Products or Prices to create: `amount_total == usdCents` holds
   because of what the session does not enable, listed in `rails/Session.mo` beside the
   body builder and asserted absent by `test/session.test.mo`.

9. **Review the admission gate** (`RUNBOOK.md` §5a). The defaults are non-zero and
   usable, but `maxPurchaseUsdCents` should sit just above your largest tier, and
   `minCanisterCycles` should suit how closely you monitor this canister.

10. **Add a backup controller.** A single controller identity with no backup means a
    lost key makes the canister permanently un-upgradeable.

11. **Wire monitoring (`RUNBOOK.md` §8) before announcing the service**, and confirm an
    alert arrives.

12. **Smoke-check the public surface**: `pricing_status` (both rates populated and
    `lastAttempt.ok` true; if not, `lastAttempt.detail` names the failing guard and no
    order can be created until it clears), `reserve_status`, `recovery_status` (sweep
    timer armed), `cycles_status` (balance above `minCanisterCycles`), `card_tiers`, and
    `can_purchase '(<your smallest tier's cents>)'`, which should answer `ok`.

13. **Buy one thing on the deployed site with a real card.** Nothing short of a live
    purchase exercises the key, the origin, the return URL and the webhook secret
    together.

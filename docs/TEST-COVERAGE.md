# What is tested, how, and what is not

One place to answer "is X covered?". `scripts/test-all.sh` runs the whole gate,
fail-fast; `--fast` skips PocketIC.

## Running them

Two suites need a one-time install:

```sh
npm --prefix test/browser ci                             # first run only
npx --prefix test/browser playwright install chromium    # first run only
cd test/integration && npm ci                            # first run only
```

Per suite:

```sh
mops test                                    # Motoko unit
npm --prefix src/frontend run test           # frontend, vitest (pure + jsdom)
npm --prefix src/frontend run typecheck
npm --prefix test/browser test               # Playwright, Chromium
npm --prefix test/integration test           # PocketIC
```

- ⚠️ **`npm test`, never `npx vitest run`, in `test/integration`.** The `pretest` hook
  fetches the sha256-pinned ledger wasms and rebuilds the backend; skipping it tests a
  stale wasm and passes.
- ⚠️ **PocketIC needs a 4 KiB-page kernel.** macOS and x86_64 Linux are fine; the replica
  cannot run inside an arm64 Linux guest with 16 KiB pages. Node ≥ 20.11 and `mops` on
  `PATH`.

## The automated suites

Counts are deliberately absent: they drift. `test/integration/README.md` has the
scenario map by id.

| Suite | What it covers | How |
|---|---|---|
| **Motoko unit** (`test/*.test.mo`) | pure logic: HMAC, the Stripe signature scheme, JSON parsing, fee/rate arithmetic, the §4 state machine, dedup, order-bound problems and the orphan list, every money position `Delivery.terminationFor` can report, `stageOf`'s resume decisions, and reserve solvency | `mops test`. No IC environment: every module takes its dependencies as a record |
| **Frontend pure** (`format.test.ts`) | status mapping, cycle/USD formatting, the §3 pricing vector, slippage flooring, deposit-fee subtraction, receipt verification, every error-message mapping | `vitest` |
| **Frontend DOM** (`main.test.ts`) | the real `index.html` body in jsdom with a stubbed backend: tier estimates, fee split, the route into the buy form, the destination it sends, the acknowledge-then-confirm quote flow, cancel visibility, the receipt render, the view machine under fake timers | `vitest` + jsdom |
| **Browser** (`test/browser/*.spec.ts`) | what jsdom is structurally blind to: the cascade, layout, reachability, and, via committed screenshot baselines, paint. Runs against a production build served statically | `npm --prefix test/browser test` (Playwright, Chromium) |
| **PocketIC** (`test/integration/src/*.spec.ts`) | end-to-end against the **real** ICP ledger, CMC and cycles ledger, plus a sha256-pinned XRC mock at the mainnet id | `npm --prefix test/integration test` |

### What makes the PocketIC suite the real bar

- **Real NNS wasms**, at their mainnet ids, deployed by `icpFeatures`.
- **Real HMAC-signed Stripe payloads**, signed by an independent Node implementation, so
  it is not testing our signer against our verifier.
- **Time control**: the 2 h delay alert, the 72 h terminate bound and the ledger's 24 h
  dedup window are reachable in seconds.
- **HTTPS outcalls are parked, not performed.** Every `create_order` and `cancel_order`
  blocks on one, and the suite answers it, so the exact bytes the canister built can be
  read back (scenario 63). Three things it cannot tell you: the real cycle cost; whether
  `max_response_bytes` is big enough for a real Stripe response; and ⚠️ **whether the
  transform strips enough for consensus**. The mock does not enforce consensus the way a
  subnet does (verified by mutating `Session.strip` to leak every header: the suite
  stayed green). That failure is first observable against real Stripe.
- **Failure injection**: the NNS canisters can be stopped (`stopNns`, impersonating NNS
  root). The one that matters is the **cycles ledger**, which is how scenarios 11, 33, 35,
  47 and 80 hold an order undelivered; delivery asks no other canister, so nothing else
  can stall it. ⚠️ Read balances **before** stopping and put the stop in `try`/`finally`:
  a throw in between leaves the ledger stopped for the rest of the run.
- **Real HTTP**: `pic.makeLive()` serves an actual gateway, so scenario 55 proves the
  webhook route over genuine HTTP.
- **Upgrade-mid-flight**: real stop → upgrade (EOP keep) → start inside the §5.1
  ambiguity windows, asserting exactly one ledger debit.

## Coverage by concern

| Concern | Where | Note |
|---|---|---|
| Signature verification (rotation overlap, both-direction window, constant-time compare) | unit + PocketIC | externally pinned vectors |
| Attribution (claimed-not-trusted, owner mismatch, malformed, expired, cancelled) | unit + PocketIC | |
| Amount honouring (exact → delivers; any other amount → a refundable obligation; ceiling; currency) | unit + PocketIC | mutation-checked: disabling the equality check fails the suite |
| Dedup / replay (event id, payment intent, post-prune resend, credited-elsewhere) | unit + PocketIC | |
| Refunds (full, partial, cumulative partials, after delivery, of an escalated order) | unit + PocketIC | |
| Async payment methods (settle, fail, out-of-order) | unit + PocketIC | |
| Pricing guards (plausibility, delta, source count, implied XDR/USD, staleness) | unit + PocketIC | every guard rejects in isolation |
| Money-out: **one transfer** out of the reserve, exactly-once, journal replay | PocketIC | incl. across real upgrades |
| Outages: the **cycles ledger** down; delivery retries, strands nothing, then delivers | PocketIC | 11, 33, 35, 47 |
| Escalation → the right money position and instruction | unit (all 8 arms) + PocketIC | see the gaps below |
| Delay alerts, and **which** of the two terminal bounds fires | PocketIC | 47 (alert cleared on escalation), 80 (24 h dedup window), 35 (72 h max hold) |
| Buyer never stuck: cancel, expiry, late payment, quote pinning | PocketIC | |
| Frontend state machine and reactions | jsdom (`main.test.ts`) | real `index.html` body, stubbed backend |
| Frontend **rendering**: cascade, layout, reachability, paint | browser (`test/browser`) | screenshot baselines for paint |

## Mutations that are caught

Each was applied to the tree and the suite run, because "no test catches it" is only
meaningful next to a list of what does:

| Mutation | Caught by |
|---|---|
| drop the `#paid` clause from `unsettledDelivery` (the escalation-freeze bug) | scenarios **73** and **76** |
| credit the floor back on `#delivered`, a real debit, breaking §5.4 rule 2/3's asymmetry in the optimistic direction | an `afterEach` hook asserting `reserveFloor ≤ reserveBalance`, which fails at scenario **06**, the first delivery in the suite. Every scenario anyone adds is born checkpointed |
| `openEntry` recording a hardcoded status instead of the order's | `test/cmc.test.mo`'s coupling test, and scenario **75** |

## What is not covered, and why

### 1. The frontend against a real backend, and real Internet Identity

`main.test.ts` runs the real `index.html` body in jsdom with a stubbed backend; the
backend's behaviour is already proven by the PocketIC suite, and a stub is the only way
to drive a `#quoteChanged`, a delivered receipt, or a poll transition on demand.
`test/browser` runs a production build in Chromium, because jsdom has neither a cascade
nor a layout: `el.hidden` reads `true` for an element a class selector is keeping on
screen, and a pulse painting *over* step numbers was found only by a screenshot
baseline. A test-only fixture hook (`src/frontend/src/fixtures.ts`) replaces the backend
and nothing else, so sign-in, routing, the view machine and the 3 s poll are the app's
own code; it is absent from a production build (`__FIXTURES__` is a `define`d literal,
and `scripts/test-all.sh` greps the shipping bundle).

What no suite closes, and `docs/SANDBOX-TESTPLAN.md` carries:

- **The CLI handoff.** `icp identity link web` has never been run, so "the cycles are
  reachable from the CLI" is unproven. Two of its five steps are unreachable by any
  suite: the id.ai CLI-access switch is a setting in another product, and the delegation
  the link command returns needs a real browser sign-in.
- **Real Stripe payload capture.** 3 of 8 fixtures are committed; the remaining
  integration tests stay skipped until the rest are captured, and the suite prints which
  are missing on every run.
- **Refunds, async payment methods, disputes, and anything live-mode.**

### 1b. Outcall consensus is not testable anywhere before mainnet

A local network is a single replica, so HTTPS-outcall consensus is never exercised
locally, by any run, manual or automated. PocketIC mocks the outcall, so the transform
runs but no consensus step exists. ⚠️ **It is therefore first tested on the first
mainnet outcall**, and the failure mode is `No consensus could be reached. Replicas had
different responses.` Two mechanisms carry it: the transform strips every response
header (`rails/Session.mo`), and `Idempotency-Key = orderId` on the session create,
without which each replica creates a distinct session and no transform can repair the
disagreement. Both are arguments, not test results.

### 2. Structural limits in PocketIC, verified by mutation

| Not covered | Why |
|---|---|
| **the reserve floor adopting a balance across an in-flight outflow** | The window is established across an `await`, so the unsafe interleaving needs a transfer issued inside a reconcile's balance-read gap, which PocketIC cannot schedule on demand. What is covered: `test/reserve.test.mo` pins that a non-quiet observation is refused, and scenario 76 covers the direction that actually shipped (one escalated order making the window permanently unsatisfiable). The predicate's inputs are pinned instead: `test/delivery.test.mo` couples `Delivery.openEntry`'s status to the order's, after a hardcoded status once made the window always satisfied and 76 pass vacuously |
| **the delivery replay sending the intent's original fee** | Re-reading `icrc1_fee()` on the replay path passes every assertion: the integration suite runs against a real cycles ledger whose fee never moves, and catching it needs a fee change inside the 24 h dedup window. If the ledger's dedup key includes the fee, a replay after a fee change pays the buyer twice, so the code comment is the guard |
| **the stored ledger fee persisting after `#BadFee`** | Staging it needs the stored fee to differ from the ledger's, and there is deliberately no `set_cycles_ledger_fee` lever: the only state such a lever fixes is one it can create, and a typo in it shorts buyers. The failure is loud: `delivery.feeChanged` fires on every delivery instead of once, which `RUNBOOK.md` §8 carries as a P3 row. The ledger's report is unit-pinned (`interpretTransfer(#Err(#BadFee))`) |
| **the reserve decision pairing a stale balance with a live tally** | Closed by design rather than by test: the decision is now synchronous against `reserveFloor`, so there are no two values to pair. Scenario 72 guards the invariant `promisedTotal ≤ balance` |
| a **trapping** daily reconcile | The reconcile is detached into its own message so a trap cannot stop the sweep, but nothing can inject that trap. What is covered is that the detached message runs, commits, and is cadence-gated in both directions (scenario 58) |

### 3. The deployment layer, covered separately by `scripts/e2e-local.sh`

The PocketIC suite installs wasms directly and never runs `icp deploy`.
`scripts/e2e-local.sh` covers that against a real local network, and is not part of
`scripts/test-all.sh` because it needs one running:

| What only this can prove | Why the PocketIC suite cannot |
|---|---|
| the deploy pipeline: recipes, `shrink: true`, embedded `candid:service`, the Candid compatibility check | it installs wasms directly |
| the **`PUBLIC_CANISTER_ID:xrc` override branch** | in PocketIC the mock sits *at* the mainnet id, so only the fallback branch ever runs |
| the asset canister and its `ic_env` cookie | there is no asset canister and no HTTP gateway serving it |
| local Internet Identity responding | `ii: true` is a network feature, not a canister one |
| `icp.yaml` environment scoping: `ic` lists only `backend` and `frontend` | a config fact, not a canister behaviour |

It reports two conditions loudly, because both look identical from inside a deploy: a
breaking Candid change against the deployed canister, and a stable-shape change whose
upgrade traps in `register_stable_type`. Locally it retries with `--yes` and
`--mode reinstall`. Two gotchas it encodes: the xrc mock keeps its canned response in
heap and sets it from `init_args`, so a routine `icp deploy` upgrades it and every later
rate call traps with "Response has not been set" (the script reinstalls it first); and
`rates` stays null until the local gateway is seeded, because the CMC's rate is settable
only by NNS governance.

### 4. Things only a real Stripe account can show

Covered by `docs/SANDBOX-TESTPLAN.md`, not by any suite: the real wire format (every
payload in the repo is hand-crafted JSON written from the API docs), completing a
hosted Checkout page, and live-mode behaviour (Radar, 3DS, payouts, account
restrictions).

### 5. No coverage measurement

There is no instrumentation, no `c8`/istanbul for TypeScript, nothing for Motoko. The
tables above are a qualitative map, deliberately: an unmeasured percentage would be
worse than an honest inventory.

## Continuous integration

`.github/workflows/mops-test.yml` runs three jobs on `ubuntu-latest`:

- **motoko**: lint, unit suites, and that the committed `.did` is current
- **frontend**: build (regenerates bindings) → typecheck → tests
- **integration**: typecheck + the PocketIC suite (`ubuntu-latest` is a 4 KiB-page host)

Triggers: `push` on `main`/`master`, `pull_request`, and `workflow_dispatch`. A push to a
feature branch matches none of the first two: open a PR or dispatch it manually.

⚠️ **`npm ci` mismatches in `test/integration` cannot be reproduced on macOS.** Linux-only
optional dependencies (`@emnapi/*`, through vitest's rolldown wasm fallback) are never
recorded by a macOS resolve, so a local `npm ci` passes while CI fails. If that job
fails on install, delete `test/integration/package-lock.json` and `node_modules` and
resolve fresh. CI is the only test for it.

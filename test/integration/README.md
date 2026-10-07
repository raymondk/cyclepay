# PocketIC integration suite — the Card rail go-live bar

> **Looking for what is tested where, across all suites?** See `docs/TEST-COVERAGE.md`.
> This file documents the PocketIC suite specifically.

End-to-end scenarios against PocketIC instances running the **real** canisters: the
cycles ledger and the cycles minting canister are deployed at their mainnet ids by
PocketIC's `icpFeatures`, the same wasms mainnet runs. The rest is driven:

- **Stripe**: crafted `checkout.session.completed` / `charge.refunded` payloads,
  HMAC-SHA256-signed exactly per the `Stripe-Signature` scheme, delivered through the
  canister's real HTTP ingress path.
- **The Exchange Rate Canister**: the released `xrc_mock` wasm is installed at the
  mainnet XRC id (`uf6dk-hyaaa-aaaaq-qaaaq-cai`, whose range lives on the II subnet,
  found by searching the topology; see `subnetHosting`). `setXrcResponse` drives it to
  return a specific rate, quality signal, or any of the 16 error variants. ⚠️ The mock's
  response is init-only, so changing it means a reinstall, which `setXrcResponse` does.
- **NNS governance**: the CMC's ICP/XDR conversion rate is set by calling
  `set_icp_xdr_conversion_rate` with the governance canister principal as sender.
- **Time**: staleness windows, the recovery timer and the delivery max-wait are driven
  with PocketIC time control; mid-flight interruption tests step rounds one tick at a
  time and upgrade the canister inside the §5.1 ambiguity windows.

## Running

```sh
cd test/integration
npm ci
npm test        # pretest fetches the pinned ledger wasm + builds the backend
```

Requirements: Node ≥ 20.11 and `mops` on PATH. The PocketIC server binary ships with
the `@dfinity/pic` npm package.

- ⚠️ **`npm test`, never `npx vitest run`.** The wasm PocketIC installs is built by the
  `pretest` hook, which does not fire for a direct `vitest` invocation, so the suite
  silently runs against a stale backend. This matters most when mutation-checking a
  scenario: mutate a `.mo` file, run bare `vitest`, and the mutation appears not to
  break anything. The tell is a suspiciously complete pass, and a mutation run that
  finishes as fast as a no-op run did not rebuild.
- **The host must run a 4 KiB-page kernel**: macOS and x86_64 Linux. The replica's
  memory tracker hard-asserts 4096-byte pages, so the server crashes at instance
  creation inside arm64 Linux VMs with 16 KiB kernels.
- **Upgrading requires stopping first, and an explicit EOP option.** A canister with
  outstanding callbacks cannot be upgraded, and enhanced orthogonal persistence requires
  `upgradeModeOptions: { wasm_memory_persistence: [{ keep: null }] }`. Both are handled
  by `upgradeBackendMidFlight`.
- ⚠️ **`stopCanister` drains outstanding callbacks rather than discarding them**, so an
  upgrade cannot be used to interrupt a call mid-await. The mid-flight scenarios assert
  the stronger property: state after the drain is consistent and the §5.1 ambiguity
  rules still hold.

## One `@icp-sdk/core`, and why `overrides` is here

`@dfinity/pic` depends on `@icp-sdk/core@^5.0.0`; `@icp-sdk/vetkeys` declares it as a
peer at `^5.0.0 || ^6.0.0`. Left alone, npm satisfies the peer with 6.x at the top level
while pic keeps 5.x nested: two copies, two nominal `Principal` types, and `TS2345`
errors in a file the change never touched. So `@icp-sdk/core` is a direct devDependency
at `^5.4.0`, and `"overrides": { "@icp-sdk/core": "$@icp-sdk/core" }` forces every
transitive user to that resolution. The override is not recorded in
`package-lock.json`; CI gets a correct tree because the lock already resolves one copy.
The override earns its place on re-resolution.

## CI

`ci/integration.yml` is a ready GitHub Actions job (ubuntu-latest is x86_64, so PocketIC
runs natively). It lives here rather than in `.github/workflows/` because the sandbox's
deploy token lacks the `workflow` scope; to enable it:

```sh
git mv test/integration/ci/integration.yml .github/workflows/integration.yml
git commit && git push   # needs a workflow-scoped token
```

## ⚠️ The suite is ORDER-COUPLED by design

One gateway, one PocketIC instance, orders shared across scenarios (`orderA`…`orderF`),
one mutable XRC mock, one clock that only moves forward. That is what makes the
scenarios cheap and lets later ones assert against state earlier ones built. **When a
scenario fails, suspect a neighbour's state before you suspect its subject**: one stale
assertion that failed before restoring the XRC mock's rate has made the next 23
scenarios fail in `ensureRates`.

- **Every fix needs a full run.** `-t` on one scenario skips the state it depends on.
  Budget ~4 minutes per verification.
- **Read the failure list top-down and fix the first one.** A genuine failure takes
  hundreds of milliseconds to seconds; cascade victims fail in ~20–90 ms in setup.
- **`advanceTime` is global and irreversible**, which is why several scenarios carry an
  explicit `ensureRates` / `setCmcRate` re-arm at their end.
- ⚠️ **Advancing the clock past ~65 minutes provokes background HTTPS outcalls.** Every
  order gets a 35-minute deadline, so once the clock passes that plus the sweep's
  30-minute grace, every lingering `#created` order becomes due for a session
  **retrieve**. The recovery sweep is therefore a second, background producer of parked
  outcalls. `awaitPendingOutcall` filters for that: it answers any sweep retrieve it
  meets with `{"status":"open"}` and keeps looking for the one the scenario asked for.
  `afterEach` drains as a backstop, and a drain count above one there is a signal.
- ⚠️ **Read ledger balances BEFORE `stopNns`, and put the stop inside `try`/`finally`.**
  A balance query against a stopped canister throws, and a throw between the stop and
  the `try` leaves the ledger stopped for the rest of the run.
- ⚠️ **76 and 77 both consume the order scenario 35 escalates**, through the suite-global
  `orderEscalated`, because reproducing that state costs another 72 h of clock advance.
  They assert the shape they depend on rather than assuming it. 76 must stay before 77:
  77 fills in the block index, which settles the entry 76 needs unsettled.

## Scenario map

⚠️ **Partial by construction, and it drifts.** `grep -oE "^test\('[0-9]+" src/gateway.spec.ts`
is the authoritative list. If you change what a scenario asserts, change its row.
Numbers with no row (20–39, 64, 70) are either unmapped or were reserved by scenarios
cut before landing.

| # | Scenario | Coverage |
|---|----------|---------|
| 01 | 503 before secret provisioning, controller-gated admin API | — |
| 02 | tier config gating | — |
| 03 | empty cache + a failing XRC → `#rateUnavailable`; every rate guard rejects in isolation | pricing fail-closed |
| 04 | XRC + CMC through the real derivation; §3 pricing vector; order authz | happy path (pricing half) |
| 05 | signature/window/404/405/413 guards on the live route table | duplicate/replay (guards) |
| 06 | the money-out path is ONE transfer out of the reserve, exact in both directions | delivery arithmetic |
| 07 | the transfer memo is the ORDER id, so two identical orders both deliver | happy path |
| 08 | event-id dedup, intent dedup, `#duplicate`, refund auto-resolve | duplicate/replay |
| 09 | claimed-not-trusted attribution → `#unattributed` | refund obligations |
| 10 | delivery to a real cycles-ledger account | happy path |
| 11 | a cycles-ledger outage strands nothing: the order stays `#paid`, no obligation is filed, and the sweep replays the same intent and delivers | reserve delivery under outage |
| 12 | an upgrade concurrent with delivery pays exactly once, and the timer re-arms | upgrade-mid-flight, §5.1 replay, postupgrade re-arm |
| 15 | audit-log seq monotonicity, the delivery path's tag contract; a failed delivery files nothing, because only fiat can be stranded | — |
| 16 | admission gate: no headroom refuses the quote; `can_purchase` agrees; restoring headroom re-opens the rail | pre-creation gate |
| 17 | per-purchase ceiling bounds both tier registration and the amount | pre-creation gate |
| 18 | expiry: only `checkout.session.expired` moves an order there; it survives a simulated year undeleted, and a late payment files a refund obligation | Stripe owns the deadline |
| 19 | owner-only `receipt`; recomputes `net × P × 10¹² / U == lockedCycles` from it | price verifiability |
| 39 | a payment against a **cancelled** order files a refund obligation and never traps | the guard that keeps `markPaid`'s trap unreachable |
| 40 | `quote_previews` fee split, §3 vector, deposit fee, and an order locking exactly the previewed figure | quote/lock agreement |
| 41 | a +40% ICP move → `#quoteChanged` naming the new figure, nothing created; a favourable move never refuses; `null` opts out | server-side quote pinning |
| 42 | owner-only `cancel_order` produces `#cancelled` and frees a slot, is idempotent, refuses a paid order, and a payment racing the cancel is refunded, not converted | buyer's decision wins |
| 43 | a partial `charge.refunded` leaves the obligation open; completing it settles | refund amount fidelity |
| 44 | a verified-but-unprocessable event is acked 200 and filed once; unverifiable input still 400s | endpoint-disable avoidance |
| 45 | `checkout.session.async_payment_succeeded` delivers a payment that `completed` reported unpaid | delayed payment methods |
| 46 | a test-mode event cannot deliver on a gateway declared live; a live one still does | livemode gate |
| 47 | a delay alert is resolved when the order escalates, not only when it delivers | no orphan worklist entries |
| 49 | `async_payment_succeeded` arriving before `completed` still delivers once | out-of-order events |
| 55 | the webhook route over a real HTTP gateway (`live-gateway.spec.ts`) | real ingress |
| 56 | the per-purchase ceiling cannot be lowered under a live tier | config safety |
| 57 | an already-credited intent is caught before attribution | double-credit protection |
| 58 | the sweep reconciles the status tallies on its own cadence and reports no drift | tally integrity |
| 59 | a Stripe resend past the dedup window does not file a second unprocessable | redelivery vs double-pay |
| 61 | a crafted `create_order` for another principal's account, or a non-default subaccount, is refused by the canister | destination enforcement |
| 62 | the order fields and `ratesFetchedAtNs` survive a real stop → upgrade → start; a `#cancelled` order stays unpayable across the upgrade | durable order record |
| 63 | the Checkout Session request is byte-for-byte what Stripe needs | outcall payload fidelity |
| 65 | a session that cannot be created fails the order in the same call | no order without a payable URL |
| 66 | cancelling is atomic with Stripe, never half-cancelled | `#cancelled` means unpayable |
| 67 | `checkout.session.expired` is the only thing that expires an order | Stripe owns the deadline |
| 68 | a cancel racing session creation cannot leave a payable URL behind | no orphaned session |
| 69 | a failed session creation racing a cancel does not double-release | tally integrity under a race |
| 71 | a custom amount is bounded by the gate in both directions | floor and ceiling on typed input |
| 72 | a delivery completing during a create cannot manufacture capacity | `promisedTotal ≤ balance` |
| 73 | a funded reserve sells nothing until the gateway observes it; one quiet observation adopts the ledger's truth | §5.4 rule 1 |
| 75 | a buyer heals their own stuck delivery; a stranger and the anonymous principal cannot; the admin lever is the only one audited | owner-scoped `process_order` |
| 76 | one escalated order must not freeze the reserve reconcile forever | regression test for a shipped bug |
| 77 | an escalated order whose cycles did arrive is recorded as delivered rather than abandoned | `#needsReview → #delivered` |
| 78 | an order whose delivery is unsettled cannot be abandoned into a double payout | the guard behind `#deliveryOutstanding` |
| 80 | a delivery stuck 25 h escalates as `staleIntent`, the dedup bound, not the 72 h one | which bound fires |
| 88 | five sub-minimum `create_order` calls tally and write zero audit lines | refusals never write a line per attempt |

Scenario 88's assertion is a line count, the only form that catches the regression:
`#amountBelowMin` needs no prior state and the audit log never prunes, so an audit line
per attempt is a permanent, free-to-provoke leak. The counter assertion is the other
half: the tally moving by exactly 5 pins that the information survived the move.

### Live HTTP gateway (`live-gateway.spec.ts`)

`pic.makeLive()` starts a real HTTP gateway on a real port, so the webhook route can be
exercised over genuine HTTP. That is also the setup for a manual Stripe run:
`setTime(new Date())` so real signature timestamps verify, then
`stripe listen --forward-to http://127.0.0.1:<port>/webhook/stripe?canisterId=<id>`. The
spec logs the URL, so a full end-to-end Stripe test needs no local network and no
mainnet; see `docs/SANDBOX-TESTPLAN.md`.

⚠️ Its own instance and its own spec file, because `makeLive` enables auto-progress,
which is incompatible with the `advanceTime` control every other scenario uses. Call
`stopLive()` before any time travel.

### Failure injection against the real NNS canisters

NNS root controls the canisters `icpFeatures` deploys, and PocketIC accepts any
impersonated sender, the same mechanism `setCmcRate` uses to impersonate governance:

```ts
await stopNns(gw, CYCLES_LEDGER_ID);   // calls into it are now rejected
await startNns(gw, CYCLES_LEDGER_ID);  // service restored
```

Stopping the cycles ledger is what makes the delivery failure paths reachable end to
end: an order parks at `#paid` with a journalled intent and no block, which is the shape
every unknown-position scenario needs (11, 35, 47, 75, 76, 78).

**What genuinely remains out of reach:** a reserve observation adopted across an
in-flight outflow (the quiet window is established across an `await`, so the unsafe
interleaving needs a transfer issued inside a reconcile's balance-read gap), and a
recorded real Stripe event. Every payload here is hand-crafted JSON; only the
`docs/SANDBOX-TESTPLAN.md` run closes that.

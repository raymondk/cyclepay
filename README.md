# CyclePay — Fully On-Chain Cycles Gateway

**Buy cycles with a credit card.** No ICP, no wallet, no exchange account: it gets a
developer from "I have a card" to "my canister has cycles" without first solving crypto
onboarding.

It runs entirely on the Internet Computer, one Motoko backend canister and one
certified-assets frontend canister, **no server**. It sells cycles from a reserve it
already holds, prices them from two on-chain rates with no outbound HTTPS in the pricing
path, and shows the buyer the cycle quantity before they commit. Everything
money-touching **fails closed**: a freshly deployed gateway accepts no orders and
delivers nothing until each lever is consciously set.

## It is running

**<https://cyclepay.raymondk.co>**, or <https://shy4u-4qaaa-aaaay-aadhq-cai.icp.net>,
the same app on the canister's own gateway origin. Internet Identity derives principals
from the **frontend canister id**, so both addresses give the same account.

⚠️ **Simulation mode, and you cannot buy on it.** Cards are charged in Stripe's sandbox,
and cycles are divided by `pricing_status().config.divisor`, so a purchase quotes *and*
delivers that fraction of what a live gateway would. Purchases also require the buyer's
principal to be allow-listed by a controller: without that, free sandbox payments
against a funded reserve would be a faucet, and an unlisted principal is refused with
`buyerNotAllowed`.

Anyone can browse, read a live quote for any amount, and check every operational number
the gateway publishes, with no identity:

```bash
icp canister call backend quote_previews '(vec { 1_000 : nat })' -e ic  # $10, with both rate inputs
icp canister call backend quote_for_cycles '(vec { 5_000_000_000_000 : nat })' -e ic  # the least amount that buys 5 T
icp canister call backend reserve_status  '()' -e ic
icp canister call backend pricing_status  '()' -e ic
icp canister call backend lifecycle_config '()' -e ic
icp canister call backend card_tiers      '()' -e ic
```

Canister ids are in `.icp/data/mappings/ic.ids.json`; the backend is
`saz2a-riaaa-aaaay-aadha-cai`.

## How it works, and how to run it

[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) has the diagram: the money path end to
end, where the trust boundaries sit, and which of the two cycle pots a delivery spends
from. Start there.

| you want to | go to |
|---|---|
| understand the system | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md), then [`docs/DESIGN.md`](docs/DESIGN.md) for *why* |
| run it locally | [`docs/OPERATE.md`, Mode 1](docs/OPERATE.md#mode-1--local) |
| deploy it | [Mode 2 — mainnet simulation](docs/OPERATE.md#mode-2--mainnet-simulation) or [Mode 3 — production](docs/OPERATE.md#mode-3--mainnet-production) |
| operate one that is misbehaving | [`RUNBOOK.md`](RUNBOOK.md#enter-here-what-you-are-looking-at), entered by symptom |
| check the claims yourself | [`docs/VERIFY.md`](docs/VERIFY.md#what-anyone-can-check-right-now) |

```sh
git clone --recurse-submodules https://github.com/marc0olo/cyclepay
icp network start -d && icp deploy && scripts/local-dev-seed.sh
```

⚠️ **The submodule and the seed are both load-bearing**, and neither failure looks like
its cause. [`docs/OPERATE.md`, Mode 1](docs/OPERATE.md#mode-1--local) explains both.

## Documents

For a **reader or verifier**:

| | |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | The diagram: canisters, money path, trust boundaries, the two cycle pots |
| [`docs/DESIGN.md`](docs/DESIGN.md) | The decision record, *why* it is built this way. What the `§N` comments point at; gate-enforced |
| [`docs/STRIPE.md`](docs/STRIPE.md) | The Card rail end to end, written from the code |
| [`docs/TEST-COVERAGE.md`](docs/TEST-COVERAGE.md) | What is tested, how, and what is not |
| [`docs/VERIFY.md`](docs/VERIFY.md) | What anyone can check about the live deployment, and what they cannot |
| [`docs/BUYER-COST-MODEL.md`](docs/BUYER-COST-MODEL.md) | What a purchase buys, and why the floor is $10 |

For an **operator**:

| | |
|---|---|
| [`docs/OPERATE.md`](docs/OPERATE.md) | Setup, one procedure per mode: local, mainnet simulation, mainnet production |
| [`RUNBOOK.md`](RUNBOOK.md) | Day-2 operations, entered by symptom |
| [`RELEASE.md`](RELEASE.md) | Cutting a release: build, publish hashes, install, gate |
| [`CHANGELOG.md`](CHANGELOG.md) | What changed per release; the release script refuses a version with no entry |
| [`docs/SANDBOX-TESTPLAN.md`](docs/SANDBOX-TESTPLAN.md) | The manual Stripe-sandbox pass required before go-live |
| [`docs/DEMO-PLAYBOOK.md`](docs/DEMO-PLAYBOOK.md) | The running order for demoing this to a technical audience |

For an **agent changing the code**: [`AGENTS.md`](AGENTS.md) (conventions, skills, the
verification gate) and `docs/DESIGN.md`. `docs/agents/` holds the loop's own
conventions: the issue-tracker rules and `deleted-vocabulary.md`.

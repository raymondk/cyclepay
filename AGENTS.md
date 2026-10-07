# Agent instructions

**This file, `docs/DESIGN.md` (decisions), `RUNBOOK.md` (operations) and
`docs/OPERATE.md` (setup) are the sources of truth.** Settlement is a cycles reserve,
payment is per-order Stripe Checkout Sessions, and `docs/OPERATE.md`'s Mode 3 carries
what remains to do before real money.

## ICP skills

ICP skills are tested, frequently updated instruction files maintained by DFINITY
(<https://skills.internetcomputer.org>). Consult the relevant skill **before** making
changes: `writing-motoko`, `icp-cli` and `canister-security` all contradict pre-training
knowledge in ways that matter here.

<!-- ic-skills:managed:start -->
<!-- state: configured (autosync) -->
ICP skills auto-update each session via a SessionStart hook
(`.claude/sync-ic-skills.sh`) and live in your agent skills directory — you don't
need to run anything to refresh them. Skills are authoritative — prefer them over
general knowledge for all ICP work. If they are not present (hook hasn't run, or
`jq` is missing), fetch them on demand per the fallback below.
<!-- ic-skills:managed:end -->

- **Fallback for agents without the hook** (Cursor, Copilot, Codex): fetch
  `https://skills.internetcomputer.org/.well-known/skills/index.json` once per session,
  then the matching skill's `SKILL.md` before writing ICP code.
- **Layout:** `.claude` is a committed symlink to `.agents`; the tracked files are
  `.agents/settings.json` and `.agents/sync-ic-skills.sh`. `.agents/skills/` is
  gitignored.

### Where this project knowingly departs from a skill

A recorded departure has a scope. When a skill and this file disagree, the skill wins
unless the reasoning below names *that specific finding*; if the reasoning has expired,
fix this file rather than working around it.

- **`canister-security` pitfall 9** ("never store API secrets in canister state"). The
  Stripe webhook signing secret *is* stored plaintext. HMAC is symmetric, so a canister
  that can verify can forge; encryption only moves the problem to a key the canister
  also needs. Confidentiality comes from the SEV-SNP subnet, and the reserve balance is
  the blast radius. See `src/backend/Secret.mo` and `docs/DESIGN.md` §7.
- **`writing-motoko` architecture pattern** is followed: endpoints live in
  `src/backend/mixins/`, `Main.mo` declares no public methods, and the domain logic is
  flat stateless modules (`Orders`, `Delivery`, `Gate`, `Reserve`, `Pricing`,
  `Receipts`, `rails/Card`) rather than a `lib/` directory. `docs/DESIGN.md` §9.1 has
  the rules the mixin split rests on. Three things a reviewer will otherwise flag:
  - **A3 (bodies over 20 lines).** Count a body as non-blank, non-comment lines
    including the `public … func` line and the closing `};`. The bodies still over
    the line are long because of an inline return type or guards, not logic, and the
    decisions that were extractable have been (`Purchase.plan`, `Orders.cancelShape`).
    Extracting a decision buys unit tests, not a shorter body.
  - ⚠️ **`create_order`'s commit → outcall → re-check → attach sequence stays in the
    endpoint.** The reserve hold is taken in a block with no `await`, the order id is
    the `client_reference_id`, and it needs actor capabilities (`raw_rand`, the
    outcall). Integration scenarios 67b and 67c guard it. Do not "finish" A3 by moving
    that block into a module.
  - ⚠️ **`scripts/check-did-signatures.sh` is an identity check, not a compatibility
    check.** Candid is structural, so naming an inline return record is wire-compatible
    and the script still reports it as changed, correctly for its job (proving a
    relocation moved nothing). There is no `didc` compatibility check in the gate; a
    deliberate interface change needs that judgement made by hand.

## Running it locally

```sh
git submodule update --init --recursive   # first time only
icp network start -d && icp deploy && scripts/local-dev-seed.sh
```

`docs/OPERATE.md`, Mode 1, is the procedure. Two things it explains that will
otherwise cost an hour: the pinned crypto submodule is a `mops` path dependency, so
without it the error names a missing package rather than a missing submodule; and the
seed is not optional, because a fresh deploy is fail-closed on five axes at once and
presents as a broken app.

### Do not cycle the network

`icp network start` takes minutes, and two projects cannot run local networks at once
(`gateway.port: 8000` is fixed in `icp.yaml`).

- Check first with `icp network status`. If one is running, use it.
- Start only if absent, and remember that you started it.
- ⚠️ **Never stop a network you did not start.** You cannot tell whether a human is
  mid-run or whether it belongs to another project; stopping one has destroyed a
  session's worth of delivered test orders and a local Internet Identity registration.
- For a clean slate, `icp deploy --mode reinstall --yes` then `scripts/local-dev-seed.sh`
  takes seconds. Do not restart.
- Leave it running when you finish, and say so in the PR.

## Scope: the Card rail is the product

CyclePay onboards developers who have no ICP, no wallet and no exchange account. A
stablecoin rail is not an option for that user, because acquiring the stablecoin is
the same problem again. The card rail is the only rail; a disabled ck-USDC rail was
removed because carrying an unshipped rail made every change bigger. `Types.Rail` and
`Types.Owner` stay single-case variants so a future rail is additive.

## Project conventions

- **`icp-cli`, never `dfx`.** Project config is `icp.yaml`; Motoko deps are
  `mops.toml` / `mops.lock`.
- **Comments document what the code does.** Not what it used to do, and not its own
  history. Design history belongs in commit messages and GitHub issues. The narrow
  exception: a comment that stops a future mistake stays, written as a rule rather
  than as a story.
- ⚠️ **The ⚠️ marker is for a trap, not a fact.** The test: *can the next editor act on
  this the wrong way, and would the damage be unrecoverable or invisible?* "Do not add a
  force flag to `refresh_reserve`" earns it; "this count going low is the oversell
  direction" is orientation and loses the glyph. `scripts/check-markers.py` enforces
  what a check can own:
  - never in a test, suite or describe name;
  - `MARKER_CEILING` is the tree-wide total and may only fall. A new marker is a
    one-line ceiling edit with the reason in the commit;
  - an endpoint `///` doc is published into `backend.did` and the generated
    TypeScript, so a marker there reaches API callers. Keep the ones a caller can act
    on; a note to the next editor goes in a `//` beside the implementation.
    `PUBLISHED_CEILING` ratchets that count the same way.
- ⚠️ **No bare `#NN` issue reference outside `docs/agents/`** (`scripts/check-issue-refs.py`).
  An issue number reads as a pointer to a live requirement and is a pointer to a closed
  argument. Say what the reference stood for in terms of the code as it is now, or drop
  it. An external tracker stays, written qualified (`dfinity/icp-js-core#1384`), because
  the number is the only way a reader reaches the claim's source.
- **Deleting a comment has no failure signal**, so three rules apply:
  1. Name the mistake the comment prevents. The test is "would the next editor of this
     function get it wrong without this line?", not "is this true and worth knowing".
  2. Never delete without a destination: `docs/DESIGN.md`, or a rule rewritten at the
     site. A removal with no destination is the one case a reviewer cannot check.
  3. The commit message carries the mapping of what moved where.
- ⚠️ **When you delete a mechanism, add its vocabulary to
  `docs/agents/deleted-vocabulary.md`**, including the *verb* for what it did and the
  names of deleted variants and fields, then grep every hit asking whether it is cited
  as the justification for unrelated behaviour. `scripts/sweep-vocabulary.py` scans the
  lines a change adds and prints each hit with its recorded disposition; it passes on
  hits, so read them. Verify a `docs/DESIGN.md` section against the code *before*
  deleting the comments it replaces: a section transcribed from comments inherits their
  errors.
- ⚠️ **`docs/DESIGN.md` is where a decision goes.** Code comments say what the code
  does; the reason goes in the `§N` record. `scripts/check-design-sections.py` fails
  the gate if a `§N` the code cites has no section, or a section is cited by nothing.
  What no check can verify is whether a section is true, so the rule is *change the
  behaviour, change that file in the same commit*.
- The committed `src/backend/dist/backend.did` is the embedded `candid:service`
  metadata *and* the frontend's bindgen source. Run `mops build` after any backend API
  change and commit the regenerated `.did`.
- **Any fact about money lives on a permanent record.** The audit log is telemetry. If
  a behaviour produces a fact someone could need in six months, put it on the order or
  the journal, not only in an audit tag.
- **Where to look for what:**

  | Question | Source |
  |---|---|
  | What does it do? | the code, and `docs/STRIPE.md` for the Card rail end to end |
  | How do I operate it? | `RUNBOOK.md` |
  | Why is it built this way? | `docs/DESIGN.md` |

## Issue tracker

GitHub Issues is the single source of truth for task and progress tracking.
`docs/agents/issue-tracker.md` has the `gh-axi` conventions and the body-rewrite script.
Triage labels are the default five: `needs-triage`, `needs-info`, `ready-for-agent`,
`ready-for-human`, `wontfix`.

## Verifying your work

Never report a task done on a build alone. One command runs the whole gate:

```sh
scripts/test-all.sh          # everything, in dependency order, fail-fast
scripts/test-all.sh --fast   # skips the PocketIC suite (see the host note below)
```

Several steps are checks a hand-run sequence silently skips, each because the thing it
checks had drifted once:

| step | asserts |
|---|---|
| `.did` is current | the committed interface matches the code |
| `check-doc-surface.py` | the docs' method lists match the `.did` |
| `check-design-sections.py` | every `§N` the code cites exists in `docs/DESIGN.md`, and every section is cited |
| `check-heredocs.sh` | no unquoted heredoc runs its own body |
| `mops check`'s stable check | the actor's stable shape is still compatible with `deployed/backend.most`; it also fails when the baseline is missing |
| `check-bindings.sh` | the integration suite's committed Candid bindings match the `.did` |
| `check-unused-exports.py` | no module exports something nothing calls |
| `sweep-vocabulary.py` | prints added lines naming a deleted mechanism; advisory |

### Two compatibility checks that are not the same thing

| what you see | it means | what to do |
|---|---|---|
| `mops build`: `.did is out of date` | the committed Candid file does not match the code | `mops build`, commit the `.did`, regenerate the suite's bindings. Says nothing about stable state |
| `mops check`: `Stable compatibility check failed` | the stable shape cannot be reinterpreted from the deployed one | locally reinstall and reseed; on mainnet this needs a migration. ⚠️ This is the one where deployed data is at stake |

### Promote the stable baseline after any shape change, compatible or not

A stale baseline blinds the check: a compatible change left un-promoted means the next
*incompatible* change is compared against a shape the canister no longer holds, and
passes. So after any change that moves the stable shape, once it is deployed:

```sh
mops build
scripts/check-stable-promotion.sh  # what IS this diff? run it BEFORE promoting
mops deployed                      # promotes dist/backend.most → deployed/backend.most
git add deployed/backend.most      # committed in the SAME PR as the shape change
```

⚠️ **Read the diff with the script, never with your eyes.** A compiler upgrade renumbers
every type hash in the file, and a real schema change is invisible inside that noise. The
script runs `moc --stable-compatible` both ways:

| Verdict | Means | Do |
|---|---|---|
| `REPRESENTATION-ONLY` | both directions pass — equivalent signatures | promote; note the compiler bump in the commit |
| `REAL shape change (upgrade-compatible)` | forward only — a field was added or widened | promote deliberately, and name what moved in the commit |
| `NOT upgrade-compatible` | forward fails — a deployed canister cannot take it | not a promotion. Reinstall pre-launch, or write the migration |

⚠️ **Re-promoting after a failed check is correct only while the shape change is
accompanied by a reinstall.** Once the canister holds data that cannot be recreated
(the first mainnet deploy you intend to keep, or the first real buyer's order), a failed
check means *write a migration* (`docs/OPERATE.md`, Mode 3), and `mops deployed` is
silently the wrong answer.

⚠️ **The baseline must be committed and must not be the build's own output.**
`src/backend/dist/backend.most` is gitignored; checking against it compares an artifact
to a regeneration of itself. Do not annotate the `.most`: it is parsed and compared for
equality. The commit message is where the reason goes.

### The individual steps

```sh
mops check                                   # lint + typecheck
mops test                                    # Motoko unit suites
mops build                                   # refreshes the committed .did
scripts/check-doc-surface.py                 # docs' method lists vs the .did
scripts/check-design-sections.py             # §N citations vs docs/DESIGN.md
npm --prefix src/frontend run build          # regenerates bindings
npm --prefix src/frontend run typecheck
npm --prefix src/frontend run test           # pure functions + jsdom (main.ts)
bash scripts/brand-lint.sh                   # banned characters, vocabulary, tokens
npm --prefix test/browser test               # Chromium: cascade, layout, paint
npm --prefix test/integration run typecheck   # vitest does not typecheck
scripts/sweep-vocabulary.py                  # added lines vs the deleted-mechanism list
mops deployed                                # promote the stable baseline after a shape change
npm --prefix test/integration test           # PocketIC scenarios — the go-live bar
```

- ⚠️ **No frontend change is done on a jsdom pass alone.** jsdom has no cascade and no
  layout, so `el.hidden` reads true for an element a class selector keeps on screen. The
  browser suite is not optional and the gate fails outright if it cannot run. Surfaces
  that need a session or a delivered order are reachable through the test-only fixture
  hook (`src/frontend/src/fixtures.ts`); see `test/browser/delivered.spec.ts`.
- ⚠️ **`npm test`, never `npx vitest run`, for the integration suite.** The latter skips
  `pretest`, which fetches the sha256-pinned wasms and rebuilds the backend, so it
  silently tests a stale wasm and passes.
- The PocketIC suite needs a **4 KiB-page host** (macOS or x86_64 Linux). The replica
  hard-asserts 4096-byte pages, so it dies at instance creation in arm64 Linux guests
  with 16 KiB pages. If you cannot run it, say the bar is unverified; do not infer it
  from the unit tests.

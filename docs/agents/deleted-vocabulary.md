# Deleted-mechanism vocabulary — the sweep list

**When you delete a mechanism, add its vocabulary here. When you sweep, start from this
file.** `scripts/sweep-vocabulary.py` reads the table below, scans the lines a change
adds, and prints each hit with its disposition. It is the last gate step so its output
sits above the summary. It passes on hits: most are correct prose, and a reviewer decides.
It fails only on an undeterminable base ref, which is the didn't-run case. An empty diff
is a pass.

Three kinds of term, because missing any one has already cost a sweep:

1. **The noun** (`ring`, `treasury`).
2. ⚠️ **The verb** for what it did (`evict`, `mint`, `reprice`). Prose cites verbs more
   often than nouns, and `evict` was the one a sweep of `ring`/`capacity`/`float` missed,
   leaving an operator-facing RUNBOOK line stating a cap that no longer existed.
3. ⚠️ **The names of deleted variants and fields** (`#deliveryDelayed`,
   `AddResult.evicted`). They hide in tables and doc comments that a code-shaped grep
   never reads; two were live operator triage rows for entries that can never appear.

## Three row types, cleared differently

1. **Dead vocabulary** (most rows): the term should appear only in end-state statements.
   Clear a new hit by fixing the prose.
2. ⚠️ **A tripwire on a live name** (`icrc1_fee`): the concept exists; what must never
   happen is a *call*. Its absence from the ledger's service type is the guard behind the
   fee-derivation rule (§5.1), which no test can catch. Clear a hit only by confirming it
   is still not a call.
3. ⚠️ **A live name colliding with a deleted one** (`#abandoned` is a live order status;
   only the queue *kind* of that name went). A bare count for this row is misleading.

The script excludes this file, since every term appears here by construction.

## Known-collision terms

Live names whose hits are almost always correct. The scan prints them in a separate
section below its main list, so a real hit is not buried among predictable ones. A term
listed here is de-emphasised, never skipped; if this section is renamed or emptied every
term prints in the main list.

- `\bmint` — the CMC is the Cycles **Minting** Canister, a live dependency.
- `\bretention\b` — `Idempotency.mo` prunes dedup keys on a real retention window.
- `#abandoned` — a live order **status**; only the deleted queue *kind* shared the name.
- `icrc1_fee` — a live ledger method. Row type 2: the hit is fine, a *call* is not.

## The terms

A disposition says what the word meant, that the thing is gone, and what makes a new
use wrong. No counts: they expired and nothing caught them.

| term | disposition |
|---|---|
| `ring buffer` | AGENTS.md, describing this very failure class |
| `\bring\b` | all end-state statements — "#37 removed the ring", "the capacity parameter and the eviction loop are gone" |
| `4,?096` | all stating the ring is gone; one is PocketIC's 4096-**byte page** requirement, unrelated |
| `\bfloat\b` | all "there is no float" statements |
| `treasury` | AGENTS.md, listing what was deleted |
| `burn cap` | DESIGN.md §7 contrasts the old cap against today's reserve-sizing argument; the others list it as deleted |
| `ck-?USDC` | AGENTS.md scope statement |
| `\bmint` | ⚠️ **all legitimate** — the Cycles **Minting** Canister is a live dependency, and `Cmc.mo` says "nothing here mints, and nothing may" |
| `\bretention\b` | ⚠️ **all legitimate** — `Idempotency.mo` prunes dedup keys on a real retention window |
| `\bevict` | all end-state statements after the #77 follow-up sweep |
| `error.?queue` | historical references to a *status* split (#34) plus RUNBOOK's account of it; the structure references are cleared |
| `Payment Link` | all "#33 deleted this" statements |
| `attach_payment` | all "#33 deleted this" statements |
| `Retention.mo` | both naming it as deleted |
| `icrc1_fee` | ⚠️ **all "no longer awaits" / "never await" — none calls it.** Its absence from the ledger's service type is a load-bearing guard (§5.1); if a hit ever becomes a call, that guard is gone |
| `delayedAlerts` | all "not the `delayedAlerts` map returning" prohibitions |
| `#deliveryDelayed` | naming what replaced it (`delayed_deliveries`). ⚠️ Was a **live triage row** in RUNBOOK §6 until #77's follow-up |
| `#abandoned` | ⚠️ **legitimate — this is a live order STATUS.** Only the deleted *queue kind* of the same name was removed, which is why a bare count is not evidence here |
| `#icpAtCmc` | clean |
| `AwaitingTreasury` | clean |
| `promisedForDecision` | TEST-COVERAGE, recording the defect its deletion closed |
| `order_stats` | clean |

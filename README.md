# adpi-data

The public current-state data feed for **Ad Platform Intelligence**.

This repository is machine-written. The acquisition pipeline on Oracle replaces
the whole branch on every successful run. **Do not edit anything here by hand** —
the next publish overwrites it.

## Where it is read

This feed is consumed by the **Ad Platform Capability Explorer** —
<https://rajeevg.com/solutions/capability-explorer>. The explorer renders these
records; this repository is the machine-written data behind it.

## What is here

| File | What it is |
| --- | --- |
| [`dataset.json`](https://raw.githubusercontent.com/Rajeev-SG/adpi-data/main/dataset.json) | The current-state v2 export: normalised ad-platform capabilities with provenance. |
| [`dataset.ndjson`](https://raw.githubusercontent.com/Rajeev-SG/adpi-data/main/dataset.ndjson) | The same records, one JSON object per line, for streaming consumers. |
| [`history.json`](https://raw.githubusercontent.com/Rajeev-SG/adpi-data/main/history.json) | The fact-history/changelog: one entry per fact version, with its evidence pointer and when it was first seen, last verified and (if superseded) when it stopped being current. |
| [`history.ndjson`](https://raw.githubusercontent.com/Rajeev-SG/adpi-data/main/history.ndjson) | The same history entries, one JSON object per line, for streaming consumers. |

Each publish is a single commit that replaces the previous one, so the branch
always points at the newest dataset and history — nothing accumulates on the
branch itself. The *history* artifact is how change accumulates: it is derived
from the superseded versions the current-state export drops, so a consumer can
see what changed and where a fact was superseded or retired.

History is optional-but-expected: a run that cannot produce it still publishes
the current state, and the run status records whether history was published.
History accrues from first observation only; an empty or short history is an
honest artifact, not a claim that nothing ever changed.

## Coverage and freshness (published, not deferred)

The coverage/gaps and freshness halves of the product are **already delivered on
every publish** as top-level fields of `dataset.json` (#60) — no separate
artifact is needed:

- **`coverage`** — a stated, not implied, manifest: `record_count`, per-`vendor`,
  per-`family` and per-source-class counts, and a `known_gaps` block
  (`unknown_availability`, `unknown_maturity`, `missing_family`). A consumer can
  see why a specific question may be unevidenced instead of reading breadth as
  completeness.
- **`freshness_note`** — the export's own statement that `generated_at` is
  publication time, never evidence freshness.
- **per record** — `last_verified_at` (when the fact was confirmed),
  `last_checked_at` (when the source was last examined) and `reconfirmed` (the
  source was examined after verification and the fact was unchanged).

These ship inside the current-state payload, so they travel with the facts.

## What a record contains

Each capability record carries:

- identity — `id`, `vendor`, `platform`, `name`, `capability_type`;
- behaviour — `control_mode` (`control` / `signal` / `recommendation` /
  `automatic` / `reporting_only` / `not_applicable`) and `maturity`;
- **evidence basis** — `evidence_basis` (`documented` / `account_observed` /
  `vendor_announced` / `unknown`) and **availability** (`supported` /
  `conditional` / `unknown`);
- scope — markets (ISO-3166 alpha-2), objectives, placements, campaign types,
  prerequisites, exclusions and structured `conditions`;
- provenance — source id/URL, a locator pointer, shortened evidence hashes and
  `last_verified_at`.

## Two different times — do not confuse them

- **`generated_at`** (top level) is when this export file was produced.
  Publication time. It says nothing about how current the facts are.
- **`last_verified_at`** (per record) is when that fact was last checked
  against its source. **This is evidence freshness.**

A recently published file can contain facts verified long ago. Read
`last_verified_at` per record; do not use `generated_at` as a freshness claim.
The same honesty applies to `history.json`: its `generated_at` is when the
history export was produced (publication time), not when the underlying facts
were last checked. Use the per-entry `first_seen_at`/`last_verified_at` for the
fact's own timeline. Richer freshness and coverage semantics are tracked
upstream (#60).

## Read it correctly

- **`evidence_basis` is load-bearing.** `documented` means a vendor help page
  states it; `account_observed` would mean it was seen live in a specific
  account. The public documentation pipeline can only produce `documented` or
  `vendor_announced`, with `conditional` or `unknown` availability. Do not read
  `documented` as "available in my account".
- **`availability: supported` requires account observation.** Documentation
  alone can only yield `conditional` (it exists, with conditions the account
  may not meet) or `unknown`. An unqualified "yes" is an account-level claim.
- **Vendors are not equivalent.** A `control_mode` difference is meaningful: a
  hard keyword target, an audience input used by optimisation, and an automatic
  placement are different things. The dataset keeps vendor-native concepts; no
  semantic-equivalence assumption is made across vendors.

## How it is published

Source of truth: [Rajeev-SG/ad-platform-intelligence](https://github.com/Rajeev-SG/ad-platform-intelligence),
specifically `deploy/publish/publish_public_dataset.py`, invoked by the Oracle
run wrapper after a successful acquisition run.

It pushes over SSH with a deploy key scoped to this repository only, so the
credential cannot read or write anything else. **Last-known-good:** a failed run
or a failed publish leaves the previous commit serving unchanged — the pipeline
writes its export atomically and the publisher validates before pushing, so a
broken refresh never replaces good data with bad.

## What is deliberately not here

Facts and provenance only. No credentials, no tokens, no session or task
identifiers, and no verbatim vendor prose beyond a short stable locator pointer.
The publisher re-checks that contract before every write.

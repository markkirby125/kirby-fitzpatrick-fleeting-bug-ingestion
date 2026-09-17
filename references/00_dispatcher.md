# Fleeting Bug Ingestion — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [I learned a system for writing effortlessly](https://www.youtube.com/watch?v=_ribgj7VIGc)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: Observation Decay & the Ledger as Reservoir

Fitzpatrick's system begins from an uncomfortable fact about human cognition: an idea is not lost because it was bad, it is lost because **the moment of noticing is not the moment of keeping**. The insight arrives whole, feels unmissable, and is gone inside a few minutes — not because memory failed but because the insight was never written down while it was still load-bearing. His prescription inverts the instinct of every careful person: *do not evaluate the thought before you capture it*. Capture is instantaneous, mechanical, and free of judgment; evaluation is a second phase, batched, low-stakes, and performed against a pool of material rather than against a fading impression. The lecture's second half is the part engineers consistently skip: the note is worthless in a cabinet, and valuable in a **windmill** — a system where notes are processed and linked until forgotten material surfaces on its own. And the navigation layer (in his case, AI) is permitted to *read and route*, never to *rewrite*. A system that lets the model smooth the notes has destroyed the raw record and replaced evidence with a summary of evidence.

Software engineering has the identical failure, at higher cost, and it arrives through a door nobody guards: **the transient bug**. A race condition that fires once. A connection-pool exhaustion that resolves on retry. A flaky assertion that fails in CI at 03:11 and passes on rerun. A one-off 500 in a p99 tail you happened to be tailing. These observations have a hard decay half-life — within roughly ten minutes the witness has lost the build SHA, the request ID, the exact click sequence, the log window position, and the ambient state of the machine. What survives the decay is not the observation but a **feeling**: *"the API seems flaky."* Every team then pays the reconstruction tax on that feeling, forever, because a feeling cannot be grepped, cannot be tested, and cannot be cited.

The structural error is upstream of remediation. Teams own a *ticket* pipeline, a *root-cause* ritual, an *error budget*, a *postmortem template* — and no **ingestion** pipeline at all. Ingestion is not ticketing. A ticket is a promoted, triaged, assigned decision; the ledger is the pool of raw sightings from which decisions are later drawn. Building the reservoir is expensive and must be cheap per-unit; drawing from it is nearly free. Fitzpatrick's "write effortlessly" is not about writing better sentences — it is about never facing an empty page, because the material was captured continuously and processed asynchronously. The engineering translation is exact: **you do not debug effortlessly by being smarter at 2 a.m.; you debug effortlessly because the exact reproduction was written down in 41 seconds at the moment it happened.**

Nine load-bearing definitions:

1. **Fleeting Observation** — a witness's report of a system state that existed transiently: a one-shot failure, an intermittent assertion, an unexplained latency excursion. Valued *because* it is unrepeatable on demand.
2. **Ledger Entry** — the durable, immutable record of one observation, keyed `BUG-YYYYMMDD-HHMMSS` (UTC timestamp of first observation). The key is the clock, not the concept: it sorts lexically, it is monotonic, it never has to be renegotiated, and it is unguessable only to people who were not there.
3. **Capture Latency (CL)** — elapsed time from first observation to the entry existing on disk. **Target p95 < 60 s.** Above ~2 minutes a witness starts diagnosing; above ~10 minutes the witness starts reconstructing.
4. **The Essential Five** — the mandatory minimum for a valid entry: *what I saw* (byte-exact), *where/when* (env, build SHA, service, request ID, timestamp), *exact steps that preceded it*, *what I expected*, *the raw artefact* (log excerpt / trace / HAR / screenshot + hash).
5. **RAW → TRIAGED** — the only two entry states. RAW means captured-and-unprocessed, and is a **success** state, not a backlog smell. TRIAGED means a reproduction exists or the entry has been linked into a class. There is deliberately no "Diagnosing" state: it is the state in which observations die.
6. **Ledger Yield (LY)** — the share of RAW entries that reach a reproduction. Monitored, never mandated: a low yield is a measurement of the *system's* nondeterminism, not of the team's diligence.
7. **Promotion** — copying an entry into a ticket, PR, test, or ADR. Promotion never deletes or moves the entry; the ticket cites the ledger ID, and the entry gains a `closed_by:` line. The ledger is append-only because its authority *is* its immutability.
8. **Reconstruction Tax** — engineer-hours spent rebuilding an observation that was never written, usually across three people who each hold a different fragment of it. This is the cost line the ledger exists to delete.
9. **Non-Destructive Navigation** — the AI boundary: the model may read the ledger, cluster it, detect duplicates, propose a class, and route owners. It may not rewrite, soften, summarise-in-place, or "clean up" an entry. Any consolidation is a **new derived entry** carrying `derived_from: BUG-…`; originals stay frozen. (See [`../../kirby-fitzpatrick-read-only-vault-isolation/SKILL.md`](../../kirby-fitzpatrick-read-only-vault-isolation/SKILL.md) — same invariant, different vault: there, ground truth; here, ground observation.)

```text
[ANTI-PATTERN: Capture Is Skipped — the Reconstruction Loop]

 14:32:07  p99 spike + one 500 on POST /checkout ..... witnessed by 1 human
           |                                           (witness memory TTL ≈ 10 min)
           v
           "huh, that's odd" ─────────────────────────► NOTHING ON DISK
                │
 15:10:00  standup: "anyone else see the 500?" ....... verbal, unlogged, 0 artefacts
                │
 Day 3      TICKET-4471: "checkout API is flaky" ..... symptom only, no env, no SHA, no steps
                │
 Day 9      closed / "cannot reproduce" ............... 0 knowledge gained, 0 tests added
                │
 Day 34     same class ships to prod ................. n recurrences, 0 learning
                │
           Reconstruction Tax = 3 engineers x 4 h = 12 h, still no repro
           Ledger Yield = 0 / 1


[FLEETING BUG LEDGER: Capture First, Comprehend Later]

 14:32:07  observe ──────────► BUG-20260914-143207   (written 14:32:48Z · 41 s · RAW)
                                ├─ saw       "ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3"
                                ├─ where     prod-eu-1 · build 8f3c21a · req 01J9K3M7QZ
                                │            2026-09-14T14:32:07.412Z
                                ├─ steps     cart(2 items) → remove last → POST /checkout
                                ├─ expected  HTTP 200 + checkout.session.id
                                ├─ artefact  log L88–L94 verbatim · HAR sha256 4c1f9a…
                                └─ ruled out not load (p50 flat 214 ms) · not deploy
                                            (SHA steady 4h17m) · not creds (200 on retry)
                                       │
 14:33:48  RESUME WORK ────────────────┘  (capture is not investigation)
                                       │
 ── async, batched, cheap ─────────────┘
                                       v
 TRIAGE (15 min daily) ──► duplicate of BUG-20260909-… (same class, 4th this month)
                                       ├─► ticket TEST-812 …… references entry, entry stays RAW
                                       ├─► regression test ….. the recorded "steps" become the spec
                                       ├─► ADR candidate ……... 4 occurrences → pool-eviction design
                                       └─► metrics ………...…….. LY 1/1 · CL p95 41 s · RR↓
```

Two failure modes kill a ledger, and both are predictable. The first is **friction at capture** — a mandated severity, component, assignee, and 12-line template turn a 40-second act into a 6-minute act, and the ledger dies within a sprint. The second is **the filing cabinet** — entries go in and stop being read, because nothing ever processes or links them; after 30 days the ledger is a graveyard that people cite but never open. Fitzpatrick's windmill is the fix for both: capture is one cheap gesture, processing is a scheduled separate phase, and the navigation layer surfaces the entry you forgot you wrote (`"4th occurrence of this class in 30 days"`) at exactly the moment it becomes decision-relevant.

---

## 2. Core Transformation Protocols

**2.1 — Capture precedes comprehension. Enforce a 60-second budget.**
Write the entry *before* forming a hypothesis. If you find yourself typing a clause that begins "this is probably…", stop and finish the capture instead. The whole value proposition is that the observation outlives the impression; a capture that took ten minutes of analysis *is* an impression again, and it is now contaminated with a guess that the next reader will mistake for a finding.

**2.2 — The key is the clock, not the concept. `BUG-YYYYMMDD-HHMMSS` is immutable and UTC.**
`BUG-20260914-143207` — always UTC, always the timestamp of **first observation**, never of the write, never renamed, never renumbered. Validate with `^BUG-\d{8}-\d{6}$`. The properties that matter: it sorts lexically into a timeline (`ls ledger/ | tail` is the chronology of what your system actually did), it is unambiguous across timezones and DST, and no human has to negotiate a title. Titles are a triage artifact; keys are a capture artifact. Naming a bug at the moment of sighting is a judgement call and therefore a delay.

**2.3 — The Essential Five, and only the Essential Five.**
`saw` · `where` · `steps` · `expected` · `artefact`. Everything else — severity, component, owner, class, milestone — is triage-time metadata and is *forbidden* at capture time. This is the rule that protects Capture Latency.

**2.4 — Verbatim over paraphrase, byte-exact.**
Paste the string, do not describe it. `ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3` is greppable, indexable, and unambiguous; "a timeout error in the cart service" is none of the three and is also probably subtly wrong. **Verbatim Ratio target: 1.0** — every entry contains at least one byte-exact string from the system, not from the observer.

**2.5 — Record the reproduction attempt as a ratio, never as an adverb.**
Not "reproducible sometimes" but `3/40 attempts, 12:04–12:19 UTC, cmd: scripts/repro-cart-cas.sh, exit 1 once`. A denominator is what separates a reproducible bug from an anecdote; the *same* entry format must serve both.

**2.6 — One observation per entry. Merging is a triage decision.**
Two symptoms in one entry destroy one-to-one traceability, and the merge is irreversible — six weeks later nobody knows which half the fix addressed. Duplicate and cluster *later*, with `linked_to:`/`class:` fields on a new derived entry, originals untouched.

**2.7 — Capture the negative space.**
Record what you already ruled out, with the evidence that ruled it out (`not load — p50 flat 214 ms`, `not deploy — SHA steady 4h17m`). The ruled-out set is the single highest-value field at triage, because it is the only field a fresh investigator cannot reconstruct.

**2.8 — Never orphan the raw artefact.**
If the artefact is a log line, inline it. If it is a HAR, screenshot, or trace, store it at a path and record `sha256` plus the retention window. If it is ephemeral (a live buffer, a scrollback, a stream), the excerpt goes in the entry *now* — the artefact has the shortest half-life in the entire system.

**2.9 — Processing is a separate, scheduled, low-stakes phase.**
Daily 15-minute triage or a weekly batch: dedupe, link, promote, retire. Never process during capture; never capture during triage. RAW is a legitimate resting state, not a failure — an entry may sit RAW for weeks and that is the system working.

**2.10 — AI navigates the ledger; it never mutates it.**
The model may read, cluster ("these 4 entries share a class"), propose owners, and draft tickets. It may not rewrite, summarise in place, or "tidy" prose, and agent-authored text is excluded from the authoritative index. Any consolidation is a new entry with `derived_from: BUG-…`. This is the direct instance of [`../../kirby-fitzpatrick-read-only-vault-isolation/SKILL.md`](../../kirby-fitzpatrick-read-only-vault-isolation/SKILL.md): if generated prose about the ledger gets re-ingested as ledger content, the ledger slowly becomes a transcript of the model's previous guesses and stops being a record of your system. Pair the ingestion surface with [`../../kirby-fitzpatrick-codebase-navigation-router/SKILL.md`](../../kirby-fitzpatrick-codebase-navigation-router/SKILL.md) so that navigation routes *to* the raw entries rather than replacing them.

**2.11 — Append-only. Corrections are new entries.**
To correct an entry, write `BUG-YYYYMMDD-HHMMSS` that cites the superseded ID and states the correction. Editing history makes the ledger unusable as evidence in exactly the situation it exists for: "what did we actually know on the 14th?"

**2.12 — Quantify the pipeline, or it silently decays.**
`CL`, `LY`, `Verbatim Ratio`, `RAW Ratio`, `Promotion Traceability` — see §2.4 of the metrics block below. Unmeasured ingestion dies within one quarter, every time.

| Metric | Definition | Target / reading |
| :--- | :--- | :--- |
| **Capture Latency (CL)** | first observation → entry on disk | **p95 < 60 s**; > 2 min means friction is killing the ledger |
| **Verbatim Ratio (VR)** | entries containing a byte-exact system string ÷ all entries | **1.0**, non-negotiable |
| **Ledger Yield (LY)** | RAW entries reaching a reproduction | tracked, not mandated; a measure of the *system's* determinism |
| **RAW Ratio (RR)** | open RAW entries ÷ open entries | a legitimate ADR input — a high RR is an architecture fact, not a team failing |
| **Promotion Traceability (PT)** | bug work items (tickets/PRs/ADRs) citing a ledger ID ÷ all bug work | **1.0**; an uncited fix is indistinguishable from an unrequested change |
| **Reconstruction Tax** | engineer-hours spent rebuilding uncaptured observations for one incident | drive to 0 |

**Entry template (the canonical shape):**

```markdown
BUG-20260914-143207                      status: RAW        captured: 2026-09-14T14:32:48Z (+41s)
────────────────────────────────────────────────────────────────────────────────────────────
saw        `ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3` on the prod-eu-1 log stream
where      prod-eu-1 · build 8f3c21a · req 01J9K3M7QZ · 2026-09-14T14:32:07.412Z
steps      cart(2 items) → remove last item → POST /checkout  → 504 after 30 s (1 occurrence)
expected   HTTP 200 + body {checkout.session.id}
artefact   app.log L88–L94 (verbatim, below) · HAR sha256 4c1f9a… (retain 30 d)
ruled out  load (p50 flat 214 ms across the hour) · deploy (SHA steady 4h17m) · creds (retry 200)
hypothesis — NOT EVIDENCE: connection-pool eviction race in cart → cas
next       triage: run scripts/repro-cart-cas.sh (40 attempts) and count the ratio

--- verbatim artefact ---
14:32:07.412 WARN  cart  ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3 conn=pool-3
14:32:07.413 ERROR cart  checkout.session.failed reason=upstream_timeout
```

```bash
# capture in <60 s, zero ceremony, no field beyond the Essential Five
bug new \
  --saw      'ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3' \
  --where    'prod-eu-1@8f3c21a req=01J9K3M7QZ' \
  --steps    'cart(2) -> remove last -> POST /checkout' \
  --expected '200 + checkout.session.id' \
  --artefact <(tail -n 7 app.log)      # -> ledger/BUG-$(date -u +%Y%m%d-%H%M%S).md · status RAW
```

| # | Anti-pattern | Why it destroys the observation | Clean replacement |
| :--- | :--- | :--- | :--- |
| 1 | *"It's probably a race in the pool"* written at capture | A hypothesis in the `saw` slot is inherited as fact by the next reader and never re-tested | `saw:` byte-exact string; hypothesis moved to a lower field, explicitly labelled `NOT EVIDENCE` |
| 2 | *"API seems flaky"* | Ungreppable, no env, no SHA, no steps — the canonical pre-decay summary | Paste the verbatim string + `env@SHA` + steps + a recurrence ratio |
| 3 | Slack DM: *"did you see that 500?"* | Private, ephemeral, unindexed, unfindable in 3 weeks | Entry first (≤ 60 s), then the DM links the entry ID |
| 4 | Screenshot-only capture | Not greppable, not diffable, OCR-hostile, invisible to every search | Verbatim text excerpt **plus** the image path and `sha256` |
| 5 | *"Reproduced it a few times"* | No denominator; nobody can distinguish 2/3 from 2/300 | `3/40 attempts, 12:04–12:19 UTC, cmd: repro-cart-cas.sh` |
| 6 | Two symptoms merged into one entry | Breaks one-to-one traceability; the merge is irreversible and un-auditable | One observation per entry; cluster at triage via `class:` / `linked_to:` |
| 7 | Editing a RAW entry later that day | Rewrites history; destroys the entry's standing as evidence | New entry citing the superseded ID — the ledger is append-only |
| 8 | AI asked to "clean up and summarise the ledger" | Generated prose becomes the next session's evidence; confidence rises while accuracy does not | AI reads/clusters/routes only; consolidation emits a new `derived_from:` entry |
| 9 | Closed as *"cannot reproduce"*, entry never written | The observation is deleted rather than deferred, so the class is invisible forever | Status `RAW/UNREPRODUCIBLE` with a retirement date; the class stays countable |
| 10 | Mandatory severity/component/assignee at capture | Pushes CL past 60 s; the ledger is abandoned inside one sprint | Essential Five only; all routing metadata is added at triage |

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews — The Reviewer as a Witness, Not an Investigator

Reviewers witness more transient behaviour than anyone else in the organisation — CI flakes, one-off 5xx lines in a pre-merge run, an assertion that failed once and passed on rerun — and capture almost none of it, because a review comment *feels* like the wrong container for a sighting. It is the right container for a *decision*, and a sighting is not a decision. The protocol: capture first (≤ 60 s, Essential Five), then comment citing the ID.

```markdown
<!-- review comment -->
**Observation** → `BUG-20260914-143207` (RAW)
Seen once in CI run 8891: `ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3` (log L88–L94 pasted there).
Not blocking — this diff does not touch `pool.ts`. Flakiness class is captured, not diagnosed here.

**Blocking** → `BUG-20260914-161044` (TRIAGED, 3/40 attempts, repro at `scripts/repro-cart-cas.sh`)
This change moves `evict()` outside the lock, which is the state the repro depends on.
Please attach a 40-attempt run on the new revision to close the loop.

**Raw observations carried forward (not silently absorbed):** `BUG-20260912-091803` (RAW), `BUG-20260913-171522` (RAW)
```

Binding rules that keep the review honest:

- **A review is a unit of work, not a debugging session.** The diff is the object under review; a 20-minute investigation inside a review thread is how a sighting becomes nobody's job. Capture, cite, move on. Deep work happens at triage, where it is scheduled and costed.
- **A RAW observation never blocks a merge; it must be *stated*.** Blocking on an unclassified sighting is how teams weaponise uncertainty. But silently absorbing it — merging while an entry sits RAW and unmentioned — is how the class disappears.
- **Reviewer quotes are byte-exact.** Never paraphrase a failing assertion, a heap dump line, or a status code. The reviewer is frequently the *only* witness to the exact string; paraphrasing is the whole failure.
- **"Looks racy" is not a finding.** It is a hypothesis with no denominator and no artefact. Capture the sighting, or write nothing — an unbacked guess in a review thread outlives its context and misleads a future reader. (See [`../../kirby-fitzpatrick-semantic-gap-hunter/SKILL.md`](../../kirby-fitzpatrick-semantic-gap-hunter/SKILL.md): name the gap, do not paper it with plausible language.)
- **A review should cite first sightings it produced.** If a reviewer's own observation created entries, the review block lists them, so the ledger has a provenance trail back to the change that exposed the behaviour.

### 3.2 PR Descriptions — The Ingestion Block

The PR body is where the ledger stops being a private log and becomes durable evidence, because the reviewer's attention is the scarcest resource in the loop and the on-call engineer six months later has none of it. The **Ingestion block** goes at the top, above "What changed", so the reproduction claim is falsifiable in fifteen seconds — before anyone is invested in the fix. This is the bug-side analogue of the cold-reader audit in [`../../kirby-fitzpatrick-cold-reader-pr-auditor/SKILL.md`](../../kirby-fitzpatrick-cold-reader-pr-auditor/SKILL.md).

```markdown
## Ingestion
| Field | Value |
|---|---|
| Ledger entries | `BUG-20260914-143207` — **closed** · `BUG-20260909-071402` — linked (same class) |
| Captured | `2026-09-14T14:32:48Z` (+41 s after first observation) · `status_before: RAW` |
| Reproduction | **3/40 attempts** on `8f3c21a` (`scripts/repro-cart-cas.sh`, 12:04–12:19 UTC) |
| Mechanism | **Unconfirmed.** Fix addresses pool eviction inside `cas`; 40/40 pass after → hypothesis, not proof |
| Regression test | `cart_cas_timeout_test.go` — red on `8f3c21a`, green on this revision (recorded steps = test spec) |
| Artefacts retained | `app.log` L88–L94 (verbatim in ledger) · HAR `4c1f9a…` · 30-day window |
| Still RAW after this change | `BUG-20260913-171522` (unrelated latency excursion) — carried forward, not absorbed |
| Verbatim string fixed | `ERR_CONN_CAS_TIMEOUT svc=cart attempts=3/3` — searchable in the ledger before and after |

## What changed
`evict()` moved outside the connection lock. No public surface, no schema, no migration.

## Not verified
The eviction race itself was never observed directly; the 40/40 result is consistent with the
fix and also consistent with a load-correlated trigger that simply did not occur in the window.

## Ledger effects on merge
`BUG-20260914-143207` gains `closed_by: PR#812 @3a77e02`. The RAW entry stays RAW-forever-readable;
no entry is edited or deleted. Class counter for `cas-pool` → 4 occurrences this month → ADR candidate.
```

**Binding rules that keep the block honest:**

- **Every bug-fix PR names at least one ledger ID.** If no entry exists, the PR *backfills* one at merge time, dated at first observation and flagged `captured: retro` — because a fix with no observation record is indistinguishable from an unexplained change, and the class counter needs the row.
- **The reproduction is the test.** The entry's `steps` field becomes the failing test, and the test name carries the ledger ID (`TestCartCasTimeout_BUG20260914143207`). A test written from a hypothesis tests the hypothesis; a test written from recorded steps tests the observation.
- **Non-reproducibility is disclosed, not disguised.** *"Mechanism unconfirmed; 40/40 pass post-fix"* is the honest and correct sentence. Pretending a probabilistic fix has been proven is the exact place where a settled-looking PR imports an unresolved fault for a year.
- **No entry is deleted on merge.** Promotion is a copy; the entry gains `closed_by:` and stays. The ledger must be able to answer "what did we see on the 14th, and did we explain it?" for as long as the code lives.
- **Carried-forward RAW entries are listed explicitly.** A merge that absorbs an unexplained sighting makes the class invisible, and invisible classes are re-discovered as incidents.

### 3.3 Architecture RFCs / ADRs — The Ledger as the Evidence Base

The ledger is the only legitimate substitute for anecdote in a design decision. An RFC's Context section written in adjectives (*"we've seen intermittent instability under load"*) is a story, and stories are how teams build infrastructure for bugs that never happened. The ledger forces the sentence to become countable: *"14 entries in this class over 90 days, 11 of them RAW."* That sentence contains two decision-relevant facts — the failure mode is chronic, and its mechanism is unknown — and it justifies two different investments (remediation **and** ingestion), which adjectives cannot distinguish.

```markdown
# ADR-021 — Ingest before you remediate: the Fleeting Bug Ledger

## Context / Trigger
`cas-pool` class: 14 ledger entries in 90 days, 11 still RAW.
- `BUG-20260909-071402` 3/40 repro · `BUG-20260914-143207` 3/40 repro · 9 entries with no repro at all
- RAW Ratio 0.79 — **we cannot reproduce ~4 in 5 of what we observe**
- Reconstruction Tax for INC-4477 alone: 3 engineers × 4 h, no reproduction produced
- The only durable asset from that incident was a paraphrase ("checkout is flaky") that no one can grep

## Decision
Ingestion is a first-class pipeline, upstream of ticketing.
- Every transient observation is captured as `BUG-YYYYMMDD-HHMMSS` (UTC, first-observation clock),
  append-only, Essential Five only, Capture Latency p95 < 60 s.
- Capture and triage are separate phases; RAW is a valid resting state.
- Promotion is a copy: tickets/PRs/tests/ADRs cite ledger IDs, never replace entries.
- AI is a **navigation layer only** (read, cluster, route) and is never allowed to rewrite an entry;
  consolidated views are new entries with `derived_from: BUG-…` and are excluded from the
  authoritative index. Enforcement: `ledger/**` is CODEOWNERS-protected, append-only, no
  generator commits.
- Retention: entries forever · raw artefacts 30 d · retirement is `RAW/UNREPRODUCIBLE` + a date,
  never a deletion.

## Consequences
- Cost: ~45 min/engineer/month (≈ 40 s × sightings) + 15 min/day triage — measured, not estimated.
- Measured cost of inaction: 12 engineer-hours for one incident, 0 reproductions, RAW Ratio 0.79.
- Verification: `ledger:check` asserts key regex `^BUG-\d{8}-\d{6}$`, UTC monotonic keys, 100 % of
  entries contain a verbatim system string, 0 agent-authored mutations, and 0 promotions missing a
  `closed_by:`/`linked_to:` reference. Target: RAW Ratio < 0.4 within two quarters.
- Ledger-anchored success signal: `cas-pool` occurrences fall below 1/quarter → revisit the pool design.
- Reverses if: Capture Latency p95 exceeds 2 min (friction) or Ledger Yield stays at 0 for two
  consecutive quarters with no triage activity — in which case fix the pipeline, do not delete it.
- Expiry: re-evaluate 2027-01-01 (@oncall-lead) against measured CL, LY, RR.
- Non-goals: this ADR does not authorise severity/owner fields at capture, does not make the ledger
  a ticket system, and does not permit any agent to author `ledger/**`.
```

**Rules for the RFC side of the pipeline:**

- **No entry, no evidence.** If the motivating incident is not in the ledger, it is a story. Cite IDs with dates and counts, and state the reproduction ratio per entry. An RFC whose Context is a paraphrase of a paraphrase has a provenance problem before it has a decision problem.
- **RAW Ratio is a first-class design input.** *"79 % of what we see is unreproducible"* is not an admission of weakness; it is the argument for observability, deterministic fixtures, or chaos experiments — and it is the architecture fact that a ticket-only backlog permanently hides.
- **Raw artefacts get a retention and ownership decision.** Store paths, hashes, and a window (30 d), and name the owner. An undocumented artefact store is a leak with a budget.
- **State the ledger effect of the proposal.** Any RFC introducing infrastructure must say how it moves Capture Latency, Ledger Yield, or RAW Ratio. A design that improves none of the three is unmeasured improvement.
- **Reversibility and expiry are anchored to measured counts.** *"Revisit if this class drops below 1 occurrence/quarter for two consecutive quarters"* is checkable in the ledger; *"revisit if things improve"* is folklore.
- **Non-goals are mandatory** — precisely because the ledger's flexibility invites every reviewer to import severity gates, SLA fields, and mandatory assignees, which is the friction that kills ingestion. Substituting any of them silently is a scope change, not an improvement; see [`../../kirby-fitzpatrick-substance-first-refactoring/SKILL.md`](../../kirby-fitzpatrick-substance-first-refactoring/SKILL.md) for the discipline of changing structure without changing the observation.

---

## 4. Verification Checklist

- [ ] **The entry exists, and it existed in time.** Every observation produced a `BUG-YYYYMMDD-HHMMSS` file whose key matches `^BUG-\d{8}-\d{6}$`, whose timestamp is UTC and corresponds to **first observation** (not to the write), and whose `captured:` line shows a delta inside the budget. Verified across the sampled entries: **Capture Latency p95 < 60 s**. No observation in this change's scope was left in a chat message, a meeting, or a memory.
- [ ] **All five essential fields are present and `saw` is byte-exact.** Sampled entries contain `saw` (verbatim system string, **Verbatim Ratio = 1.0**, no paraphrase, no translation, no editorial shortening), `where` (env + build SHA + request ID + timestamp with offset), `steps` (ordered and re-runnable), `expected`, and an `artefact` (inline excerpt or path + `sha256` + retention). Ruled-out hypotheses are recorded **with the evidence that ruled them out**, and any hypothesis present is explicitly labelled `NOT EVIDENCE`.
- [ ] **The ledger is append-only, immutable, and has no agent-authored mutations.** `git log` on `ledger/**` shows no rewriting, softening, or deletion of an entry; every correction is a new entry citing the superseded ID; every consolidation is a separate entry carrying `derived_from: BUG-…` and excluded from the authoritative index. AI participation in this change was navigation only — read, cluster, route, draft — and the model produced **zero** commits touching `ledger/**`.
- [ ] **Every promotion is traceable and no RAW entry was silently absorbed.** Each ticket, PR, test, and ADR in scope cites its ledger ID; the reproduction ratio for each promoted entry is stated with a denominator (`3/40 attempts`, not "sometimes"); non-reproducibility is disclosed rather than implied as proven; the recorded `steps` became the regression test, which was shown **red on the recorded SHA and green after**; and every merge lists the RAW entries it carried forward instead of absorbing them. **Promotion Traceability = 1.0.**
- [ ] **The pipeline is measured, and the ledger is being processed.** `ledger:check` ran with a reported result (key regex, UTC monotonicity, verbatim presence, zero generator mutations, zero unreferenced promotions), and the four operating numbers are reported — **CL, LY, Verbatim Ratio, RAW Ratio** — with a named owner and a reviewed date for surfacing links (`"Nth occurrence of this class in M days"`). An entry that has sat RAW past its review date is either triaged, explicitly retired as `RAW/UNREPRODUCIBLE` with a date, or escalated — never quietly edited, and never deleted to make the ledger look clean.
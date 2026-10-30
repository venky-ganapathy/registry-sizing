# Registry Hardware Sizing and Estimation — Methodology Reference

**Read this first if you're revising `index.html` without the conversation that built it.**

This file exists because the sizing logic in `index.html` was built up over many
chat turns, each grounded in a specific real benchmark or a specific piece of
real source code (cited inline as JS comments next to each constant/formula).
This document is the durable index of *all* of that — every input, every
constant, every formula, every "measured" vs "assumed" call, and every known
simplification — so a future revision can be made correctly without having
read the original conversation.

**When you change something here, update this file in the same commit.** A
stale methodology doc is worse than none — it actively misleads the next
person (possibly future-you).

## 1. What this tool is

A single-page HTML app (`index.html`), published as a Claude Artifact and also
deployed as a GitHub Pages site from this repo
(`https://venky-ganapathy.github.io/registry-sizing/`), that lets an OpenG2P
Registry implementation partner enter their deployment's expected scale
(Tab 1: *Implementation Partner Inputs*) and see a derived Kubernetes
pod/database/storage sizing recommendation with an AWS instance suggestion
(Tab 2: *Estimation and Sizing*). Every pod and storage box on Tab 2 is
clickable and shows a popover with its sizing rationale, a `measured` or
`assumed` tag, and a free-text note.

The live Claude Artifact (private, same URL re-published on every change):
`https://claude.ai/artifact/2rCDSyrQkzFoMzYm2gsKYT`

## 2. Measured vs Assumed — what the tag means

- **`measured`**: the dominant/dynamic driver of this number traces to a real
  benchmark run (a Locust load test with a results report) or is computed
  live from real Tab 1 inputs, rather than being a fixed historical constant
  someone picked once. A box can still be tagged `measured` even if a *minor*
  sub-component inside it is a flat placeholder — see the Direct-Attached
  Storage precedent (its dominant component, the main register DB, is real;
  that's enough to call the whole box `measured`).
- **`assumed`**: either a genuine guess (no benchmark exists), or a
  *simplification* of real measured numbers — e.g. a blended/averaged curve
  standing in for several real curves because Tab 1 only collects one
  aggregate input, not a per-pipeline breakdown. Several `assumed` tags in
  this tool are "every underlying number is measured, but the blend/average
  itself is a modeling choice" — that distinction is called out explicitly
  everywhere it applies.

## 3. Tab 1 inputs — what exists and what it feeds

| Input id | Label | Default | Feeds |
|---|---|---:|---|
| `inNamespace` | Kubernetes namespace | `perftest` | cosmetic (pod names) |
| `inRegistryName` | Registry name | Farmer Registry | cosmetic (pod names, page) |
| `inPrimaryRecords` | Primary Register records | 1,000,000 | `totalRegisterRecords`, `totalTableRecords`, `totalDocuments` |
| `inSecondaryRegisters` | Number of Secondary Registers | 1 | `totalSecondaryRecords` → `totalRegisterRecords` |
| `inRatioSecondaryRecords` | Secondary records per Primary record | 0.41 | `totalSecondaryRecords` |
| `inSupportingTables` | Number of Supporting Tables | 14 | `totalTableRecords` |
| `inRatioTableRecords` | Supporting Table records per Primary record | 1.33 | `totalTableRecords` |
| `inIntakeForms` | Number of Intake Forms | 3 | informational only (not yet wired into a calculation) |
| `inMaxFileSizeKB` | Max file size per document (KB) | 384 | `commonsMinioGB` |
| `inFilesPerRecord` | Files per Primary Register record | 4 | `totalDocuments` → `commonsMinioGB` |
| `inConcurrentUsers` | Concurrent Staff UI users | 20 | `staffApiPods` |
| `inStaffTxnPerDay` | Staff UI transactions/day (CR + Intake combined) | 500 | `totalDailyTxns`, Audit Manager DB (staff-only) |
| `inStaffSearchTxnPerDay` | Staff UI search transactions/day | 500 | Loki, Audit Manager DB |
| `inNumPartners` | Number of partners | 2 | informational only |
| `inPartnerTxnPerDay` | Partner transactions/day (CR + Intake combined) | 1,000 | `totalDailyTxns`, partner-api ingest pods |
| `inPartnerSearchTxnPerDay` | Partner search transactions/day | 5,000 | Loki, Kafka, partner-api search pods |
| `inLogRetention` | Log retention (days) | 15 | `lokiGB` |
| `inSizingYears` | Years to size storage for | 5 | `txnDbGrowthGB`, `auditMgrGB` |

Every input id above must appear in three places in `index.html`, or the
input silently does nothing: the `<input id=...>` element, a `num()`/`txt()`
call in `readInputs()`, and the Tab-1 change-listener array near the bottom of
the file. (This is checked by a small Python script before every publish —
see §8.)

## 4. Derived model — one subsection per Tab 2 box

All of this lives in `deriveModel()`. `totalCrPerDay` and `totalIntakePerDay`
are both `(inStaffTxnPerDay + inPartnerTxnPerDay) / 2` — Tab 1 collects one
combined CR+Intake counter per source (staff/partner) with **no split**
between the two, so an even 50/50 split is assumed throughout. This is the
single most consequential simplification in the whole tool — if a registry's
real CR:Intake ratio is far from 50/50, every formula below that uses these
two terms should be revisited.

### 4.1 Staff-API pods (`staffApiPods`)

```
staffCaps     = interpolate(STAFF_CAPS, STAFF_TIERS, totalRegisterRecords)   // concurrent users supported, pod 1/2/3
staffNeed     = inConcurrentUsers / (1 - HEADROOM)
staffApiPods  = podsNeeded(staffCaps, staffNeed)
```
- `STAFF_TIERS = [10150268, 50000264]` — the two register-row volume tiers the staff-api benchmark was run at.
- `STAFF_CAPS = [[184,319,453],[164,329,369]]` — real concurrent users supported at pod 1/2/3, at each tier (e2e ingress). Note the *smaller* tier supports *more* concurrent users at pod-1 (184 vs 164) — real measured behavior, not a typo.
- `HEADROOM = 0.30` — fixed 30% headroom applied to every capacity check in this tool. No Tab 1 input for it.
- Source: staff-api performance benchmark (`performance-benchmarks/staff-api/final-report-summary.md`).
- Tag: `measured`.

### 4.2 Partner-API pods (`partnerApiPods`)

```
partnerSearchNeedRate = (inPartnerSearchTxnPerDay / 86400) / (1 - HEADROOM)
partnerIngestNeedRate = (inPartnerTxnPerDay / 86400) / (1 - HEADROOM)
partnerApiPods = max(
  podsNeeded(PARTNER_SEARCH_RATES, partnerSearchNeedRate),
  podsNeeded(PARTNER_INGEST_RATES, partnerIngestNeedRate)
)
```
- `PARTNER_SEARCH_RATES = [3.33, 5.36, 5.10]` — sync RPS at pod 1/2/3, averaged across the `expression`/`graphql`/`predicate` DCI range query types. Single Farmer Search using ID excluded as an outlier (10-30x higher RPS, a trivial 1-record lookup). **Sync**, not async, is used because the benchmark report states async's bottleneck is the downstream Kafka/WebSub publish path, not the partner pod itself — sync is what actually stresses this pod.
- `PARTNER_INGEST_RATES = [85.74, 141.13, 187.91]` — raw Ingest API RPS, one curve, no blending.
- Max (not sum) because no combined-load benchmark exists; each curve was measured against its endpoint type in isolation.
- Source: `performance-benchmarks/partner-api/final-report-summary.md`.
- Tag: `measured` (the pod itself); the range-type blend is called out as `assumed` in the popover even though every underlying number is real.

### 4.3 Celery-worker pods (`celeryWorkerPods`) — drain-time model

Sized to a **12-hour drain-time goal**, not a raw throughput ceiling. A day's
Change Request + Intake Submission volume populates 7 queues:

```
celeryQueueItemsPerDay =
    totalCrPerDay        // 1. Change Request ingest      (change_request_ingest)
  + totalIntakePerDay    // 2. Intake Submission ingest    (intake_register_ingest)
  + totalIntakePerDay    // 3. Intake -> ID Generation      (functional_id_allocation)
  + totalCrPerDay        // 4. Change Request -> Dedup      (dedup_cr_vs_register)
  + totalIntakePerDay    // 5. Intake -> Dedup              (dedup_intake_vs_register / dedup_intake_vs_intake)
  + totalCrPerDay + totalIntakePerDay // 6. CR + Intake -> Completion Score (completion_score)
  + totalIntakePerDay    // 7. Intake -> Score Compute      (score_compute)

celeryNeedRate  = (celeryQueueItemsPerDay / (DRAIN_GOAL_HOURS * 60)) / (1 - HEADROOM)
celeryWorkerPods = podsNeeded(CELERY_BLENDED_DRAIN_RATES, celeryNeedRate)
```
- `DRAIN_GOAL_HOURS = 12` — explicit user requirement: a day's queue volume must fully drain within 12 hours.
- `CELERY_BLENDED_DRAIN_RATES = [1249, 2179, 2179]` rows/min at pod 1/2/3. **This is the real "Blended" benchmark run** — all 14 measured queue types draining *concurrently* against the same worker pool (not 14 isolated per-queue curves summed, which would overstate capacity since isolated runs never actually contend for the same CPU/DB connections). Back-computed from `celery/final-report-summary.md` §Blended:
  - Pod-1: 112 min window, 4 stragglers all ≥99.7% done → ~139,880 rows / 112 min ≈ **1249**
  - Pod-2: 64 min window, same 4 stragglers all ≥98.6% done → ~139,440 / 64 ≈ **2179**
  - Pod-3: 73 min window; `change_request_ingest`/`ingest_data` regress hard (1,166 and 1,091 of 10,000 rows still in-progress — the report's own headline finding: these two share one beat producer and get *worse*, not better, at pod-3) → raw ≈ **1883**, which is *below* pod-2.
  - The raw pod-3 number (1883) is **floored at pod-2's rate (2179)** in the array used by `podsNeeded()` — the model never claims pod-3 drains faster than pod-2 (it measurably doesn't), and flooring also means the extrapolation-beyond-pod-3 logic sees a flat (not negative) marginal gain and caps at 3 pods rather than inventing a slope the benchmark never showed.
  - **Implication for revisions**: if a future registry's computed need exceeds pod-2's demonstrated drain rate, the tool caps at 3 pods and that's a deliberate signal, not a bug — the benchmark found a real contention ceiling (the shared beat-producer issue), not a "throw more pods at it" problem. Don't "fix" this by letting it extrapolate past 3 without first re-reading the Blended section of the report.
- Why "blended" and not per-queue curves: the user explicitly asked for this (see conversation: *"You should use Celery blended run drain time to estimate the number of pods required"*) — it's the one real benchmark where every queue type actually competes for the same worker pool, which matches production reality.
- Simplification flagged: the Blended benchmark mixed all 14 scenario types with an **equal** 10,000-row backlog each; our real workload's mix (3 queue-types weighted by CR volume, 5 by Intake volume) is not equal-weighted the way the benchmark's was. Using one flat rows/min figure for an arbitrary task-type mix is an approximation inherited from that benchmark shape, not a new one introduced here.
- Source: `performance-benchmarks/celery/final-report-summary.md` (and the farmer-registry repo's own copy under `farmer-registry/performance-testing/documentation/celery/`).
- Tag: `measured`.

### 4.4 Celery-beat — fixed at 1 pod, never scaled (to avoid duplicate task scheduling). Tag `measured`.

### 4.5 farmer_registry (main) database (`mainDbGB`, folded with `tableStorageGB` and `txnDbGrowthGB`)

```
totalSecondaryRecords = inSecondaryRegisters * inRatioSecondaryRecords * inPrimaryRecords
totalRegisterRecords  = inPrimaryRecords + totalSecondaryRecords
mainDbGB              = lerp(STORAGE_POINTS, totalRegisterRecords)   // extrapolates outside the two measured points

totalTableRecords  = inSupportingTables * inRatioTableRecords * inPrimaryRecords
tableStorageGB     = (totalTableRecords * ASSUMED_BYTES_PER_TABLE_ROW) / 1e9

dailyTxnDbGrowthBytes = totalCrPerDay * MEASURED_UNIT_SIZES.dbGrowthBytesPerChangeRequest
                      + totalIntakePerDay * MEASURED_UNIT_SIZES.dbGrowthBytesPerIntakeSubmission
txnDbGrowthGB = (dailyTxnDbGrowthBytes * 365 * inSizingYears) / 1e9

farmer_registry DB size = mainDbGB + tableStorageGB + txnDbGrowthGB
```
- `STORAGE_POINTS = [[10150268, 139.3], [50000264, 715.1]]` — real register row counts → total measured Postgres GB, at the staff-api benchmark's two volume tiers.
- `ASSUMED_BYTES_PER_TABLE_ROW = 200` — flat guess, no benchmark backing it (unlike almost everything else in this tool). Candidate for replacement if a real Supporting-Table-row-size measurement ever becomes available.
- `MEASURED_UNIT_SIZES.dbGrowthBytesPerChangeRequest = 4980`, `.dbGrowthBytesPerIntakeSubmission = 43078` — real pg_total_relation_size before/after deltas, from 10,000 seeded CRs and 10,000 seeded intake submissions against the farmer-registry perf-test database.
- `inSecondaryRegisters` defaults to 1 (the Farmer Registry's own Household secondary register), `inRatioSecondaryRecords` to 0.41 (4,150,063 households / 10,150,268 farmers, real seed counts), `inSupportingTables` to 14 (only the tables that actually scale with farmer count — ~69 static config/lookup tables are excluded), `inRatioTableRecords` to 1.33 (real measured ratio).
- Tag: `measured` (dominant drivers are real benchmarks/real seed-ratio measurements; `tableStorageGB`'s flat per-row guess is the one weak link, tolerated under the DAS/NFS precedent in §2).

### 4.6 Audit Manager database (`auditMgrGB`)

**Rebuilt from the real `AuditMiddleware` source**, not from request-body
sizes — the middleware never reads the request or response body. See the
long comment block directly above `AUDIT_EVENT_BYTES` in `index.html` for the
full reasoning; summary:

```
staffCrPerDay      = inStaffTxnPerDay / 2
staffIntakePerDay  = inStaffTxnPerDay / 2       // staff-only -- partner-api traffic excluded, see below

dailyAuditEvents =
    staffCrPerDay * 2                                          // create_change_request + submit_task_decision
  + staffIntakePerDay * (AUDIT_EVENTS_PER_INTAKE_SUBMISSION_CREATE + 1) // 10 section saves + submit_task_decision
  + inStaffSearchTxnPerDay                                     // search_in_a_register

auditMgrGB = (dailyAuditEvents * AUDIT_EVENT_BYTES * 365 * inSizingYears) / 1e9
```
- `AUDIT_EVENT_BYTES = 720` — every audited call's CloudEvent is near-constant size (~700-735 bytes measured by reconstructing the exact `_build_event`/`_build_actor` shape from the real staff-api `audit_middleware.py` source) regardless of action, because **no business payload is ever captured** — only actor/action/outcome/context.
- `AUDIT_EVENTS_PER_INTAKE_SUBMISSION_CREATE = 10` — one `save_intake_form_submission` HTTP call per form section (10 sections in the real `SECTION_DEFS`), not 1 call per submission.
- **Partner-API volume is excluded entirely.** Grepped the whole `registry-platform` partner-api source: no controller anywhere sets `request.state.audit_actor`, so per partner-api's own `audit_middleware.py` docstring, a successful partner-api call is **not audited at all** today — only failures are (anonymous-failure policy). If that ever gets wired up (a controller starts setting `audit_actor`), this formula needs a partner-side term added.
- Source: `registry-platform/apis/openg2p-registry-staff-api/src/openg2p_registry_staff_api/audit_middleware.py` and the partner-api equivalent; `openg2p-documentation/platform/platform-services/audit-manager/technical-architecture.md` for the `audit_events` table schema (`details JSONB` holds the actor/context extras, not a resource payload).
- Retention assumption: sized for the full `inSizingYears` horizon (Audit Manager's own 7-year default partition retention, per its technical-architecture.md — an accumulate-style box like the main DB, not a rolling window like Loki/Kafka).
- Tag: `assumed` — grounded in real middleware/locust source, but not an actual Audit Manager benchmark run (none exists yet).

### 4.7 Loki (`lokiGB`, part of the NFS `loki-minio` partition)

```
dailyLogBytes = inStaffSearchTxnPerDay * MEASURED_UNIT_SIZES.logBytesPerStaffSearch
              + inPartnerSearchTxnPerDay * MEASURED_UNIT_SIZES.logBytesPerPartnerSearch
              + totalCrPerDay * MEASURED_UNIT_SIZES.logBytesPerStaffChangeRequestCreation
              + totalIntakePerDay * MEASURED_UNIT_SIZES.logBytesPerStaffIntakeSubmission
              + totalIntakePerDay * MEASURED_UNIT_SIZES.logBytesPerCeleryIntakeSubmission
lokiGB = (dailyLogBytes / 1e9) * inLogRetention
```
All five `MEASURED_UNIT_SIZES` log figures are real Loki byte-deltas from the
same 10,000-request seeded runs used for `dbGrowthBytesPer*` above:
`logBytesPerStaffSearch=855`, `logBytesPerPartnerSearch=2175`,
`logBytesPerStaffChangeRequestCreation=1347`,
`logBytesPerStaffIntakeSubmission=866`,
`logBytesPerCeleryIntakeSubmission=9401`. The CR-creation/intake-submission
figures were only ever measured on staff-api; partner-api is **assumed** to
log the same amount for the same operation (no partner-api measurement
exists for these two). Rolling window (retention days), not accumulate-forever.
Tag: `assumed` (per-figure measured, but the model as a whole — and the
staff≈partner logging assumption — is called `assumed`, consistent with how
Loki's sibling Kafka/commons-minio partitions are tagged in this file).

### 4.8 commons-minio (`commonsMinioGB`)

```
totalDocuments = inPrimaryRecords * inFilesPerRecord
commonsMinioGB = (totalDocuments * (inMaxFileSizeKB / 1024)) / 1024   // KB -> MB -> GB
```
No per-document measurement exists; `inFilesPerRecord` and `inMaxFileSizeKB`
are both direct Tab 1 inputs with no additional transformation. Tag: `assumed`.

### 4.9 Kafka (`kafkaGB`, the `commons-kafka-controller` NFS partition)

Kafka in this architecture carries exactly one payload: the async on-search
envelope partner-api publishes to the `PARTNER_STAFF_PORTAL` WebSub topic.
Per explicit user instruction, **all** Partner Search traffic is assumed async
(sync search returns its page over plain HTTP and never touches Kafka).

```
dailyKafkaBytes = inPartnerSearchTxnPerDay * AVG_SEARCH_RESPONSE_BYTES
kafkaGB = (dailyKafkaBytes * KAFKA_RETENTION_DAYS) / 1e9
```
- `AVG_SEARCH_RESPONSE_BYTES = 554633` — real measured "Average Content Size" (Locust) for a page-of-100-records sync response, averaged across the same three range query types used in §4.2: `expression_p100` 555,722.68, `graphql_p100` 553,889.91, `predicate_p100` 554,285.24 bytes (source: `partner-api/raw-report.md`, Sync, Pod-1). Used as a stand-in for the actual Kafka message size (never captured directly) on the assumption the async envelope carries a comparable page with comparable overhead.
- `KAFKA_RETENTION_DAYS = 7` — Kafka's own out-of-the-box default (`log.retention.hours=168`), **not** a registry-specific measurement; no retention override was found in this registry's config. Rolling window, not accumulate-forever.
- Tag: `assumed`.

### 4.10 Direct-Attached Storage / NFS totals

Both are pure roll-ups: DAS = `sum(DATABASES[*].size) * 1.3` (30% headroom);
NFS = `sum(PARTITIONS[*].size)`. Both tagged `measured` — their dominant
components trace to real benchmarks or real Tab-1-driven computation (see
the DAS/NFS precedent in §2).

### 4.11 Compute Node hardware recommendation

```
guaranteed vCPU/RAM = sum over guaranteed-QoS pods of (request * podCount)
burstable requests   = sum over burstable-QoS pods of (request * podCount)
burstable limits     = sum over burstable-QoS pods of (limit * podCount)
recommended vCPU = guaranteed_vcpu + K8S_OVERHEAD_VCPU + burstable_requests_vcpu + 0.5*(burstable_limits_vcpu - burstable_requests_vcpu)
recommended RAM  = guaranteed_ram  + K8S_OVERHEAD_RAM_GB + burstable_requests_ram  + 0.5*(burstable_limits_ram  - burstable_requests_ram)
```
Burstable pods are budgeted at their request floor plus half the gap to their
limit ceiling — an assumption that not every burstable pod bursts to its
limit simultaneously. `K8S_OVERHEAD_VCPU = K8S_OVERHEAD_RAM_GB = 8` — flat
assumption for kubelet/containerd/kube-proxy/CNI/CoreDNS/RKE2 control-plane
overhead on this single-node-cluster topology, not measured.

`suggestInstanceCombo()` then converts `(recVcpu, recRam)` into the fewest
m5a instances (4xlarge/2xlarge/large only, each exactly 4 GB RAM/vCPU) that
cover whichever of vCPU-needed or RAM-needed/4 is larger — see the long
comment above that function in `index.html` for the base-4 decomposition
proof. `AWS_DISK_GB_PER_VCPU = 8`, `AWS_DISK_IOPS = 3000` — ratio from the
real m5a.4xlarge benchmark instance (128 GB disk for 16 vCPU).

### 4.12 Storage Node

Fixed at `m5a.2xlarge` (8 vCPU / 32 GB) — not derived from any input. Hosts
PostgreSQL 16 (host-based, not a K8s pod) and the NFS server. Tag: `measured`.

## 5. Pod color coding (Tab 2 diagram)

- Fill color: workload origin (OpenG2P-built vs. 3rd-party), **except** in the
  API / Celery Beat / Celery Worker columns, where every pod is OpenG2P-built
  (origin fill would be uniform/wasted there) — fill instead carries
  QoS × measured/assumed (4 combinations, brand-palette tints only).
- Border color: QoS class (Guaranteed = black/white, Burstable = grey),
  **except** burstable+assumed pods in those same 3 columns, which get a
  deliberate blue border (`--qtag-ba-border`) so they don't visually collide
  with the 3rd-party columns' plain grey-on-grey — the one intentional
  exception to the "brand-palette-only" rule, added at explicit user request.
- `openg2p-staff-ui` is the one fill exception outside those 3 columns too —
  gold "landing" fill via `highlight:"landing"`, to mark the common staff
  portal home page.

Full palette rationale is in the comment block at the very top of
`index.html`.

## 6. Known limitations / deliberate simplifications (read before "fixing" these)

1. **50/50 CR:Intake split.** Tab 1 collects one combined transaction counter
   per source (staff/partner); every formula that needs CR vs. Intake
   separately assumes an even split. No real ratio measurement exists.
2. **Celery queue items use a uniform-weight blended curve.** See §4.3 — the
   benchmark that grounds `CELERY_BLENDED_DRAIN_RATES` mixed 14 scenario
   types at equal backlog size; our real per-queue-type volume mix differs
   from that, and this is a known, flagged approximation.
3. **Celery-worker caps at 3 pods for very high demand**, by design — see
   §4.3's pod-3 regression. Don't extrapolate past 3 without revisiting the
   Blended benchmark section first.
4. **Partner-API volume is excluded from Audit Manager sizing** (§4.6)
   because it genuinely isn't audited on success under the real middleware
   today. If partner-api controllers start setting `request.state.audit_actor`,
   this needs a new term.
5. **Audit Manager sizing assumes no payload capture** (§4.6) — if Audit
   Manager's `details` JSONB is ever extended to carry the actual resource
   payload, `AUDIT_EVENT_BYTES` would need to grow substantially (an earlier,
   now-superseded version of this model assumed full payload capture and got
   ~5x higher numbers — see conversation history / git log if that tradeoff
   needs revisiting).
6. **Kafka assumes 100% async Partner Search** — an explicit user instruction,
   not a measurement of real sync/async traffic split.
7. **`ASSUMED_BYTES_PER_TABLE_ROW` (200) and `K8S_OVERHEAD_VCPU`/`_RAM_GB` (8/8)
   are flat guesses** with no benchmark behind them — the weakest links in
   this tool, unlike almost everything else.
8. **`inIntakeForms` and `inNumPartners` are captured but not wired into any
   calculation** — informational Tab 1 fields only, as of this writing.

## 7. Source documents (paths as of this machine/repo layout)

- `~/github-venky/openg2p-documentation/products/registry/registry/performance-benchmarks/{staff-api,partner-api,celery}/final-report-summary.md` — canonical, published versions.
- `~/github-venky/farmer-registry/performance-testing/documentation/{staff-api,partner-api,celery}/{final-report-summary.md,raw-report.md,final-report.md}` — working/raw versions (WIP, not yet always identical to the published copy above — see `project_farmer_registry_docs_location` memory).
- `~/github-venky/openg2p-documentation/platform/platform-services/audit-manager/technical-architecture.md` — `audit_events` table schema.
- `~/github-venky/registry-platform/apis/openg2p-registry-staff-api/src/openg2p_registry_staff_api/audit_middleware.py` and the `openg2p-registry-partner-api` equivalent — real `AuditMiddleware` source, grounds §4.6 entirely.
- `~/github-venky/farmer-registry/performance-testing/locust/api/{staff-api,partner-api}/**` — real Locust request-building code, used to reconstruct realistic payload/event shapes for §4.6 and §4.9.

## 8. Validation checklist before every publish

Run these (Python, no deps) against `index.html` before publishing a new version:

1. JS brace/bracket/paren balance (skip string/template-literal/comment contents) → must end with an empty stack.
2. CSS brace depth → must end at 0, never go negative.
3. `grep -inE "todo|fixme|not yet implemented|xxx|will add later"` → must be empty.
4. Every `<input id="...">` must appear in both a `num()`/`txt()` call in `readInputs()` and the Tab-1 change-listener array — three-way set difference must all be empty.
5. For any new formula: a standalone Python script mirroring it, run at the real default input values and at a 10x-scaled value, to sanity-check the output is in a plausible range and scales the way it should (usually linearly).

## 9. Full machine-readable reference

The JSON below mirrors everything in §3-§4 in a structured form, for tooling
or quick lookup. Keep it in sync with the prose above and with `index.html`
itself when either changes.

```json
{
  "artifact_url": "https://claude.ai/artifact/2rCDSyrQkzFoMzYm2gsKYT",
  "github_pages_url": "https://venky-ganapathy.github.io/registry-sizing/",
  "repo": "venky-ganapathy/registry-sizing",
  "source_file": "index.html",
  "last_updated": "2026-10-30",
  "tabs": {
    "tab1": "Implementation Partner Inputs",
    "tab2": "Estimation and Sizing"
  },
  "fixed_constants": {
    "HEADROOM": { "value": 0.30, "tag": "assumed", "note": "applied to every capacity check in the tool; no Tab 1 input" },
    "ASSUMED_BYTES_PER_TABLE_ROW": { "value": 200, "unit": "bytes/row", "tag": "assumed", "note": "flat guess, no benchmark" },
    "ASSUMED_DB_GB": { "value": { "masterdata": 8, "awe": 8, "idgenerator": 8, "keycloak": 3, "partnermgr": 8, "connector": 8 }, "unit": "GB", "tag": "assumed" },
    "K8S_OVERHEAD_VCPU": { "value": 8, "tag": "assumed" },
    "K8S_OVERHEAD_RAM_GB": { "value": 8, "tag": "assumed" },
    "AWS_DISK_GB_PER_VCPU": { "value": 8, "tag": "measured", "note": "ratio from real m5a.4xlarge benchmark instance (128GB/16vCPU)" },
    "AWS_DISK_IOPS": { "value": 3000, "tag": "measured" },
    "STAFF_TIERS": { "value": [10150268, 50000264], "unit": "register rows", "tag": "measured" },
    "STAFF_CAPS": { "value": [[184, 319, 453], [164, 329, 369]], "unit": "concurrent users at pod 1/2/3", "tag": "measured", "source": "performance-benchmarks/staff-api/final-report-summary.md" },
    "STORAGE_POINTS": { "value": [[10150268, 139.3], [50000264, 715.1]], "unit": "register rows -> total Postgres GB", "tag": "measured" },
    "PARTNER_SEARCH_RATES": { "value": [3.33, 5.36, 5.10], "unit": "sync RPS at pod 1/2/3", "tag": "assumed-blend-of-measured", "note": "avg of expression/graphql/predicate p100, ID-lookup excluded as outlier", "source": "performance-benchmarks/partner-api/final-report-summary.md" },
    "PARTNER_INGEST_RATES": { "value": [85.74, 141.13, 187.91], "unit": "RPS at pod 1/2/3", "tag": "measured" },
    "CELERY_BLENDED_DRAIN_RATES": { "value": [1249, 2179, 2179], "unit": "rows/min at pod 1/2/3", "tag": "measured", "note": "pod-3 floored at pod-2's rate; raw measured pod-3 was 1883 (a real regression) -- see methodology section 4.3", "source": "performance-benchmarks/celery/final-report-summary.md section Blended" },
    "DRAIN_GOAL_HOURS": { "value": 12, "tag": "requirement", "note": "explicit user-set goal, not a measurement" },
    "AVG_SEARCH_RESPONSE_BYTES": { "value": 554633, "unit": "bytes per page-of-100 search response", "tag": "assumed-blend-of-measured", "note": "avg of expression_p100/graphql_p100/predicate_p100 Average Content Size", "source": "performance-benchmarks/partner-api/raw-report.md, Sync Pod-1" },
    "KAFKA_RETENTION_DAYS": { "value": 7, "tag": "assumed", "note": "Kafka's own default log.retention.hours=168, not registry-specific" },
    "AUDIT_EVENT_BYTES": { "value": 720, "unit": "bytes per audited CloudEvent", "tag": "assumed", "note": "reconstructed from real AuditMiddleware source; no payload ever captured", "source": "registry-platform audit_middleware.py (staff-api and partner-api)" },
    "AUDIT_EVENTS_PER_INTAKE_SUBMISSION_CREATE": { "value": 10, "tag": "measured", "note": "one save_intake_form_submission HTTP call per SECTION_DEFS section" },
    "MEASURED_UNIT_SIZES": {
      "dbGrowthBytesPerChangeRequest": 4980,
      "dbGrowthBytesPerIntakeSubmission": 43078,
      "logBytesPerStaffSearch": 855,
      "logBytesPerPartnerSearch": 2175,
      "logBytesPerStaffChangeRequestCreation": 1347,
      "logBytesPerStaffIntakeSubmission": 866,
      "logBytesPerCeleryIntakeSubmission": 9401,
      "unit": "bytes per unit (one request/task/CR/intake submission)",
      "tag": "measured",
      "note": "from 10,000 seeded CRs and 10,000 seeded intake submissions against the farmer-registry perf-test database; CR-creation/intake-submission log sizes only measured on staff-api, assumed equal for partner-api"
    }
  },
  "tab1_inputs": [
    { "id": "inPrimaryRecords", "label": "Primary Register records", "default": 1000000, "feeds": ["totalRegisterRecords", "totalTableRecords", "totalDocuments"] },
    { "id": "inSecondaryRegisters", "label": "Number of Secondary Registers", "default": 1, "feeds": ["totalSecondaryRecords"] },
    { "id": "inRatioSecondaryRecords", "label": "Secondary records per Primary record", "default": 0.41, "feeds": ["totalSecondaryRecords"] },
    { "id": "inSupportingTables", "label": "Number of Supporting Tables", "default": 14, "feeds": ["totalTableRecords"] },
    { "id": "inRatioTableRecords", "label": "Supporting Table records per Primary record", "default": 1.33, "feeds": ["totalTableRecords"] },
    { "id": "inIntakeForms", "label": "Number of Intake Forms", "default": 3, "feeds": ["informational only"] },
    { "id": "inMaxFileSizeKB", "label": "Max file size per document (KB)", "default": 384, "feeds": ["commonsMinioGB"] },
    { "id": "inFilesPerRecord", "label": "Files per Primary Register record", "default": 4, "feeds": ["totalDocuments"] },
    { "id": "inConcurrentUsers", "label": "Concurrent Staff UI users", "default": 20, "feeds": ["staffApiPods"] },
    { "id": "inStaffTxnPerDay", "label": "Staff UI transactions/day (CR+Intake)", "default": 500, "feeds": ["totalDailyTxns", "auditMgrGB"] },
    { "id": "inStaffSearchTxnPerDay", "label": "Staff UI search transactions/day", "default": 500, "feeds": ["lokiGB", "auditMgrGB"] },
    { "id": "inNumPartners", "label": "Number of partners", "default": 2, "feeds": ["informational only"] },
    { "id": "inPartnerTxnPerDay", "label": "Partner transactions/day (CR+Intake)", "default": 1000, "feeds": ["totalDailyTxns", "partnerApiPods (ingest)"] },
    { "id": "inPartnerSearchTxnPerDay", "label": "Partner search transactions/day", "default": 5000, "feeds": ["lokiGB", "kafkaGB", "partnerApiPods (search)"] },
    { "id": "inLogRetention", "label": "Log retention (days)", "default": 15, "feeds": ["lokiGB"] },
    { "id": "inSizingYears", "label": "Years to size storage for", "default": 5, "feeds": ["txnDbGrowthGB", "auditMgrGB"] }
  ],
  "derived_outputs": {
    "staffApiPods": { "formula": "podsNeeded(interp(STAFF_CAPS,STAFF_TIERS,totalRegisterRecords), inConcurrentUsers/(1-HEADROOM))", "tag": "measured" },
    "partnerApiPods": { "formula": "max(podsNeeded(PARTNER_SEARCH_RATES, searchNeedRate), podsNeeded(PARTNER_INGEST_RATES, ingestNeedRate))", "tag": "measured" },
    "celeryWorkerPods": { "formula": "podsNeeded(CELERY_BLENDED_DRAIN_RATES, (sum of 7 queue terms)/(DRAIN_GOAL_HOURS*60)/(1-HEADROOM))", "tag": "measured" },
    "mainDbGB": { "formula": "lerp(STORAGE_POINTS, totalRegisterRecords)", "tag": "measured" },
    "tableStorageGB": { "formula": "totalTableRecords * ASSUMED_BYTES_PER_TABLE_ROW / 1e9", "tag": "assumed" },
    "txnDbGrowthGB": { "formula": "(totalCrPerDay*4980 + totalIntakePerDay*43078) * 365 * inSizingYears / 1e9", "tag": "measured" },
    "auditMgrGB": { "formula": "(staffCrPerDay*2 + staffIntakePerDay*11 + inStaffSearchTxnPerDay) * 720 * 365 * inSizingYears / 1e9", "tag": "assumed" },
    "lokiGB": { "formula": "dailyLogBytes/1e9 * inLogRetention", "tag": "assumed" },
    "commonsMinioGB": { "formula": "inPrimaryRecords * inFilesPerRecord * (inMaxFileSizeKB/1024) / 1024", "tag": "assumed" },
    "kafkaGB": { "formula": "inPartnerSearchTxnPerDay * 554633 * 7 / 1e9", "tag": "assumed" }
  },
  "known_limitations": [
    "50/50 CR:Intake split assumed -- no real ratio measurement exists",
    "Celery blended curve mixed 14 scenarios at equal backlog weight; real workload mix differs",
    "Celery-worker pods cap at 3 by design -- pod-3 regression in the real benchmark",
    "Audit Manager sizing excludes all Partner-API volume -- not audited on success today",
    "Audit Manager sizing assumes zero payload capture, per the real middleware source",
    "Kafka sizing assumes 100% async Partner Search -- explicit user instruction, not measured",
    "ASSUMED_BYTES_PER_TABLE_ROW and K8S_OVERHEAD_* are flat unmeasured guesses",
    "inIntakeForms and inNumPartners are captured but not wired into any calculation"
  ]
}
```

## 10. Revision log

Add an entry here every time the sizing model changes, with the date and a
one-line summary of what changed and why. (Empty as of this file's creation —
everything above reflects the state as of 2026-10-30.)

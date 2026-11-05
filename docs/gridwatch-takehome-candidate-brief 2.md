# Take-Home Assignment: GridWatch — EV Network Monitoring Platform

**Role:** Tech Lead + Solution Architect

**Time budget:** This is a 3-part assignment. We expect a strong submission takes **roughly 10–14 hours of focused work**, and we will agree a delivery window with you (typically 48 hours from receipt — tell us when you can dedicate the time and we will set the deadline together).

**A note on scope:** The assignment is deliberately larger than what fits comfortably in the time budget. We do not expect you to finish everything. We would rather see how you prioritise, where you go deep, and where you consciously stop, than see a rushed attempt at 100%. Read "What 'Done' Means" before you plan your time.

**Tools:** Use anything — Claude, ChatGPT, Cursor, Copilot, Google, Stack Overflow, the official docs, Reddit. We assume you will. The discussion round is where we make sure you understand what you submitted.

**Stack:** Your choice for the build slice. Node.js/TypeScript preferred, but pick what lets you move fastest. RDBMS of your choice. Docker Compose required for anything you run.

---

## Business Context

We operate **GridWatch**, a centralised monitoring platform for electric-vehicle charging networks. Our customers are **Charge Point Operators (CPOs)** — companies that run fleets of public and semi-public charging stations.

We do **not** talk to chargers directly, and you will not either. Each CPO already runs a **Charging Station Management System (CSMS)** — the vendor platform that the chargers themselves connect to over OCPP. The chargers send their data up to the CSMS; **our job begins at the CSMS boundary.** GridWatch ingests whatever the vendor's CSMS exposes to us (an API and a webhook — documented below), normalises it, stores it, and turns it into a single operational picture across every CPO, site, and charger we monitor.

The product GridWatch delivers is a **centralised monitoring dashboard with a live GIS map** at its centre: an operator opens GridWatch and sees, in near-real-time, the health and activity of their entire network on a map and in a set of dashboards — which chargers are live, which are faulted, what is charging right now, where utilisation is high, where revenue is being earned, and where something is wrong.

You are being considered to **architect and technically lead** this platform. This assignment asks you to design it, critique an existing design, and build the hardest slice of it.

---

## The System We Want

A multi-tenant platform with this hierarchy. Every data point, alert, and report is anchored to a node in it:

```
Operator (tenant)  →  Site (a physical location)  →  Charger (an EVSE)  →  Connector (a plug)
```

**Scale we are building for:** Production is **~50,000 chargers across ~12,000 sites**, spread across **India and Germany**, each charger emitting telemetry roughly **every 10 seconds**. The sample data we give you (see "Provided Materials") is a **50-site / ~220-charger pilot** — it exists to let you validate correctness, not to represent scale. Design for production; validate on the pilot.

### Core capabilities

1. **Ingestion from the vendor CSMS.** Pull and receive telemetry through the vendor interface (contract below). Normalise it into a clean internal schema. Telemetry includes connector/charger status, active charging sessions, energy and power readings, and fault events.

2. **The centralised monitoring dashboard.** Real-time view of the whole network: charger/connector status, active sessions, faults, utilisation. Operators live in this screen.

3. **The live GIS map (critical).** Every site and charger on a map, coloured by live status. Operators need: the network plotted geographically; "find the nearest available charger to a point"; and **coverage / density analytics** (e.g. chargers per km² by region, gaps in coverage). The map is the centre of the product, not a decoration.

4. **Sessions, energy & settlement.** A charging session has a start, an end, and energy delivered. Convert energy delivered into an invoiced amount using a per-CPO tariff (per-kWh price + a session fee), in the destination's local currency (**INR** for India, **EUR** for Germany). A **daily settlement run** produces the invoiced totals the CPO reconciles against.

5. **Analytics & alerts.** Utilisation per site/region, fault trends, offline-charger detection, and threshold alerts. (Demand forecasting is interesting but is explicitly **architectural-understanding-only** — describe how you'd approach it; do not build models.)

### Success metrics (these are contractual freshness targets)

The platform is sold on freshness. These are the KPIs in the customer contract:

| Signal on the dashboard | Target latency | Percentile |
|---|---|---|
| Connector / charger status | < 5 s | P99.9 |
| Active session power & energy | < 10 s | P99.9 |
| Fault / alarm surfaced | < 5 s | P99.9 |
| Dashboard tile end-to-end freshness | < 2 s | P99.9 |
| Map live-status update | < 10 s | P95 |

---

## The Vendor CSMS Interface

This is the only way data reaches GridWatch. Treat it as a production dependency: it rate-limits, it has downtime, and its data is shaped the way it is shaped, not the way you wish.

**a) Poll API (REST)**
- `GET /v1/chargers?site_id={id}&since={iso8601}` — returns current charger/connector status and any sessions/meter readings since the cursor.
- **Documented minimum poll interval: 30 seconds.** The vendor rate-limits more frequent polling per site.
- Paginated. Occasionally returns `500`s under load. Returns `429` if you poll too fast.

**b) Webhook (push)**
- The CSMS POSTs events (status changes, session start/stop, faults) to a URL you register.
- Delivery is **best-effort and at-least-once. Events may be delivered out of order, and may be batched up to 60 seconds.** No ordering or exactly-once guarantee.

**c) Data notes (read carefully — this is how the data actually arrives)**
- **Energy** in meter readings is reported as a **cumulative lifetime register in watt-hours (Wh)** per connector — not per-session, not per-interval, not kWh.
- A **site record carries an address string and a `site_id`** (e.g. `"Sector 18 Metro Station, Noida, UP 201301"` or `"Ladestation Hauptbahnhof, 70173 Stuttgart"`). It does **not** carry latitude/longitude.
- Timestamps arrive in the **charger's local timezone** with offset, and are not normalised across the network.

We provide **CSMS stubs** (`gridwatch-csms-stubs.md`) — the full endpoint contract plus canned request/response payloads for the normal cases *and* the failure cases (429, 500, malformed page, duplicated and out-of-order events). Integrate against these. If it helps you, wrap them in a trivial local fake — but you do not need a running vendor service, and we are not shipping one.

---

## The Assignment

### Part A — Architecture & Design Document *(this is the centre of the assignment)*

Produce an architecture for GridWatch at the production scale described above. We are not looking for a pretty diagram; we are looking for **decisions and the reasoning behind them.** Cover at least:

- The end-to-end data flow from the vendor CSMS to the dashboard and map.
- Storage design — what stores what, and why. Schema sketch for the core entities and the telemetry.
- The GIS architecture — how the map is served and queried at scale, and how spatial queries (nearest, within-region, density) are computed and kept fast.
- How you meet — or **where you cannot meet, and what you would do about it** — the freshness KPIs in the table above.
- High availability, failover, and what happens to data when a component dies (state your RPO/RTO and justify them against your replication choices).
- Rough capacity sizing for the production scale (compute, storage growth, throughput). Show your arithmetic.
- Security and multi-tenant isolation.
- Observability — specifically, how you would *prove* you are meeting the freshness KPIs.

Include a **decision log**: the three or four hardest calls you made, the options you weighed, and why you chose what you chose. If anything in this brief is unclear, underspecified, or in tension with something else, that is exactly the kind of thing we want to see you handle — email us a sharp question, or make a defensible assumption and write it down. **Tell us what you would not build, and why.**

### Part B — Architecture Review

We have attached a draft architecture document from a previous engagement (`gridwatch-reference-architecture-DRAFT.md`). A previous contributor produced it. **We have not vetted it.**

Review it as you would review a senior engineer's design before it goes to a client. Produce a written review that:

- Identifies the problems you find.
- **Ranks them by severity / blast radius** — what hurts the business most, soonest. Be explicit about your ranking criteria.
- Proposes a concrete fix for each.
- Calls out anything you would need to verify or ask about before signing off.

Length is not the point. A precise, well-prioritised review beats an exhaustive one.

### Part C — Build the Hardest Slice

Implement one **thin vertical slice**, end-to-end and runnable, of the path you consider hardest or most critical. At minimum, we want to see real ingestion working:

> **Ingest from the CSMS stubs → normalise → store → expose one correct, useful aggregate** (e.g. live status rollup per site, or current network utilisation, or a "nearest available charger" query that returns correct results).

Requirements for whatever you build:
- A **`docker compose up`** that brings up your service and its datastore(s) in one command. (No vendor service to run — you integrate against the stubs.)
- **Infrastructure-as-code** (Terraform or equivalent) for any cloud-style resources you introduce, executed locally (LocalStack is fine).
- Real error handling around the vendor interface — the stubs include 429s, 500s, and malformed pages; show what your system does about them.
- A **small** set of tests aimed at the failure modes you designed for, not at coverage.
- The map / GIS coordinate handling has to deal with the data as it actually arrives.

Build the slice that proves the riskiest part of your architecture. Going wide on shallow CRUD is the wrong move; going deep on the part that would sink the project is the right one.

---

## Provided Materials

- `gridwatch-reference-architecture-DRAFT.md` — the document to review in Part B.
- `pilot-sample-data.json` — the 50-site / ~220-charger pilot dataset, with telemetry samples and the field notes that explain how the data really arrives.
- `gridwatch-csms-stubs.md` — the vendor CSMS interface contract plus canned payloads (success, error, and quirk cases) to integrate against for Part C. No running vendor service is provided or required.

---

## What "Done" Means

The assignment is intentionally oversized. We score on how far you get **past a baseline**, and how well, not on a percentage.

**Baseline we need to see (the gate):**
- Part A: a coherent architecture that addresses the core data flow, the map, and the freshness targets — and engages honestly with the parts that are hard or don't add up.
- Part B: the high-severity problems found and correctly prioritised.
- Part C: a slice that **actually runs** with one command and ingests the CSMS stub data into a store.

**Beyond the gate (this is where candidates differentiate):** depth on HA/DR and sizing; the GIS design at scale; idempotency and correctness under the vendor's delivery guarantees; richer failure handling; the settlement path; thoughtful observability; a decision log that anticipates our questions.

If you run low on time, **stop and write down what you would have done next and why.** A clear "here's where I stopped and what I'd tackle next" is worth more than half-finished code with no explanation.

---

## Submission

- A link to a **public GitHub repo** (share with `[hiring@cidroy.com, shubham@cidroy.com]`) containing the Part A document, the Part B review, and the Part C code + Docker Compose + IaC + tests.
- A **`README.md`** with: how to run the slice; your design summary and decision log; your assumptions; and a "where I stopped / what's next" section.
- If you have questions about the brief, reply to your invitation email with your full list. We answer in writing; if a 30-minute call is needed, we'll schedule one.

---

## What Happens Next

After we review your submission, we hold a **75–90 minute discussion round.** We will walk through your design and code, ask you to defend specific decisions, hand you a fresh piece of architecture to critique on the spot, and run a short live code-review and extension exercise. The point of that round is to confirm you understand everything you submitted, whoever or whatever helped you produce it. Candidates who can defend their own work do well in it.

---

*Most people we hire for this role have not used some part of this stack before — a CSMS interface, a particular GIS approach, one of the tools you'll reach for. That is intentional. We are not testing whether you have seen these exact things; we are testing how fast you read the docs, judge the options, and ship something that works, and how clearly you reason about the parts you can't finish. So don't give up early. The further you get, the more we have to talk about.*

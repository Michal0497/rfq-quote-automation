# Decision log

The decisions that shaped the product, in the order they were made. Each one names the trade-off that was accepted.

## Product and scope

**D1. Verified draft + explicit exceptions + human approval — not an autonomous quoting bot.**
B2B quotes carry commercial risk (wrong part number, wrong price, wrong customer terms). The product prepares a draft, lists everything it could not decide safely, and a person approves before anything reaches the customer.
*Trade-off:* the rep still clicks for every quote; full automation was deliberately not a goal.

**D2. One vertical first (electrical / technical distribution).**
Part numbers, manufacturers, units and MOQ rules are concrete enough to build deterministic matching and pricing. Other verticals only after this one is validated.
*Trade-off:* some rules are vertical-specific (electrical ratings in the suggestion validator).

**D3. Installed per client (own n8n + own PostgreSQL), no multi-tenant SaaS.**
Distributors keep customer prices and RFQs on their own infrastructure; one installation = one company.
*Trade-off:* installation and updates must be scripted (see D20) and every client runs their own stack.

## Architecture

**D4. Five workflows: one core identical for every client, separate workflows only where clients differ.**
Started as 12 workflows, consolidated to core + channels (HTTP API, email); later Ops Alerts (must work when the core does not) and the ERP connector (the only client-specific piece). Inside a workflow, lanes are chosen by a command API `{ operation, payload }`, because n8n allows one Execute Workflow trigger per workflow.
*Trade-off:* the core is a large workflow (≈120 nodes) with six operations instead of many small ones.

**D5. Input adapters produce a canonical RFQ; the core never knows the source.**
Email, HTTP API (and future channels) map their input to one contract; validation, idempotency and processing live in one place.

**D6. Business state lives in PostgreSQL, not in n8n execution history.**
Every step writes its result (checkpoint); statuses are explicit (`RECEIVED → PROCESSING → REVIEW_REQUIRED / PRICED → QUOTE_DRAFTED → COMPLETED`). Recovery, audit and reporting read the database.

**D7. Arithmetic and pricing rules in SQL `numeric`, never in the LLM and never in floats.**
Line totals, discounts, VAT and totals are computed in single atomic queries; the quote freezes its prices.

**D8. Business exceptions are separate from technical errors.**
`PRODUCT_NOT_FOUND`, `AMBIGUOUS_MATCH`, `PRICE_NOT_FOUND`, `UNIT_MISMATCH`, `BELOW_MOQ`, `SEMANTIC_MATCH_SUGGESTED` become tasks for a sales rep; database, LLM, ERP or SMTP failures become alerts for an administrator.

## AI

**D9. LLM → structured output → deterministic validation → rules → controlled action.**
Email extraction uses a strict JSON schema; every extracted description, part number and quantity must appear in the source text (grounding), units are mapped by a deterministic alias table, the customer email is always taken from the header. A message that cannot be read safely goes to a person *as a whole* — no partial RFQs.
*Trade-off:* one incomplete line rejects the whole email to manual handling.

**D10. LLM product suggestions are never accepted automatically.**
For lines without a usable part number: SQL preselects catalogue candidates → one LLM call per RFQ with an enum of candidate SKUs (the model cannot invent a product) → deterministic check of manufacturer and ratings → the suggestion becomes a review task.
*Trade-off:* a correct suggestion still costs one click.

**D11. The model is pinned to a dated version and called once per message / line.**
Model changes are a configuration change plus regression run; extraction and suggestion results are stored, so recovery never calls the LLM twice for the same input.

## Reliability

**D12. Idempotency at every entry point.**
`submission_id` (API), Message-ID (email, plus an `EXTRACTING` lease after a real double delivery caused two LLM calls), one quote per RFQ, a `SENDING` claim before the customer email.

**D13. Leases + a recovery schedule instead of hoping executions finish.**
Processing claims the RFQ with a lease; every minute one query lists stuck work (expired leases, lost notifications, approved but undelivered quotes) and re-sends the commands. Consecutive failures end in `FAILED` with an alert.

**D14. Never resend to the customer automatically.**
If SMTP fails or the result is uncertain, the quote becomes `SEND_UNCERTAIN` and a person checks the Sent folder (mark sent / resend). A duplicate quote to a customer is worse than a delayed one.
*Trade-off:* reviewer notifications use the opposite rule — a duplicate is better than a lost task.

**D15. Resume after human review from the matching checkpoint.**
After the last decision the RFQ is recalculated on stored lines with the decisions as overrides — one matching/pricing logic, no special path for reviewed lines.
*Trade-off:* untouched lines are re-priced with the current price list.

**D16. Error Workflow + handled `ops_event`s + throttled alerts + public `/health`.**
Failed executions and handled technical problems are recorded and emailed at most once per fingerprint per 30 min; `/health` lets an external monitor see a dead database or stopped schedules (when alert emails cannot be sent).

## Data and integrations

**D17. Client data through a universal import, as full snapshots.**
Catalogue, customers and customer prices come from ERP/Excel as CSV/XLSX or JSON. A snapshot deactivates what is missing (never deletes), is applied all-or-nothing, and a file that would deactivate more than 20 % of records is rejected (truncated export protection). Identical to the last applied import = duplicate; an older snapshot sent again is applied (reverted price changes).

**D18. Customer recognition is deterministic and fails safe.**
Exact address beats domain; two customers on the same level = not recognised (list prices). Price priority: review price > live ERP price > customer price > list price − discount > list price.

**D19. The core may ask the ERP for live prices — only through a replaceable connector.**
Changed from "the core never calls client systems". The connector is the only client-specific workflow; its answer is validated like external data and any failure falls back to imported prices with a note for the approver and an admin alert. Quoting never stops because of the ERP.
*Trade-off:* every recalculation waits for the ERP up to 10 s. Automatic scheduled data sync is deferred to the first client deployment (it depends entirely on their ERP).

## Security and packaging

**D20. Installer keeps workflow and credential IDs.**
Channels call the core by ID and every workflow reports to Ops Alerts by ID, so `install.sh` imports workflows and credentials through the n8n CLI with their IDs, fills credentials from `.env`, applies migrations once each (checksummed) and publishes in dependency order. The same script updates an installation.

**D21. Two authentication layers for decisions.**
An installation API key (stored as SHA-256, 401 in the product's error format) for the HTTP API, plus the per-task / per-quote token for every decision — checked in the core, so API and email forms get the same protection. Found in a gap analysis: before, anyone with a quote ID could approve it through the API.

**D22. Fresh installation is a regression test.**
A throwaway PostgreSQL container gets every migration from scratch and is compared with the development database (schema, settings, text keys), and every SQL query of the workflow exports must compile against it. Added after the same gap analysis found that migrations no longer installed from zero.

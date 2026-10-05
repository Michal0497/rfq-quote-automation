# Testing

Tests run against a real installation (n8n + PostgreSQL + a local mail catcher), end to end through the public webhooks, and check the results in HTTP responses, in the database and in the mailbox. Last full run: **27/27**.

| Mode | Scope |
|---|---|
| default | 20 tests without LLM calls, ~35 s — run after every workflow change |
| `--llm` | + 5 tests with real OpenAI calls |
| `--slow` | + recovery scenarios (waits for the recovery schedule) |
| `--fresh-install` | empty database → all migrations → compared with the development database |
| `--all` | everything (27) — before every release |
| `--smoke` / post-install check | safe on a client installation: health, 401 without key, 422 for an invalid request — creates no data |

The full suite refuses to run outside `localhost` or without the mail catcher, so test emails can never reach real people. Every test uses its own IDs; failures print the exact difference (got / expected). Verified to catch regressions by mutating configuration (e.g. VAT rate) and by running the fresh-install test on the pre-fix migrations.

## Suites

| Suite | Test | What it proves |
|---|---|---|
| smoke | health endpoint reports ok | database reachable, recovery schedules running |
| smoke | extraction validator (9 cases, no LLM) | grounding rejects invented part numbers, quantities, descriptions; unit aliases; default unit; not-an-RFQ consistency |
| smoke | suggestion validator (7 cases, no LLM) | wrong manufacturer, different rating (25 A vs 40 A, 20 W vs 40 W), SKU outside candidates, `null` answer |
| core | API key | missing / wrong key and wrong Bearer → 401 in the product format on all three endpoints, nothing stored; valid Bearer → 202 |
| core | intake: valid RFQ | 202, priced, quote drafted with exact net / VAT / gross |
| core | intake: same submission again | 200 `DUPLICATE` with the original RFQ id, nothing stored twice |
| core | intake: invalid RFQ | 422 with all errors (date, duplicate line number, quantity), nothing stored |
| core | matching + pricing: mixed RFQ | exact and normalised match, `BELOW_MOQ`, `PRICE_NOT_FOUND`, `UNIT_MISMATCH`, `PRODUCT_NOT_FOUND` — one task each |
| core | review via API | 404 / 409 / 422 (incl. wrong and missing task token), decisions resume processing, a manually selected product without price creates a new task, final totals |
| core | quote: approve | wrong / missing quote token rejected, `MARK_SENT` while pending → 409, exactly one customer email, `SENT`, RFQ `COMPLETED` |
| core | quote: reject | reason required, rejected quote is never sent |
| core | quote: two simultaneous approvals | one 200, one 409, one customer email |
| core | review form | GET never changes data (safe for link scanners), token required, decimal comma, customer `<script>` escaped in page and email, units shown as labels |
| core | quote approval page | token required, shows the exact customer email |
| core | email: scanned PDF | manual handling without an LLM call |
| import | catalogue CSV (Windows-1250) | applied → same file = `DUPLICATE` → older snapshot restores (product deactivated, not deleted) |
| import | truncated and invalid files | 422 with spreadsheet row numbers, nothing changed, admin alert |
| import | customer pricing | domain → customer price + discount, exact address, unregistered address of a known domain → list prices; approval email explains prices; customer email shows net prices only |
| import | ERP live price | fake ERP: ERP price wins; ERP down / invalid prices → imported prices + approver note + alert |
| import | customers XLSX, customer prices CSV/JSON | upload form behind basic auth, local times in the import list |
| llm | Polish email body | extracted, matched, quote drafted |
| llm | newsletter | `NOT_RFQ`, no RFQ |
| llm | missing quantity | whole message to manual handling |
| llm | prompt injection | instructions in the email ignored, only the real item extracted |
| llm | matching level 3 | suggestions for two lines, honest "no match" (does not pick 32 A for a 40 A request), no LLM call without candidates |
| slow | recovery: five failure scenarios | stuck `RECEIVED`, expired lease, lease expired on the last attempt (`FAILED` + alert), lost reviewer email, lost resume after the last decision |
| fresh install | empty database → migrations → dev | schema, constraints, indexes, settings and text keys identical; workflow IDs exist and are active; all 50 SQL queries of the workflows compile |

## Failure paths (roadmap section 13)

| Scenario | Covered by |
|---|---|
| Missing / invalid fields, duplicate line | intake: invalid RFQ |
| Unknown SKU, ambiguous match, unit mismatch, missing price, below MOQ | matching + pricing (ambiguous match: SQL check only — the demo catalogue has no collision) |
| Malformed document | scanned PDF; unsupported attachments → manual handling (manual test) |
| Malformed / invented LLM output | extraction and suggestion validators; parsing of broken JSON / refusals — manual test only |
| API failure, timeout | ERP live price (failure, invalid answer; timeout manual); OpenAI failure — manual test |
| Duplicate webhook / event | same submission again; double email delivery (3 concurrent deliveries → 1 LLM call) — manual test |
| Retry / recovery | recovery: five scenarios |
| Human review path | review via API, review form |
| Failure before / after side effects | simultaneous approvals; SMTP down during sending and crash after sending → `SEND_UNCERTAIN` — manual tests |
| Regression dataset for extraction / matching | validator case files, sample emails (Polish body, PDF, CSV, newsletter, missing quantity, prompt injection, scanned PDF), semantic matching set |

## Known gaps

- Not automated: `AMBIGUOUS_MATCH` with real data, LLM response parsing errors, OpenAI outage, `SEND_UNCERTAIN` paths, double email delivery, approval threshold (`ELEVATED`), ERP timeout, update rollback from backup.
- LLM tests depend on a pinned model version and cost a few cents per run; they are not part of the default suite.
- The fresh-install test validates the database and compiles every query, but runs the end-to-end flows only against the development installation (the installer itself was verified end to end on a separate stack).

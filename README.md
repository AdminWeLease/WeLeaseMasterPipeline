# WeLease Operations Dashboard

Internal operations dashboard for **WeLease Property Management**, San Diego — roughly 404 doors under management.

A Google Apps Script project bound to a Google Sheet. It pulls from AppFolio, LeadSimple and ShowMojo overnight, writes everything to tabs, and serves a single-page HTML dashboard to the team.

> **Private repository.** Nothing here is secret in itself — no credentials are in the code — but it contains WeLease business rules, GL account ids, property-group conventions and resident-data field names. Keep it private.

---

## What it does

Twelve views, each answering one operational question:

| View | Answers |
|---|---|
| **Needs Attention** | What arrived overnight, and what is overdue right now |
| **Leasing** | Vacancies, applications, days on market, showings |
| **Lease Renewals** | The 80-day inspection clock, who is overdue, who has no renewal date |
| **Maintenance** | Open work orders, no vendor assigned, aged, awaiting owner approval |
| **Utility Billing** | The billing schedule and what is unbilled |
| **Delinquency** | Who owes what, aged, and when rent actually gets paid |
| **Collections** | Vendor bills, stalled approvals, owner shortfalls, former residents |
| **Insurance** | Resident and owner coverage, three-way split, chase list |
| **Process Adherence** | LeadSimple process health, overdue and stalled |
| **Owners Pipeline** | Deals, win rate, stale deals, nobody-assigned |
| **Weekly Report** | 37 metrics across 8 departments, three weeks side by side |
| **Performance** | KPIs against industry benchmarks, onboarding, door history |

Plus: nightly Drive snapshots for monthly owner reporting, a notice merge feed for the PDF filler, and a status/notes log so the team can record what they are waiting on.

---

## Architecture

```
AppFolio Reports API v2 ─┐
LeadSimple REST ─────────┼──> nightly jobs ──> Google Sheet tabs ──> renderPage_() ──> web app
ShowMojo ────────────────┘                          │
                                                    ├──> Drive snapshots  (owner reporting)
                                                    └──> Residents feed   (PDF notice filler)
```

**Everything is server-rendered.** All twelve views are built into one HTML document and toggled with `display:none`. That makes switching views instant and first paint slow — a deliberate trade, and the reason the page-weight test exists.

### Files

| File | Holds |
|---|---|
| `Core_v2.gs` | HTTP, retries, pagination, tab read/write, dates, money, the admin gate |
| `Jobs.gs` | The nightly job chain and the AppFolio column allow-lists |
| `Shared.gs` | Address book, scope filters, tenancy liveness |
| `Master_Scope.gs` | The two human-owned sheets that decide what counts |
| `Metrics.gs` | KPI calculations against the benchmark registry |
| `Portfolio.gs` | Portfolio summary, door history, rent income, rent timing |
| `Collections.gs` | Vendor bills, approvals, owner shortfalls |
| `Inspections.gs` | The renewal pipeline and the 80-day clock |
| `Data_Quality.gs` | The audit, and insurance compliance |
| `LeadSimple.gs` | Process and deal sync, onboarding journeys |
| `ShowMojo.gs` | Showings and listing performance |
| `Utility_Schedule.gs` | Utility billing schedule |
| `Throughput.gs` | Added versus cleared, per list, per run |
| `Status.gs` | Per-item status, follow-ups and the note thread |
| `Weekly_Report.gs` | The Tuesday management report |
| `Drive_Export.gs` | Daily and monthly snapshots for the owner review |
| `Owner_Update.gs` | Owner packs |
| `Notice_Feed.gs` | The merge feed the PDF notice filler reads |
| `Dashboard.gs` | All rendering, CSS, client JS |
| `Diagnostic.gs` | Everything named `diagnose*` and `reconcile*` |
| `Server.gs` | **Separate project.** The PDF notice filler backend — do not paste into this one |

---

## Setup

1. Create a Google Sheet, then **Extensions → Apps Script**.
2. Paste each `.gs` file in as a file of the same name.
3. **Project Settings → Script Properties:**

```
APPFOLIO_CLIENT_ID          from AppFolio
APPFOLIO_CLIENT_SECRET      from AppFolio
LEADSIMPLE_API_KEY          from LeadSimple
SHOWMOJO_API_TOKEN          from ShowMojo
DASHBOARD_ADMINS            comma-separated, who sees maintenance notes
NOTICE_TOOL_URL             optional, the PDF filler web app URL
```

4. Run `setupSheets()`, then `verifyConnections()`.
5. Run `installTriggers()` — ⚠️ deletes existing triggers first.
6. **Deploy → New deployment → Web app.** Execute as **Me**, access **Anyone in WeLease**.
7. Run `rebuildAll()` once by hand, then `rebuildDashboardCache()`.

### Credentials never go in a file

Every secret is read through `PropertiesService.getScriptProperties()`. There are no literal keys anywhere in this repo and there must never be — a private repo can be made public by accident, and a key in git history outlives the file it was in.

---

## Nightly schedule

```
02:00  rebuildAll              every AppFolio report
02:30  rebuildFinancials       rent roll, delinquency, receivables
03:00  rebuildLeasing          applications, vacancy, tickler
03:30  rebuildCompliance       inspections, deposits, renewals
03:45  rebuildCollections
04:00  rebuildPortfolio        summary, door history, rent income, rent timing
04:10  rebuildMasterScope
04:15  buildUtilitySchedule
04:20  rebuildLeadSimple       incremental, plus onboarding journeys
04:25  rebuildShowMojo
04:30  computeMetrics
04:45  rebuildDataQuality
04:50  rebuildThroughput       needs two runs before movement appears
04:55  buildNoticeMergeFeed
05:00  rebuildDashboardCache   must be last
05:30  exportDailySnapshot
06:30  buildWeeklyReport       Mondays
06:00  exportMonthlySnapshot   15th
```

---

## Tests

21 suites, run in a Node VM sandbox with Apps Script stubbed. No deployment needed.

```bash
npm install jsdom          # two suites need a real DOM
for f in *_test.js jsdom_ui.js; do node "$f" || echo "FAILED: $f"; done
```

`harness.js` loads every `.gs` into one VM context — which also catches **duplicate function definitions**, the failure mode Apps Script hides by silently letting the last one win.

`size_test.js` projects page weight at real portfolio volume. Currently ~400 KB. **Keep it under 600 KB** — beyond that, first paint on the Apps Script iframe gets noticeably worse.

---

# Things that will bite you

This is the part worth reading. Every item below cost real time to find.

## AppFolio

**The report routes are POST-only.** A `GET` on `bill_detail.json` returns 404 no matter how well-formed it is. Pagination never worked because of this, and it stayed hidden until one report first exceeded 5,000 rows.

**5,000 rows is the page cap.** A tab sitting at exactly 5,000 rows is truncated, not complete. `diagnoseTruncation()` flags tabs on a page boundary.

**`next_page_url` can be query-only** — no path. Pasting it onto the account origin gives you the account root and a 404.

**`receivables_activity` is scoped to CURRENT occupancies.** Nine probe parameters returned byte-identical results. Movers-out take their payment history with them, so it cannot be used for anything historical.

**`account_number` is null on the general ledger.** The number lives inside `account_name`. Resolve on the name.

**Window boundaries must snap to a month.** 93% of this portfolio's rent posts on the 1st, so a window starting on the 2nd loses almost the whole month. `buildRentIncomeGL` counts back in months, never in days, for exactly this reason.

**Hidden records need `property_visibility: 'all'`.** AppFolio hides offboarded properties, and door history needs them to count losses.

**`work_order` defaults to the open queue only.** The rent roll is as-of-today only — there is no historical rent roll, which is why rent income comes from the ledger.

## Rent has two measures, and they are not the same number

**Rent income** (GL 4100 + 4105, accrual) is recognised in the month the rent covers. This is what the income statement reports.

**Resident receipts** (cash on arrival) include RBP, pet rent, fees and deposits, and land in the month the money arrived.

They cross over, because prepayment moves cash a month earlier than the income it funds. Expect them to trade places rather than track.

## Prepayment

**Most WeLease residents pay before the month they are paying for.** In September 2026: $894,431 applied out of account 2300 Prepayment on the 1st, across 172 payers. Only 135 payers sent cash during 1–2 September.

The receipts feed dates money when it *arrives*. Naively, every prepayer looks like a non-payer — and any of them who also paid a $50 benefits charge got dragged into the month, measured against full rent, found short, and counted as **late**. The dashboard reported 72% paying after the 5th when roughly 72% owed nothing at all.

`buildRentTiming` now models a **rolling balance per household**, the way the ledger does. Carried credit satisfies the next month on day 0. One dollar is spent once. There is a `before_1st` bucket because paying early is the most common case here.

**A month that has not reached the 5th cannot report how it ended.** The headline uses the last closed month.

## Google Sheets

**Sheets coerces anything date-shaped.** `2026-08` becomes a Date; a small integer in a date-formatted cell becomes 4 January 1900. `monthNorm_` repairs on read, and `writeTab_` pins text columns to `@` **before** the values land — afterwards is too late, the original is already gone.

**`appendTab_` keeps to the sheet's own header row.** It used to write positionally against the header the *caller* computed, so a builder gaining a field shifted every value after it one column left. Silently. For months.

## Apps Script

**One global scope across all files.** Two functions with the same name — the last one silently wins. The test harness checks for this.

**Triggers run as their owner, whatever the deployment's "Execute as" setting says.** This is the single most expensive thing in this project's history: the nightly job cached an *admin* render of the page and served it to the whole team, which defeated the admin gate completely and could not be fixed by changing the deployment. Both audiences are cached now.

**`logError_` and `logRun_` must never throw.** Nineteen of them sit inside catch blocks in the render path. A throw from inside a handler escapes the catch and takes the whole page down.

**The 6-minute execution ceiling.** `setupSheets` once timed out from calling `getSheetByName` inside a 70-iteration loop.

**`everyDays` / `onMonthDay` / `onWeekDay` are mutually exclusive.**

## LeadSimple

Raw `Authorization` header — **every other scheme returns "Invalid Access Token"**, which reads like a vendor problem and is not. Records are metered at 1000/minute, so pacing must count records rather than requests. Step names are `step.description`; there is no `step.name`, and reading it returned `undefined` on all 57,913 tasks.

## ShowMojo

`Token token="KEY"` auth — a Bearer header returns 401. The payload is wrapped: `{"response": {...}}`.

## Free-text fields will lie to you

`inspection_name` has 1,039 distinct values. A **negative** filter (exclude move-outs) accepted 453 records — 17% of every inspection on the account — including `MII walkthrough` and `Pre-MOI`, and reported units as compliant that nobody had inspected. Match **positively** on `annual|AI`, and confirm the name.

The same problem blocks "Vacancy Visits" and is why `DATA_STANDARDS.md` asks for eight fixed inspection templates.

## Rendering

**Every view is in the DOM at once.** An entrance animation that fires on load plays for tabs nobody has opened and is finished before they get there — it has to re-arm per view, with a forced reflow between removing and re-adding the class.

**Any number that has to match the layout must be measured from the layout.** A hardcoded `--navh: 60px` was right for one row of tabs and wrong for twelve, which parked the sticky filter bar on top of the nav.

---

## Compliance

WeLease operates in California. Two rules hold throughout:

**Nothing in this code states a law, a notice period, a rent cap or a deposit rule.** `CFG.DEPOSIT_DISPOSITION_DAYS` is WeLease's stated operating standard, not a legal determination. Anything resembling legal wording belongs in a counsel-approved template, not in a render function.

**Nothing auto-sends.** Insurance chases, notices and owner emails are all compose-then-log: a person sends, and the log records it separately. The dashboard cannot know whether a draft was actually sent, so it does not claim to.

Owner correspondence uses `owners@weleaseusa.com`, resident correspondence `residents@weleaseusa.com`. No personal addresses appear in anything a landlord or resident sees.

---

## Conventions

**Say why a number is missing.** "No data yet" and "this job failed" must never look the same. `emptyState_` has four kinds for exactly this.

**Measure, don't assume.** Severity, coverage and completeness are computed. A findings list that pads itself stops being read.

**A partial period is labelled, not published.** Running months, clipped windows and unfinished weeks all carry a flag and stay out of headline figures.

**Maintenance notes are admin-only; interpretation notes are for everyone.** "Run `buildX()`" is gated. "September is still running" is not — hiding that would leave the team confidently misreading the page.

**Comments explain why, not what.** Most comments in this codebase record a bug that was actually hit. Leave them; they are the reason it is not re-hit.

---

## Known gaps

- `LS.LEADS_PIPELINE_ID` is unset — lead-to-move-in stays blank until it is filled from `lsListPipelines()`
- Owner vacancy update log — a promise to owners with no record it was kept. Highest-value item outstanding.
- Streamed rendering — first paint is 2–5s and capped by the platform, not by this code
- `unit_turn_category` and `ai_resolved` — populate or retire, WeLease's decision
- Historical rent timing is judged against *today's* rent, which is the only rent the pipeline stores

---

## Docs

| File | What |
|---|---|
| `docs/PROJECT_RULES.md` | Start here when picking this up cold |
| `docs/DATA_STANDARDS.md` | What the team should standardise in AppFolio, and why |
| `docs/TEAM_GUIDE.md` | For dashboard users, not developers |
| `docs/NOTICES_DESIGN.md` | The PDF notice filler and its compliance split |
| `docs/DESIGN_REVIEW.md` | Design system, and an honest read on the platform's limits |

---

## Diagnostics

Every one of these prints to the log and answers a specific question:

```javascript
verifyConnections()        // can we reach all three APIs
verifyProject()            // are all the files present and wired
verifyTriggers()           // daily, weekly, monthly
verifyTeamAccess()         // what a colleague can actually reach
whoAmI()                   // identity, admin status, deployment mode
diagnoseTruncation()       // which tabs sit on a 5,000-row page boundary
reconcileRentIncome()      // ledger against the exported cash flow
reconcileRentTiming()      // receipts against aged receivables
diagnoseNoticeFeed()       // what the notice filler will offer, with cell types
```

When a number looks wrong, run the matching `reconcile*` before changing anything. They compare two independently-built figures and say which direction they disagree in — which is the part that was missing every time this project has argued with itself about a number.

---

*Internal WeLease Property Management tool. Not for distribution.*

# Workforce & Operational Intelligence Platform: Project Charter v2

Revised 12 Oct 2026, with working rules added 19 Oct 2026. Public product name is still to be chosen. Working repo name: `workforce-ops-intelligence`.

## 1. Business problem

A fictional multi-site service operation, **Halden Service Group** (invented name; checked against real companies before publishing), delivers guarding and contact-centre-style services to about 6 clients from 40 sites with roughly 250 employees. Its managers find problems weeks late because performance data lives in scattered spreadsheets: late arrivals, uncovered shifts, incidents and SLA breaches are noticed after the damage is done, and nobody can say with confidence which contracts make money.

The platform must answer six questions:

- **Coverage:** are we delivering the hours each contract sells us (scheduled vs contracted vs worked)?
- **Attendance:** where is punctuality slipping, by site, shift type and employee?
- **Incidents:** which sites produce the most and the most severe incidents, and is that rising week over week?
- **SLA breaches:** which sites and contracts miss their service-level targets, and when?
- **Profitability:** which contracts earn more in billing than they cost in labour (gross margin)?
- **Monday briefing:** what should the Operations Director read in 60 seconds? (the AI summary)

**Users:** Operations Manager (weekly), Account Manager (per contract), Finance/MI Lead (monthly).

**Why this project:** it shows the full MI analyst workflow end to end: messy data in, cleaned, modelled in SQL, reported in Power BI, and explained in plain English.

## 2. Target roles and employers

Three roles only: **Junior Data Analyst, MI Analyst and BI Analyst**. Data Scientist and AI Engineer skills are out of scope here.

| What the ad asks for | Where this project proves it |
| --- | --- |
| SQL: joins, aggregation, subqueries, window functions | Multi-table PostgreSQL schema, 25+ documented queries |
| Advanced Excel: Power Query, PivotTables, lookups | Messy workbook cleaned in Power Query, pivot reconciliation |
| Power BI and DAX | Star schema, time-intelligence measures, drill-through, tooltips, bookmarks |
| Data quality and reconciliation | Error ledger plus pivot-vs-SQL totals that must match |
| Defining KPIs with stakeholders | Metric dictionary with formula, grain, inclusions and exclusions |
| Explaining numbers to non-technical people | One auditable AI executive summary |

**Where to aim**, in priority order: Dominican Republic BPO and contact-centre operators, LATAM-remote analytics roles, Gulf/UAE facilities and outsourcing groups, Europe, and UK MI roles. These employers run shift-based, multi-site operations with SLAs, attendance and incident reporting, which is exactly what this dataset models.

## 3. Clean-room rules

Everything here is designed from scratch, from general industry knowledge and our own invented design.

1. **No old files open.** Do not open, copy from, or screenshot any previous employer's sheets, reports, code or dashboards while building.
2. **Invented everything.** Table names, column names, site names, employee names, KPI formulas and layouts are designed fresh. KPI definitions use plain textbook meaning, written in our own metric dictionary.
3. **Fake data only.** All rows come from a seeded Python script. Names come from a fake-name generator, and sites are invented.
4. **Design log.** `docs/design_log.md` records each design decision and why.
5. **Public wording.** The README and LinkedIn describe it as a synthetic-data project inspired by general operational-reporting experience, and never name a former employer or its clients.
6. **Name check.** The public project uses a different name from any real system. The public product name must pass a quick search for clashes first (ShiftSight was dropped for that reason).
7. **CV wording.** Keep CV claims about any past work general and accurate.
8. **Claims rule.** You only claim a skill on your CV, LinkedIn or in interviews if a file in this repo demonstrates it.

## 4. Synthetic data design

A seeded Python script generates **12 months (1 Jan to 31 Dec 2025)** of data for the invented company: about 6 clients, 8 contracts, 40 sites, 250 employees, roughly 65,000 shifts, 65,000 attendance records and 4,000 incidents, so **more than 130,000 rows** in total. The same seed always gives the same data. Columns are drafts until Phase 1 locks them.

| Table | Purpose | Key columns |
| --- | --- | --- |
| clients | Who we serve | client_id, client_name, sector |
| contracts | What we sell | contract_id, client_id, start_date, end_date, bill_rate_per_hour, contracted_hours_per_week, sla_target_pct |
| sites | Where we deliver | site_id, contract_id, site_name, service_type (guarding or contact centre), region, risk_rating |
| employees | Who works | employee_id, full_name, role, hire_date, employment_type, pay_rate_per_hour, supervisor_id (self-join) |
| shifts | What was scheduled | shift_id, site_id, employee_id, shift_date, start_time, end_time, shift_type, status |
| attendance | What actually happened | attendance_id, shift_id, clock_in, clock_out, minutes_late, hours_worked |
| incident_categories | Lookup | category_id, category_name, is_sla_breach |
| incidents | What went wrong | incident_id, site_id, shift_id, reported_by_employee_id, incident_date, category_id, severity, hours_to_resolve |
| dim_date | Calendar for DAX | date_key, week_start, month, quarter, is_weekend |

**Messy Excel layer.** The first output is deliberately dirty: one shifts sheet per month, mixed date formats, duplicate rows, blanks and "N/A", trailing spaces, inconsistent site-name spellings, numbers stored as text, hours written like "8h30", and a banner row above the headers. The script also writes an **error ledger** listing every injected fault, so we can prove the cleaning caught them all.

**Planted stories.** One high-risk site whose incidents rise from August, night shifts with more lateness, a handful of chronically late employees, and one contract that under-delivers its contracted hours. Also: one contact-centre-style site breaches its SLA mostly on Monday mornings, and one contract is priced so low that its gross margin is thin.

## 5. Metric dictionary and quality layer

Every KPI is defined once, in `docs/metric_dictionary.md`, before any SQL or DAX uses it. Each entry records the name, the business question, the exact formula, the grain, what is included and excluded, the SQL view, the DAX measure, and its reconciliation status. Names must match the calculation: *gross margin* (billing minus labour cost only), never "profit", and *scheduled hours* or *worked hours*, never just "hours".

| KPI | Formula | Grain | Inclusions and exclusions |
| --- | --- | --- | --- |
| Attendance rate | shifts attended ÷ shifts scheduled | shift | Excludes shifts the client cancelled |
| Punctuality rate | on-time arrivals ÷ shifts attended | shift | On time = within a threshold set in Phase 1 |
| Average minutes late | total minutes late ÷ late arrivals | shift | Late arrivals only |
| Shift cover rate | shifts filled ÷ shifts required | shift | Counts shifts covered by another employee |
| Hours delivered vs contracted | worked hours ÷ contracted hours | contract-week | Worked hours come from attendance, not the roster |
| Incident rate | incidents per 1,000 worked hours | site-week | SLA breaches are counted separately |
| SLA breach rate | shifts with an SLA breach ÷ shifts attended | site-week | Only sites with a contract SLA target |
| Gross margin per contract | (worked hours × bill rate) − (worked hours × pay rate) | contract-month | Excludes overheads and agency costs, hence "gross" |

**Quality checks built in**

- **SQL vs Power BI:** every KPI is computed in a SQL view and in DAX, then compared in a reconciliation table. The difference must be zero.
- **JOIN fan-out checks:** after every multi-table JOIN, compare row counts and SUM totals with the base table before trusting the result.
- **Pivot vs SQL:** PivotTable totals from the cleaned Excel must match the SQL load.
- **Error ledger:** every injected fault in the messy Excel is listed, so we can prove the cleaning caught it.

## 6. Tech stack

| Tool | Role | Why |
| --- | --- | --- |
| Python 3.11 (pandas, numpy, Faker) | Generates the data, calls the LLM | Used only for generation and the AI step |
| Excel with Power Query and PivotTables | Cleans the messy workbook and sanity-checks it | Every MI and BI job screens for it |
| PostgreSQL (with pgAdmin) | The relational database | Closer to what employers run than SQLite; Power BI connects natively |
| Power BI Desktop and DAX | Model, measures and dashboard | Target roles list it most often |
| Git, GitHub, VS Code | Version control and editing | Incremental commits from day one |
| LLM API (provider-agnostic) | The executive summary | Small and swappable; key kept in a `.env` file that is never committed |

**Fallback:** if Power BI will not connect to PostgreSQL, we load the cleaned CSVs into Power BI directly, and the SQL work stays unchanged.

**Cost note:** everything is free except the LLM call, which costs a few cents. If budget is a problem we use a free-tier API.

## 7. Architecture

Data flows one way through five stages, with two reconciliation checkpoints (R1 and R2).

```text
Python generator (seeded)
  -> data/raw/messy workbook (.xlsx) + error ledger
Excel: Power Query cleaning -> PivotTable sanity checks
  -> data/clean/*.csv
  R1: pivot totals must equal SQL totals
PostgreSQL: schema + load -> JOIN / CTE / window queries -> KPI views
  R2: SQL view results must equal DAX measures
Power BI: star schema over the views -> DAX -> pages
  (drill-through, tooltip, bookmarks)
AI step: KPI view -> LLM -> summary saved with its input numbers
  -> shown beside the numbers in Power BI
```

The star schema has one fact table for shifts (joined to attendance) and one for incidents. Both share the dimensions: dim_date, sites, employees and contracts.

## 8. Scope: MVP and advanced layer

**MVP: must ship (cut line after Phase 5)**

- Multi-table PostgreSQL schema loaded from the seeded generator (130,000+ rows), with JOINs, subqueries, CTEs, window functions and 3 or more views
- Messy Excel cleaned with Power Query, with PivotTable reconciliation against SQL
- Metric dictionary, JOIN fan-out checks and SQL-vs-Power BI reconciliation
- Power BI star schema, core DAX (week-over-week, 4-week rolling average), 3 pages and a working site-to-employee drill-through
- A basic README with screenshots and the honest synthetic-data note

**Advanced layer (Phases 6 to 8)**

- Power BI: tooltip page, 3 or more bookmarks, month-to-date and year-to-date measures, and 2 more pages (incidents and SLA)
- One AI executive summary, with the KPI numbers shown beside it
- GitHub profile README, one consolidated PDF, and interview prep

**Out of scope (on purpose)**

- RAG, vector databases, agents, or any model training
- Web apps, Streamlit, live Google Sheets connections, Tableau
- Year-over-year comparisons (one year of data), and publishing to the Power BI web service, which typically needs a work or school account. We deliver the `.pbix` file, screenshots and a short screen recording instead.

**The AI feature, precisely.** A Python script reads the week's KPI figures from a SQL view, sends only those numbers to the LLM, and gets back a summary of about 120 words. The summary is saved in a table next to the exact numbers it was given. A check script confirms every number in the summary appears in the input. The Power BI page shows the text and the KPI table side by side, so a reader can audit it. The model never sees raw rows and never does arithmetic.

## 9. Success criteria

- [ ] **Git:** at least 40 incremental commits, a tag per phase, and every file in a real runnable format (`.sql`, `.csv`, `.md`)
- [ ] **Data:** seeded generator producing 130,000+ rows, a messy Excel workbook and an error ledger, with every injected fault caught in cleaning
- [ ] **SQL:** 25 or more documented queries, including 8 multi-table JOINs, 5 subqueries or CTEs, 8 window functions (RANK, LAG, NTILE, running totals), 1 self-join and 3 views. You write each one first, and each JOIN has a fan-out check
- [ ] **Metric dictionary:** every KPI has a formula, grain, inclusions and exclusions, and a name that matches its calculation
- [ ] **Reconciliation:** PivotTable totals equal SQL totals, and every KPI in SQL equals its DAX measure (difference zero)
- [ ] **Power BI:** star schema, 25 or more measures in one measures table, week-over-week and rolling-average measures, 5 pages, a site-to-employee drill-through, a tooltip page and 3 or more bookmarks
- [ ] **AI:** summary generated from real KPIs, shown beside the numbers, with the number-check script passing
- [ ] **Delivery:** findings-first README (metric dictionary, screenshots, honest synthetic-data note), the `.pbix`, a GitHub profile README, and one consolidated PDF with every step (why, how, where), challenges and fixes, and interview prep
- [ ] **Claims and understanding:** every CV claim is backed by a file in the repo, you score 80% or higher on each phase quiz, and you can explain any KPI or query in about two minutes

## 10. Roadmap and checkpoints

One small step at a time, with exact instructions. **Attempt-first:** you write the SQL and DAX before the mentor shows a version, then it is reviewed. Each phase ends with a checkpoint, a recap and a quiz. The plan assumes about 2 to 3 hours a day, part-time, and finishes around 25 Nov 2026 (re-baselined on 19 Oct). That leaves time for applications and interviews.

| Phase | Window | What we build | Checkpoint | CV or repo output |
| --- | --- | --- | --- | --- |
| 1 | 19–22 Oct | Repo and Git setup, ERD, data dictionary, metric dictionary v1, design log | Repo on GitHub; every table explained | `metric_dictionary.md`, ERD image, first commits |
| 2 | 23–29 Oct | Python generator: clean tables, messy Excel, error ledger | Same seed gives identical files; 130,000+ rows | `generator/` code, error ledger |
| 3 | 30 Oct–3 Nov | Excel: Power Query cleaning and PivotTable sanity checks | Cleaned counts match the error ledger | Power Query step log, pivot workbook |
| 4 | 4–10 Nov | PostgreSQL load, then JOINs, CTEs, windows, views, fan-out checks | R1: pivot totals equal SQL totals | 25+ queries in `sql/`, views, LinkedIn midpoint post |
| 5 | 11–15 Nov | Power BI MVP: star schema, core DAX, 3 pages, drill-through | R2: DAX equals SQL. MVP cut line | Dashboard screenshot, `measures.md`, reconciliation table, README v1 |
| 6 | 16–18 Nov | Power BI advanced: tooltip, bookmarks, MTD/YTD, incidents and SLA pages | Full click-through demo works | Demo screenshots or short recording |
| 7 | 19–20 Nov | AI executive summary and number-check | Check script passes | `ai/` script, summary beside its numbers |
| 8 | 21–25 Nov | Findings-first README, profile README, final PDF, interview prep | Fresh clone runs end to end | Final README, profile README, PDF |

**Cut line.** After Phase 5 the project is publishable (SQL, Excel, a working drill-through, a basic README). Phases 6 to 8 are the advanced layer. If time slips, the AI step and the extra pages go first, then the tooltip and bookmarks, and nothing in the MVP is cut.

**Weekly milestones.** End of week 1, the repo, ERD and metric dictionary are public; end of week 2, the generator and messy workbook; end of week 3, the SQL layer, with a LinkedIn post at the midpoint; end of week 4, the Power BI MVP; end of week 5, the advanced layer and final documents.

## 11. Git and GitHub plan

We commit after every small working step and push at the end of each session. Messages follow a simple pattern such as `docs: add project charter`, `data: add shift generator`, `sql: add window-function queries`. Each phase gets a short branch merged into `main`, and a tag (`v0.1` and so on) at each checkpoint.

Planned folder layout:

```text
workforce-ops-intelligence/
  README.md             findings first
  .gitignore            blocks .env and large temp files
  .env.example          shows which keys are needed, no real values
  docs/                 charter, design_log, data_dictionary, metric_dictionary
  generator/            Python data generator and config
  data/
    raw/                messy Excel workbook, error ledger
    clean/              cleaned CSVs
  excel/                Power Query notes, pivot check workbook
  sql/                  01_schema ... 06_views, reconciliation checks
  powerbi/              .pbix, measures.md, screenshots
  ai/                   executive summary script and prompt
```

A `.pbix` file is binary, so Git cannot show what changed inside it. To keep the history readable, every DAX measure is also copied into `powerbi/measures.md` as text.

**Git hygiene.** Stage files by name (never `git add .`), run every command from the project folder, and keep the database, notebooks and private PDFs out of commits. The `.gitignore` enforces this.

## 12. Risks and fallbacks

| Risk | What we do about it |
| --- | --- |
| Scope creep (four tools plus AI) | Fixed phase order, with the AI step last and droppable |
| Clean-room contamination | The rules in section 3, plus the design log |
| Data looks fake and flat | Seeded patterns: weekend peaks, night-shift lateness, risky sites, chronic late employees |
| Overwhelm or distraction | One concept per step, checkpoints, quizzes before moving on |
| Power BI cannot reach PostgreSQL | Load the cleaned CSVs directly |
| LLM cost | Provider-agnostic design, free-tier option |
| Job-search time pressure | Publish at the cut line, finish later |
| The 5-week plan is tight alongside PL-300 and SQL-platform practice | Fixed daily slot for that practice; MVP cut line after Phase 5; advanced layer trimmed first |
| Overclaiming skills | Claims rule: only what a file in the repo demonstrates |
| JOINs silently inflating totals | Fan-out check after every multi-table JOIN; R1 and R2 reconciliation |
| Job-search deadline | Project finishes about 25 Nov, so the remaining weeks go to applications and interviews |

## 13. Working rules (addendum, 19 Oct 2026)

These come from the StayIQ correction and override anything above that conflicts.

1. **Metric dictionary first.** Before any SQL, define every KPI: exact formula, grain, inclusions and exclusions, and which word is justified (exposure vs loss, gross vs net, rate vs count).
2. **Wording matches the calculation.** Check every headline label against its formula before it goes in the README, dashboard or CV.
3. **No unstated units.** Even synthetic data states its currency, time zone and shift hours, decided in Phase 1 and written in the data dictionary.
4. **Reconcile SQL against Power BI.** Every headline number must match in both.
5. **JOIN fan-out checks.** After every JOIN, confirm row counts and totals did not silently inflate.
6. **Keep superseded logic, labelled.** Old queries are not overwritten.
7. **One number, every place.** When a number changes, update SQL, README, dashboard, CV and LinkedIn.
8. **Claim only what the repo demonstrates.**
9. **Git hygiene.** Stage files by name, run commands from the project folder, and keep the database, notebooks and private PDFs out of commits.
10. **No `%%sql`.** Use a normal client such as pgAdmin or DBeaver for PostgreSQL, and `pd.read_sql_query` only for SQLite.
11. **Weekly milestones**, with a LinkedIn post at the midpoint.
12. **Every step produces something** for the CV, the repo or the profile README.

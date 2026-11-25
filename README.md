# JobShot — Audit, Problems, Solutions and Roadmap

> **Project:** JobShot — Intelligent Job Discovery, Applicant Profile & Application Automation
> **Owner:** Jayesh Borse
> **Document date:** 2026-11-25
> **Status:** Phase 1 in progress · Task 1.1 ✅ (2026-11-25) · Task 1.2 ✅ · Task 1.3 ✅ (2026-11-26) · Task 1.4 dropped (no old data) · Next: Task 1.5
> **Project home:** `D:\Interview\Job_Search\` — all work happens here. No separate project folder is created.
> **Code paths** in this document (for example `db.js`, `profile/`, `db/migrations/`) are relative to `D:\Interview\Job_Search\_pipeline\`.

---

## Table of contents

1. [Purpose of this document](#1-purpose-of-this-document)
2. [Product goal](#2-product-goal)
3. [What exists today (audit)](#3-what-exists-today-audit)
4. [Problems found](#4-problems-found)
5. [Token consumption: measured root cause](#5-token-consumption-measured-root-cause)
6. [Solution overview](#6-solution-overview)
7. [Target architecture](#7-target-architecture)
8. [Which work uses Claude, and which does not](#8-which-work-uses-claude-and-which-does-not)
9. [MCP and connector plan](#9-mcp-and-connector-plan)
10. [Database upgrade plan](#10-database-upgrade-plan)
11. [Target repository layout](#11-target-repository-layout)
12. [Development roadmap (phases and tasks)](#12-development-roadmap-phases-and-tasks)
13. [Security and privacy rules](#13-security-and-privacy-rules)
14. [Compliance rules (websites, emails, applications)](#14-compliance-rules-websites-emails-applications)
15. [How we measure success](#15-how-we-measure-success)
16. [Expected results (realistic)](#16-expected-results-realistic)
17. [Open decisions](#17-open-decisions)
18. [How we work together (implementation method)](#18-how-we-work-together-implementation-method)
19. [Interview explanation guide](#19-interview-explanation-guide)

---

## 1. Purpose of this document

This file records:

- what the existing JobShot project contains,
- what is working, broken or missing,
- why job discovery used about 85% of the Claude session allowance (measured, not guessed),
- how we will fix each problem,
- the complete, phased plan to upgrade the project without rebuilding working parts.

It is the reference for every implementation session. Keep it updated as tasks finish.

---

## 2. Product goal

Help a fresher find relevant jobs and apply with as little repetitive work as possible:

```
Find jobs → Match jobs → Prepare resumes → Fill application forms
         → Send recruiter emails → Track applications
```

The time saved goes to Core Java, Spring Boot, DSA, system design and interview preparation.

**Guiding principle:** scripts do repetitive work; Claude only does work that needs reasoning.

---

## 3. What exists today (audit)

The project is spread across four places that are not connected to each other.

| Location | Contents | State |
|---|---|---|
| `D:\Interview\Job_Search\_pipeline\` | **The real job engine** — about 2,000 lines of Node.js | Works, but no git, no database; CSV files act as the database |
| `D:\Interview\Job_Search\` (other folders) | `SHOOT_PROTOCOL.md`, `CLAUDE.md`, JBD/FSD master resumes, trackers (`All_Jobs.csv` 225 rows, `Historical_Jobs.csv` 366 rows), daily reports, 65+ tailored resume folders for 24 Nov | Large amount of generated output |
| `D:\Jayesh_Job_Applications\` (older folder) | Original master resume `.tex`, 10 hand-tailored resumes (Batch 01–03), 10-row tracker CSV | Separate tracker from 16 Nov, not linked to the one above |
| `D:\n8n Project\Workflows\` | `Job Collector.json`, `AI_Job_Processing.json` (June 2026) | Both broken (see section 4) |
| PostgreSQL 18 (Windows service `postgresql-x64-18`) | `public.jobs`, `public.applications` | Service **stopped**; 5 junk rows in `jobs` |
| Docker Desktop / n8n | — | Docker **not running** at audit time |
| Dashboard | Claude artifact "Jayesh's Job Search Dashboard" | Static page with 138 jobs from 24 Nov built in (23 Apply First / 24 Good Match / 91 Stretch) |

### 3.1 Existing pipeline scripts (`_pipeline/`)

| Script | What it does | Uses an LLM? |
|---|---|---|
| `harvest.js` | Collects jobs from LinkedIn guest pages, Internshala, Lever, SmartRecruiters and Workday feeds | No |
| `aggregators.js` | Reads 12 fresher-site RSS feeds, finds the employer's apply link, checks whether the posting is closed | No |
| `score.js` | Keyword-based fit score (0–100) and bucket (Apply First / Good Match / Stretch) | No |
| `resume_lib.js` | LaTeX resume builder using an approved bullet library (facts from the master resume only) | No |
| `shoot_run.js`, `daily.js` | Daily run: analysis, tailoring, validation, tracker updates | No |
| `reports.js`, `shoot_reports.js` | Markdown reports | No |
| `share_pdf.js` | Shareable daily PDF of verified jobs | No |
| `qc.js` | Quality checks | No |

**Important finding:** the collection, scoring and tailoring logic is already deterministic. It does not need Claude to run.

### 3.2 Existing database schema (verified live, Task 1.1, 2026-11-25)

PostgreSQL 18.4 · database **`jobhunter`** · schema `public` only. Other databases on the server (`library_db`, `securitydb`) are unrelated and not touched.

```
public.jobs            101 rows
  id serial PK, company_name varchar(255) NOT NULL, job_title varchar(255) NOT NULL,
  location varchar(255), job_url text UNIQUE (jobs_job_url_unique),
  job_description text, match_score int, status varchar(50) DEFAULT 'NEW',
  created_at timestamp DEFAULT now, strengths text, missing_skills text

public.applications    2 rows (status GENERATED)
  id serial PK, job_id → jobs(id) ON DELETE CASCADE, recruiter_id → recruiters(id),
  email_subject, email_body, application_date DEFAULT now, followup_date,
  status varchar(50) DEFAULT 'DRAFT_CREATED', recruiter_email,
  generated_subject, generated_email

public.recruiters      0 rows   (not visible in the n8n workflow files)
  id serial PK, company_name, recruiter_name, email, linkedin_url, created_at
```

**Data quality of the existing rows:**

| Check | Result |
|---|---|
| Source of jobs | 100 from remoteOK.com, 1 test row (`test-job.com`) |
| Company name | Mostly `Unknown` (RSS feed has no company field) |
| Relevant titles | Few. Examples: "Upper Elementary Online Teacher", "Medical Scribe", "Administrator" |
| Scored (`match_score > 0`) | 2 of 101 |
| Garbled text (encoding errors) | 42 of 101 |
| Date range | 18–19 June 2026 |

This confirms problem P2: the June collector stored unrelated global jobs. The rows are kept untouched; the new `jobshot` schema is separate.

**Personal data found in code:** a phone number and email address are hard-coded in 5 scripts (`resume_lib.js`, `daily.js`, `reports.js`, `share_pdf.js`, `shoot_run.js`). This is fine in a local git repo, but must move to the profile/`.env` (Phase 2) **before** the repo is ever pushed to GitHub.

### 3.3 What works correctly

- Collection from public ATS feeds (Lever, SmartRecruiters, Workday JSON) and RSS feeds.
- Rule-based verification: closed-posting detection, follow-up checks on Lever, Greenhouse, SmartRecruiters and LinkedIn links.
- Keyword fit scoring and bucketing (25 Nov run scored 152 jobs).
- Resume tailoring that only uses facts from the master resume; compile, one-page and validation checks pass.
- Written truthfulness rules (never invent skills, metrics or experience).
- Shareable daily PDF and Markdown reports.

---

## 4. Problems found

### P1 — Job discovery consumes most of the Claude session allowance
The pipeline was run *inside* one Claude conversation that lasted 9 days. See section 5 for measurements.

### P2 — n8n "Job Collector" workflow is broken
- `Job Normalizer` returns `null` for rejected jobs but **never returns accepted jobs**, so valid jobs never reach the database.
- It never maps `title → job_title`, `link → job_url`, or fills `company_name` (a required column). This is why the table contains rows with score 0 and garbled text such as "â".
- Its sources (RemoteOK, WeWorkRemotely) are global remote boards with mostly senior roles — a poor fit for an Indian fresher.

### P3 — n8n "AI_Job_Processing" workflow is broken
- `SELECT … LIMIT 1` with no `WHERE status = 'NEW'` → it re-scores the same row on every run.
- Only the title and location are sent to Gemini, not the job description → the score is meaningless.
- Candidate skills are hard-coded in the prompt.
- The API key is placed in the URL query string, where it ends up in logs.

### P4 — Dashboard is static
It contains the 24 Nov data built into the page. The 25 Nov run (85 new jobs) is missing. It is not connected to any database.

### P5 — Too many duplicate resumes
`SHOOT_PROTOCOL.md` requires a tailored resume for **every** Apply First and Good Match job → 46 resumes per day, most for jobs that will never be applied to.

### P6 — Data is scattered and unversioned
Three folders, two separate trackers, CSV files as the database, no git history, no single source of truth.

### P7 — Website terms-of-service risk
- `harvest.js` uses LinkedIn's unofficial guest pages (it hit HTTP 429 rate limits).
- The 25 Nov run had Claude open Naukri 74 times in the browser, even though the workspace `CLAUDE.md` already says Naukri blocks automated access.
- Both conflict with those websites' terms.

### P8 — Missing features
Database storage of jobs, Master Applicant Profile, application form automation, HR contact discovery, cold email sending, status tracking, audit trail.

---

## 5. Token consumption: measured root cause

Measured from Claude Code session logs in `~/.claude/projects/` using a log-analysis script (to be saved as `_pipeline/session_usage.js`).

### 5.1 Discovery session ("Pan-India fresher job discovery automation", 16–25 Nov)

| Metric | Value |
|---|---|
| API turns (all on Opus) | **392** |
| Context size across the session | 73k → 173k → 294k → **483k** → 695k → **849k** (peak **965k**), then compacted |
| Cache-read tokens | **183.4 million** (**98.3%** of all input processed) |
| Cache-write tokens | 3.2 million |
| Output tokens | 0.49 million |
| Browser visits | naukri.com 74 times |

### 5.2 Why it was so expensive

Each Claude turn re-reads the **entire conversation so far**. When one conversation grows to 500k–900k tokens and continues for hundreds of turns, the cost multiplies.

```
cost ≈ (number of turns) × (average context size)
     ≈ 392 × ~470k  ≈ 183 million tokens re-read
```

Even the starting context (system prompt + tool definitions ≈ 73k tokens) was paid 392 times.

### 5.3 What filled the context

| Cause | Tokens added | Approx. re-read cost later |
|---|---|---|
| Bash output dumps (printing parts of `scored.json` and reports) | 268k | ~51 M |
| Claude writing / rewriting pipeline scripts during runs (`harvest.js`, `score.js` twice, `daily.js` 28 KB…) | 192k | ~39 M |
| Browser scraping of Naukri (`browser_batch` + JavaScript calls) | 172k | ~18 M |
| Edits, PDF reads, web search and fetch | ~140k | ~27 M |
| Four large pasted protocol prompts, task-list and wrap-up overhead | — | ~15 M+ |

### 5.4 Tailoring session in this folder ("Job application organization and research")

54 turns, peak context 286k, 10.2 M cache-read tokens. Main cost: reading the 11-page job-report PDF **as images** (about 2 million characters per read).

### 5.5 Conclusion

The scraping was not the problem — it was already scripted. The problem was:

1. one very long conversation instead of short, fresh sessions,
2. Claude doing orchestration work a scheduler should do,
3. large outputs printed into the conversation,
4. Claude rewriting code during routine runs,
5. browser scraping by Claude,
6. reading PDFs visually instead of reading structured data.

---

## 6. Solution overview

| Problem | Solution |
|---|---|
| P1 Token usage | Move collection, dedup, scoring, PDF and dashboard updates to scheduled scripts (0 Claude tokens). Claude only sees a compact shortlist in a short, fresh session. |
| P2 Collector broken | Replace the broken n8n collector with a scheduled run of the existing working Node collectors, writing to PostgreSQL. |
| P3 AI processing broken | Process only `status = 'NEW'` rows, send the job summary, read skills from the profile, move the API key to a header. |
| P4 Static dashboard | Generate the dashboard from PostgreSQL with the same design. |
| P5 Duplicate resumes | Tailor only for jobs you shortlist; reuse when the content hash matches. |
| P6 Scattered data | One git repo, one PostgreSQL database as the source of truth. Old CSV history is **not** imported (owner decision); the product starts from the 25 Nov 2026 run. |
| P7 ToS risk | Stop LinkedIn/Naukri scraping; parse their job-alert **emails** from Gmail instead. Prefer official ATS APIs. |
| P8 Missing features | Phases 2–6 (profile, form automation, contacts, email, tracking). |

### 6.1 Token optimization principles (applied everywhere)

1. Job discovery does not depend on Claude.
2. A scheduler (n8n or Windows Task Scheduler) runs collection.
3. Public ATS APIs and RSS feeds are preferred over web pages.
4. Parsing and deduplication happen in scripts and SQL, before any AI.
5. PostgreSQL stores all jobs and history.
6. Only new, relevant jobs reach Claude.
7. Claude gets a short summary (≤300 characters), never full job descriptions in bulk.
8. No loading of entire websites when structured data exists.
9. Reusable prompt templates.
10. Scripts handle repetitive browser work (Playwright library, not Claude clicking).
11. Tools print compact JSON, never large dumps.
12. `CLAUDE.md` holds project context, so it is never re-explained.
13. Discovery, tailoring and application are separate workflows.
14. Each Claude session starts fresh and continues from database checkpoints.
15. Few MCP servers connected (each adds tool definitions paid on every turn).
16. PDFs are never read visually; structured data is read instead.

---

## 7. Target architecture

```
 ┌──────────────── ZERO-LLM ZONE (runs automatically) ──────────────────┐
 │ Scheduler (n8n, or Windows Task Scheduler)                            │
 │   ├─ _pipeline/  (existing harvest + aggregators, logic unchanged)    │
 │   │    Greenhouse · Lever · Ashby · SmartRecruiters · Workday JSON    │
 │   │    fresher RSS feeds                                              │
 │   │    Gmail job-alert emails (LinkedIn / Naukri / Internshala        │
 │   │    alerts parsed instead of scraped)                              │
 │   ├─ normalize → canonical_key → DEDUP in SQL (before any AI)         │
 │   ├─ verify (existing rules) → score.js (existing, deterministic)     │
 │   └─ regex contact extraction from job text ("send CV to hr@…")      │
 │                         ▼                                             │
 │              PostgreSQL 18  (schema `jobshot` = source of truth)      │
 │                         ▼                                             │
 │   dashboard/build.js → same dashboard design, data from database      │
 │   share_pdf.js (existing) → daily PDF                                 │
 │   n8n → daily / weekly digest                                         │
 └───────────────────────────────────────────────────────────────────────┘
          ▼  compact shortlist (~30 rows × ~60 tokens, no full JDs)
 ┌──────────────── CLAUDE ZONE (short, fresh sessions only) ────────────┐
 │ Review shortlist · edge-case judgement · screening answers that need │
 │ thought · cold-email personalization · development work              │
 └───────────────────────────────────────────────────────────────────────┘
          ▼
 Resume:   pick JBD or FSD master → tailor ONLY shortlisted jobs
           → reuse if content hash matches (no duplicates)
 Profile:  Master Applicant Profile (local form → PostgreSQL → PDF export)
 Apply:    Playwright scripts + ATS adapters + Field Mapping Engine
           → fill form, upload resume, flag missing fields
           → STOP. You review and click Submit yourself.
 Outreach: contacts (source + verified / unverified / unavailable)
           → email drafts → your approval
           → n8n Gmail/SMTP sends ONLY from your configured sender
             (sender shown before every send)
 Tracking: status_history + Gmail confirmation parsing → dashboard
```

### 7.1 Morning → evening flow (final product)

| Time | What happens | Claude involved? |
|---|---|---|
| Morning (automatic) | Collect → dedup → verify → score → store → dashboard + PDF updated | No |
| Morning (you) | Short session: review compact shortlist, pick jobs | Yes (small) |
| Resume step | Select master, tailor only shortlisted jobs | Only for wording choices |
| Application session | Playwright opens forms, fills known fields, uploads resume, flags gaps; you submit | Only for unmapped questions |
| Outreach | Contacts found with source; drafts generated; you approve; n8n sends | Drafting only |
| Evening (automatic) | Gmail confirmations update status; daily summary | No |

---

## 8. Which work uses Claude, and which does not

| Operation | Runs where | Claude tokens |
|---|---|---|
| Fetch jobs from ATS APIs / RSS / alert emails | Node scripts via scheduler | 0 |
| Normalize, deduplicate, verify | Scripts + SQL | 0 |
| Fit scoring and bucketing | `score.js` | 0 |
| Store jobs and history | PostgreSQL | 0 |
| Dashboard and daily PDF | Generator scripts | 0 |
| Daily / weekly reports | n8n | 0 |
| Confirmation-email status updates | n8n Gmail node | 0 |
| Filling known form fields | Playwright scripts | 0 |
| Shortlist review and judgement | Claude Code, fresh session | Small |
| Answering new / unusual screening questions | Claude | Small, per question |
| Cold-email personalization | Gemini in n8n (free tier) or Claude for a few | 0–small |
| Building and fixing the system | Claude Code | Development cost |

---

## 9. MCP and connector plan

**Rule:** every connected MCP server adds tool definitions that are paid on every turn. Default to a script or CLI; use an MCP only where it clearly helps.

| Integration | Decision | Used for | Cost / maintainer | Effect on Claude context | Risks |
|---|---|---|---|---|---|
| PostgreSQL MCP | ❌ Skip. Use a small `jobshot` CLI or `psql` with compact queries | Shortlist review | The official reference server was archived and had a read-only SQL-injection bypass; community alternatives exist | CLI **reduces** context (prints only what is needed, no tool schema) | Broad database access |
| Playwright MCP (Microsoft) | ⚠️ Development only — explore a new ATS form once to write its adapter | Phase 3 adapter building | Free, official | **Increases** context (page snapshots are thousands of tokens); never used per application | Browser access; use an isolated profile |
| Playwright library | ✅ Runtime engine | Every autofill | Free, official | 0 Claude tokens | Runs locally under your control |
| n8n native nodes (Schedule, Postgres, Gmail, SMTP, IMAP) | ✅ Core | Scheduling, email, confirmations, digests | Free, self-hosted | 0 | Credentials in n8n's encrypted store |
| n8n MCP | ⚠️ Optional, later | Triggering workflows from Claude | Free (built-in in recent n8n; community server also exists) | Slight increase | Can run workflows |
| GitHub | ✅ `git` + `gh` CLI, no MCP | Source control, private repo | Free | None | Never commit `.env`, profile data or resumes |
| Gmail | ✅ n8n Gmail node (OAuth) or SMTP with app password — **custom sender only** | Sending, confirmation parsing | Free | 0 | Claude's Gmail connector is **not** used for sending (tied to the Claude account identity) |
| Google Drive | ❌ Not needed | — | — | — | — |
| Google Sheets | ❌ Not needed now (optional reporting later) | — | — | — | — |
| Context7 | ⚠️ Optional, development only | Current Playwright / n8n docs | Free tier, community (Upstash) | Small increase per call | Low |
| Gemini API (already in n8n) | ✅ Keep for bulk email drafts | Phase 5 | Free tier | 0 Claude tokens | Free-tier prompts may be used by Google — send job data and skills only, never phone or address |

---

## 10. Database upgrade plan

### 10.1 Rules

- Inspect the live schema first (Task 1.1).
- New schema `jobshot`; existing `public.jobs` and `public.applications` stay untouched.
- Numbered, **additive-only** migrations in `db/migrations/` (`V001__…sql`, `V002__…sql`).
- No `DROP`, no destructive `ALTER`, no deletion of existing rows without your approval.

### 10.2 New tables

| Table | Key fields | Purpose |
|---|---|---|
| `companies` | id, company_name, normalized_name (UNIQUE), company_website, career_page_url, industry, company_size, headquarters, logo_url, linkedin_url, ats_type, ats_slug, created_at, updated_at | One row per company; dedup by normalized_name; enables ATS reuse. Descriptive fields filled only from public data, never invented |
| `jobs` | company_id, title, location, work_mode, exp_min, exp_max, salary_text, skills[], description, summary (≤300 chars), description_hash, posted_at, first_seen_at, last_seen_at, verified_at, verification_status, apply_url, source, source_job_id, **canonical_key UNIQUE**, fit_score, bucket, missing_skills[], status | Single job record; dedup key |
| `job_sources` | job_id, source, source_url, seen_at | Every place a job was found |
| `collection_runs` | source, started_at, finished_at, fetched, new_count, dup_count, errors | Checkpoints and metrics |
| `master_resumes` | track (JBD/FSD), version, tex_path, pdf_path, approved_at | 2–3 master versions |
| `resume_versions` | job_id, master_id, tex_path, pdf_path, **content_hash UNIQUE**, created_at | Job ↔ resume link; prevents duplicates |
| `profile_personal` | legal name, preferred name, email, mobile, alt mobile, nationality; DOB / gender optional and encrypted | Section A |
| `profile_address` | current / permanent address, city, state, postal code, country (encrypted) | Section B |
| `profile_education` | degree, university, college, start/end, graduation year, CGPA/percentage, backlogs | Section C |
| `profile_experience` | company, title, type (job/internship), start, end, responsibilities | Section D |
| `profile_skills` | category (language/framework/database/tool/cloud/other), name | Section E |
| `profile_projects` | name, description, stack, github_url, live_url, role | Section F |
| `profile_links` | type (LinkedIn/GitHub/portfolio/other), url | Section G |
| `profile_preferences` | designations[], locations[], work_mode, relocation, employment_type, expected_salary, notice_period, joining_date | Section H |
| `screening_answers` | canonical_key, question_patterns[], answer, needs_customization, last_used | Section I |
| `profile_history` | table, field, old_value (redacted if sensitive), new_value, changed_at | Version history |
| `field_mappings` | ats_type, normalized_label, selector_hint, profile_key, confidence, source (manual/learned), uses | Field Mapping Engine memory |
| `applications` | job_id **UNIQUE**, resume_version_id, status, autofill_report (JSON), missing_fields[], submitted_at, confirmation_ref | Prevents duplicate applications |
| `contacts` | company_id, job_id, name, email, phone, source_url, verification_status (verified/unverified/unavailable) | HR contacts with evidence |
| `emails` | job_id, contact_id, sender_address, subject, body, attachment_path, status (draft/approved/sent/failed), approved_at, sent_at, message_id | Outreach history |
| `status_history` | job_id, from_status, to_status, changed_at, actor, note | Tracking timeline |
| `audit_log` | action, entity, entity_id, at, actor, details (no personal data) | Audit trail |

### 10.3 Application statuses

`Discovered → Verified → Shortlisted → Resume Ready → Application Pending → Autofill Completed → Awaiting Approval → Applied → Confirmation Received → HR Contacted → Assessment Received → Interview Scheduled`

Terminal / side statuses: `Rejected`, `Failed`, `Duplicate`, `Expired`.

### 10.4 Fields added to every job record (from requirements)

Posting date · company name · designation · location · experience · salary (if available) · required skills · job description · match score · direct application URL · original source · HR name (if public) · HR email (if public) · HR phone (if public) · contact source · contact verification status · personalized cold email · selected resume version · application status.

These are covered by `jobs` + `contacts` + `emails` + `resume_versions` + `applications`, and exposed together through a database view `jobshot.v_job_full`.

### 10.5 Deduplication key

```
canonical_key = ats_type + ':' + source_job_id          (when an ATS ID exists)
              = hash(normalized company + normalized title + city)  (otherwise)
```

Upsert on `canonical_key`; every additional source is recorded in `job_sources`.

---

## 11. Target repository layout

Everything stays inside the existing `D:\Interview\Job_Search\` folder. No new project folder is created.

```
D:\Interview\Job_Search\               (JobShot project root, git repository)
├── CLAUDE.md                          existing — updated with the short review routine
├── SHOOT_PROTOCOL.md                  existing — kept for reference
├── JobShot_Audit_and_Roadmap.md       this file
├── JobShot_Token_Baseline.md          Task 1.1
├── .env.example                       Task 1.1 — variable names only, no secrets
├── .gitignore                         Task 1.1 — excludes .env, profile data, resumes, logs
├── _pipeline\                         existing code folder — all new code goes here
│   ├── harvest.js, aggregators.js, score.js, resume_lib.js,
│   │   shoot_run.js, reports.js, share_pdf.js, qc.js …   existing, kept in place
│   ├── session_usage.js               Claude token-usage analyzer (Task 1.1)
│   ├── db.js, jobmap.js, collect.js, cli.js                Phase 1
│   ├── db\migrations\                 V001__jobshot_schema.sql, V002__… (Phase 1+)
│   ├── dashboard\                     dashboard generator, same design (Phase 1)
│   ├── n8n\                           exported workflow JSON (Phase 1)
│   ├── profile\                       Master Applicant Profile (Phase 2)
│   ├── apply\                         Playwright engine, mapper, ATS adapters (Phase 3)
│   ├── outreach\                      contacts + email drafts (Phases 4–5)
│   ├── tracking\                      status updates, reports (Phase 6)
│   └── data\                          existing
├── Master_Resume\                     existing, unchanged
├── Tailored_Resumes\                  existing, unchanged
├── Daily_Reports\                     existing, unchanged
├── Daily_Jobs_PDF\                    existing, unchanged
├── Application_Tracker\               existing, unchanged
├── Source_Directory\                  existing, unchanged
└── Skill_Development\                 existing, unchanged
```

- Subfolders under `_pipeline\` are created only when their phase starts.
- Existing working scripts are not moved or rewritten.
- `D:\Jayesh_Job_Applications\` is left as it is and is not imported. Old trackers and CSVs stay on disk as archives only.

---

## 12. Development roadmap (phases and tasks)

Effort assumes about 4 focused hours per day. One task at a time; each task is tested before the next starts.

### Phase 1 — Audit and token optimization (5–7 days)

| Task | What | Files | Done when |
|---|---|---|---|
| 1.1 | Start PostgreSQL; inspect the live schema; `git init` in `D:\Interview\Job_Search` with `.gitignore`; save the token analyzer and baseline | `.gitignore`, `.env.example`, `_pipeline/session_usage.js`, `JobShot_Token_Baseline.md` | Live schema documented; repo committed; baseline numbers saved |
| 1.2 | Migration V001: `companies`, `jobs`, `job_sources`, `collection_runs`, `status_history`, `audit_log` | `_pipeline/db/migrations/V001__jobshot_schema.sql` | Migration applies cleanly; existing tables unchanged |
| 1.3 | Add `db.js` and `collect.js` that run the existing harvest, aggregators and score scripts and write to PostgreSQL with dedup | `_pipeline/db.js`, `_pipeline/collect.js` | Run 1 stores jobs; run 2 adds 0 duplicates |
| 1.4 | ~~Import CSV history~~ **Dropped 2026-11-26 (owner decision):** no old data in the product. The database starts from the 25 Nov 2026 run; old CSVs stay on disk as archives | — | Database holds only data from 25 Nov 2026 onwards |
| 1.5 | Compliance switch: disable LinkedIn guest-page and Naukri browser scraping; add Greenhouse/Ashby boards; job-alert email parsing | `_pipeline/harvest.js`, `_pipeline/aggregators.js`, n8n Gmail workflow | Collection works without scraping blocked sites |
| 1.6 | Scheduling: n8n (or Task Scheduler) runs `collect.js`; fix or retire the broken n8n workflows | `_pipeline/n8n/*.json` | Scheduled run completes with 0 Claude tokens |
| 1.7 | `jobshot shortlist --compact` CLI; update the existing `CLAUDE.md` with a short review routine replacing SHOOT-in-Claude | `_pipeline/cli.js`, `CLAUDE.md` | Shortlist prints ~30 rows in compact form |
| 1.8 | Dashboard generator from PostgreSQL, same design as today | `_pipeline/dashboard/build.js` | Dashboard shows current data |
| 1.9 | Measure a review session with the analyzer; compare to baseline | `JobShot_Token_Baseline.md` | Before/after numbers recorded |

- **Database changes:** V001.
- **n8n changes:** collector schedule; old collector retired or fixed.
- **MCP:** none.
- **Testing:** dedup test (run twice), scheduled-run test, token measurement.
- **Acceptance:** collection uses 0 Claude tokens; no duplicates; dashboard current; review session measured.
- **Quality gate (parity test):** the old Claude-driven SHOOT run is retired only after the scheduled pipeline is compared with the 25 Nov 2026 results and keeps the same quality of fresher jobs.

  25 Nov baseline, Apply First jobs by source (33 in total):

  | Source | Apply First jobs |
  |---|---|
  | Naukri | 20 |
  | Internshala | 10 |
  | Lever | 2 |
  | Employer site | 1 |

  The shareable PDF had 150 verified jobs; 66 of them came from Naukri.

  Naukri was collected by Claude's browser, so the script pipeline needs a replacement for it (Naukri job-alert emails parsed from Gmail) before the gate can pass.

### Why the June n8n collector found no fresher jobs (and why that is not the plan)

| June n8n collector | 25 Nov run that worked |
|---|---|
| Sources: RemoteOK and WeWorkRemotely (global, remote, mostly senior) | Sources: Naukri, Internshala, 12 Indian fresher sites, company ATS feeds |
| Bug: `Job Normalizer` never returned accepted jobs, so 0 valid rows were saved | Deterministic Node scripts (`harvest.js`, `aggregators.js`, `score.js`) with fresher/India filters |
| Strict title allowlist of 7 phrases | Broad title rules plus experience parsing (0–2 years) |

The plan does **not** go back to the June workflow. It schedules the same scripts that produced the 25 Nov results.

### Phase 2 — Master Applicant Profile (5–7 days)

| Task | What | Done when |
|---|---|---|
| 2.1 | Migration V002: all `profile_*`, `screening_answers`, `profile_history` | Tables created |
| 2.2 | Profile API (create / read / update, field validation) | API tests pass |
| 2.3 | Local profile form, all sections A–I | Data saves and reloads |
| 2.4 | Missing-information report (unknown → marked missing, never invented) | Report lists every missing required field |
| 2.5 | Version history on every edit | Changes visible in `profile_history` |
| 2.6 | Master Profile PDF export | PDF matches database content |
| 2.7 | Encrypt sensitive fields (DOB, address, demographic) | Values unreadable in raw table |

- **Files:** `profile/*`, `db/migrations/V002__profile.sql`.
- **Rules:** demographic fields optional; nothing fabricated; PostgreSQL is the source of truth, PDF is only an export.
- **Acceptance:** one-time entry, editable, validated, missing fields detected, PDF exported.

### Phase 3 — Application form automation (8–12 days)

| Task | What | Done when |
|---|---|---|
| 3.1 | Migration V003: `field_mappings`, `applications` | Tables created |
| 3.2 | Field Mapping Engine: label normalization → profile key; known mappings reused | Mapping table matches sample forms |
| 3.3 | Greenhouse adapter | Fills a real posting in dry-run mode |
| 3.4 | Lever adapter | Same |
| 3.5 | Ashby adapter | Same |
| 3.6 | SmartRecruiters adapter | Same |
| 3.7 | Workday adapter (account-based; pauses for login) | Same |
| 3.8 | Generic fallback for unknown forms (flags unmapped fields) | Unknown fields listed, not guessed |
| 3.9 | Batch runner: select N jobs, pause / resume / retry, consolidated report | 10-job batch report produced |

**Engine steps per job:**
1. Read shortlisted job from PostgreSQL.
2. Open the direct company application page.
3. Detect the form and its fields with labels.
4. Map fields to the profile (known mappings first).
5. Fill matching values.
6. Upload the job's resume version.
7. Fill reusable screening answers.
8. Flag missing or ambiguous fields.
9. Save progress where the site supports it.
10. Leave the form open for review → **you click Submit**.

**Rules:** company career pages first; LinkedIn Easy Apply and Naukri apply are not targets; CAPTCHA, login or verification → pause for you; never bypass protections; never submit inaccurate data.

**Example field mapping:**

| Website field | Profile field |
|---|---|
| First Name | first_name |
| Last Name | last_name |
| Email Address | email |
| Phone Number | phone |
| University | university |
| Graduation Year | graduation_year |
| Current Location | current_city |
| LinkedIn URL | linkedin_url |
| GitHub URL | github_url |
| Resume Upload | selected_resume_path |

- **MCP:** Playwright MCP only while writing a new adapter; runtime uses the Playwright library.
- **Acceptance:** mappings reused across companies on the same ATS; nothing submitted automatically; batch report shows filled / missing / failed per job.

### Phase 4 — HR contact discovery (2–3 days)

| Task | What | Done when |
|---|---|---|
| 4.1 | Migration V004: `contacts` | Table created |
| 4.2 | Regex extraction from job text ("send CV to …", hr@ / careers@) | Contacts found with source |
| 4.3 | Check the company's careers / contact page for published addresses | Source URL recorded |
| 4.4 | Status: verified / unverified / unavailable | Every contact labelled |

**Rules:** never guess email patterns; never invent phone numbers; no scraping of LinkedIn profiles; record the source of every contact.

### Phase 5 — Cold email and custom email account (3–4 days)

| Task | What | Done when |
|---|---|---|
| 5.1 | Migration V005: `emails` | Table created |
| 5.2 | Configure the custom sender in n8n (Gmail OAuth or SMTP app password) | Test email to yourself works |
| 5.3 | Draft generator (template + personalization) | Drafts saved with status `draft` |
| 5.4 | Approval queue (view, edit, approve) | Only approved drafts can be sent |
| 5.5 | Send workflow: shows sender, attaches the job's resume, records status | Status `sent` / `failed` saved |
| 5.6 | Controlled batch sending after explicit authorization, with a daily cap | Cap enforced |

**Each email includes:** professional greeting · actual job title · company · why the profile matches · relevant skills · one relevant project or experience · interest in the role · professional closing.

**Sender rules (strict):**
- Use only the custom email account you explicitly configure.
- **Never** use the Claude account email as sender.
- Never switch automatically to another sender.
- Credentials only in n8n's credential store or `.env`; never hardcoded.
- Sender address displayed before every send.
- Drafts first; sending only after approval.

### Phase 6 — Application tracking (3–4 days)

| Task | What | Done when |
|---|---|---|
| 6.1 | Migration V006 (if needed): views for tracking | Views created |
| 6.2 | Status machine with allowed transitions; every change logged | Invalid transitions blocked |
| 6.3 | Duplicate-application guard (`applications.job_id` UNIQUE + canonical_key) | Second application blocked |
| 6.4 | Gmail confirmation parsing in n8n → status update | Test confirmation updates status |
| 6.5 | Daily and weekly reports | Reports generated automatically |
| 6.6 | Dashboard sections: completed, pending, failed | Visible on dashboard |

### Phase 7 — Testing and production readiness (3–5 days)

| Task | What |
|---|---|
| 7.1 | Automatic `pg_dump` backups + restore test |
| 7.2 | Log redaction (no personal data, no secrets) |
| 7.3 | n8n error workflow → notification on failure |
| 7.4 | End-to-end test of a full day with no manual steps except approvals |
| 7.5 | Final documentation and architecture diagram |
| 7.6 | Interview write-up (see section 19) |

### Roadmap summary

| Phase | Effort |
|---|---|
| 1. Audit + token optimization | 5–7 days |
| 2. Master Applicant Profile | 5–7 days |
| 3. Application automation | 8–12 days |
| 4. HR contact discovery | 2–3 days |
| 5. Cold email + custom sender | 3–4 days |
| 6. Tracking | 3–4 days |
| 7. Production readiness | 3–5 days |
| **Total** | **~29–42 days** |

---

## 13. Security and privacy rules

- Secrets only in `.env` (git-ignored) or n8n's encrypted credential store.
- `.env.example` lists variable names only.
- PostgreSQL accepts connections from this machine only.
- Dedicated `jobshot_app` database role with minimal permissions.
- pgcrypto encryption for DOB, gender, nationality, addresses.
- Resumes and profile exports stored in git-ignored folders.
- No personal data or secrets in logs.
- Audit log for application actions and email sends.
- Human confirmation for final submission and every email send.
- Gemini API key moved from URL to request header.
- Hardware: PostgreSQL native (already installed), WSL2 capped at ~3 GB, one Playwright browser at a time (8 GB RAM).

---

## 14. Compliance rules (websites, emails, applications)

- Respect website terms, rate limits and access restrictions.
- Never bypass CAPTCHAs, logins or anti-bot protection; pause and hand over to you.
- LinkedIn and Naukri: use their job-alert emails, not scraping.
- Prefer official public ATS APIs (Greenhouse, Lever, Ashby, SmartRecruiters).
- Never fabricate information in applications, contacts or emails.
- Keep cold-email volume low and personal; respect replies asking not to be contacted.

---

## 15. How we measure success

| Area | Metric | Baseline | Target method |
|---|---|---|---|
| Claude tokens | Cache-read + input + output per daily cycle | 183.4 M cache-read over the SHOOT session | Measure with `_pipeline/session_usage.js` after Phase 1 |
| Collection | Claude tokens used by collection | Most of the session | 0 (runs outside Claude) |
| Dedup | Duplicate rows after two runs | Not measured | 0 |
| Resumes | Tailored resumes per day | 46 | Only for shortlisted jobs |
| Applications | Manual fields typed per application | All fields | Only flagged missing fields |
| Tracking | Jobs with a known status | Partial (CSV) | 100% in database |

No percentage reduction is promised until before/after numbers are measured.

---

## 16. Expected results (realistic)

- **Manual job searching:** scheduled runs replace the SHOOT conversation; you read a digest.
- **Repeated data entry:** profile entered once; known fields filled automatically; unknown fields flagged.
- **Resume duplication:** tailoring only for shortlisted jobs with hash-based reuse.
- **Browser operations:** one adapter per ATS reused across all companies on that platform.
- **Claude tokens:** collection, dedup, scoring, PDF and dashboard cost 0; Claude sees a compact shortlist in short sessions.
- **Time per application:** reduced to reviewing a pre-filled form and answering flagged questions.

**Known limits:** Workday needs per-company accounts; some sites use CAPTCHAs; LinkedIn and Naukri are not automated; some forms will always need manual answers.

---

## 17. Open decisions

| # | Decision | Recommended default |
|---|---|---|
| 1 | Repository home | ✅ **Decided:** `D:\Interview\Job_Search` — all work happens there; no separate project folder |
| 2 | Scheduler | n8n run natively via `npm` (saves ~2 GB RAM vs Docker) so it can run Node scripts — recent n8n disables the "Execute Command" node by default, so it needs enabling. Alternative: Windows Task Scheduler for collection + n8n in Docker for email |
| 3 | Sender email | Job forms currently use a dedicated job-search Gmail address (kept private). Confirm whether that is also the cold-email sender or a new address will be created. Never the Claude account email |
| 4 | Database password | You put it in `.env` or `pgpass` yourself; Claude never needs to see it |
| 5 | Profile service stack (Phase 2) | Decide at Phase 2: Spring Boot (strong interview value for a Java backend role) or Node/Express (same language as the pipeline) |

---

## 18. How we work together (implementation method)

For every task:

1. Explain what we are building.
2. Explain why it is needed.
3. Explain how it works.
4. List the exact files that will change.
5. Give the exact terminal commands.
6. Implement.
7. Test.
8. Show the result.
9. Explain any errors and their fixes.
10. Update documentation.
11. Move on only after the task is validated.

One task at a time. No batch of 20 tasks at once.

---

## 19. Interview explanation guide

Short answers you can give about this project:

**What is it?**
A job-application automation system: scheduled collectors gather jobs from public ATS APIs and feeds, PostgreSQL stores and deduplicates them, a rule-based scorer ranks them, and an LLM is used only for judgement tasks. Playwright fills application forms from a structured applicant profile, and n8n handles email and status tracking with human approval gates.

**Key design decisions**
- *Deterministic first, LLM last:* reduced LLM cost and made results reproducible.
- *Dedup before AI:* a `canonical_key` unique constraint means each job is processed once.
- *Database as source of truth:* PDFs and dashboards are generated outputs, never inputs.
- *Adapter pattern for ATS platforms:* one adapter per platform, reused across companies.
- *Field Mapping Engine:* learned label → profile-field mappings stored in a table.
- *Human-in-the-loop:* the system prepares; the person submits and approves.
- *Additive migrations:* schema evolves without breaking existing data.

**Problem you solved**
"Discovery used most of my AI quota. I measured it from session logs: 392 turns re-reading up to 965k tokens of context each time — 183 million cached tokens. The scraping was already scripted; the cost came from running orchestration inside one long LLM conversation. I moved collection to a scheduler and database, so the LLM only sees a compact shortlist."

**Technologies**
Node.js · PostgreSQL · n8n · Playwright · LaTeX · Gmail API / SMTP · Git · Claude Code · Gemini API.

---

*Next step: Phase 1, Task 1.5 — compliance switch: stop LinkedIn guest-page and Naukri browser scraping, use job-alert emails and official ATS feeds, then run the live collectors through `collect.js`.*

## Progress log

| Date | Task | Result |
|---|---|---|
| 2026-11-25 | 1.1 | PostgreSQL started; live schema documented (`jobhunter`: jobs 101, applications 2, recruiters 0); git initialised (commit `1b08a62`); token baseline saved |
| 2026-11-26 | 1.2 | `V001__jobshot_schema.sql` applied in one transaction: 6 tables + `schema_migrations`, `updated_at` triggers, automatic `status_history` trigger. Tested: dedup rejects duplicate `canonical_key`, invalid status rejected, status change logged with actor/note. `public` tables unchanged |
| 2026-11-26 | 1.3 | V002 adds 8 scoring columns (deadline, job_family, eligibility, eligibility_note, flags, match_reasons, score_breakdown, applicants_text). New `db.js` (pool + transactions, password from pgpass), `jobmap.js` (pure mapping: company/city normalization, experience parsing, ATS detection, fingerprint key `fp:sha1(company\|title\|city)`), `collect.js` (runs existing collectors unchanged, loads `scored_DAY.json`, one-line JSON output, logs to `data/logs/`). Driver `pg@8.13.1`. Tested in load mode: 24 Nov → 138 new; 25 Nov → 86 new + 66 seen again; reload → 0 new. 224 jobs, 188 companies, 1 cross-source duplicate merged (Qualcomm: LinkedIn + Naukri). Live collector steps not run yet (Task 1.5 compliance first) |
| 2026-11-26 | 1.4 | **Dropped.** Owner decision: no old data in the product. `jobshot` data tables were cleared (TRUNCATE, logged as `data.reset` in `audit_log`) and reloaded with the 25 Nov 2026 run only: 152 jobs (33 Apply First, 31 Good Match, 88 Stretch), 133 companies. Schema, migrations and `public` tables untouched |
| 2026-11-26 | 1.5 (in progress) | Checked robots.txt of every collected site. New `robots.js` gate on every collector request: robots.txt rules (longest match, `*`/`$`), crawl delay + 1 s minimum per host, honest User-Agent `JobShot/1.0`; documented job APIs (Lever, SmartRecruiters, Greenhouse, Ashby) use API terms. Results: LinkedIn disallows all → `harvest.js linkedin` disabled, LinkedIn links marked unconfirmed in `aggregators.js`; Freshershunt and OffCampusJobs4u disallow `/feed/` → skipped automatically; Internshala job paths, Jobs Adda, Workday allowed. `CLAUDE.md` now forbids scraping LinkedIn/Naukri/Indeed. 12 robots rule tests pass. **Pending:** check results of the first live `node collect.js` run (26 Nov), job-alert email parsing (Naukri/LinkedIn), Greenhouse/Ashby boards |

# Job Auto-Apply System — Plan

Goal: stop applying by hand. The system should **find jobs that match my CV, fill the
application forms, upload the CV and submit** — and when a job says "send your CV to
an email address", **write and send the email automatically**.

---

## 1. How it works (big picture)

```
 ┌────────────┐   ┌────────────┐   ┌─────────────┐   ┌──────────────┐   ┌───────────┐
 │ 1. Collect │──▶│ 2. Match   │──▶│ 3. Prepare  │──▶│ 4. Apply     │──▶│ 5. Track  │
 │   jobs     │   │  with CV   │   │ cover letter│   │ form / email │   │ & report  │
 └────────────┘   └────────────┘   └─────────────┘   └──────────────┘   └───────────┘
   scrapers,        AI score         AI writes a       Playwright fills   database +
   APIs, RSS,       0–100, keep      short letter      the form + uploads daily summary
   email alerts     only ≥ 70        per job           CV, or Gmail sends  email
```

Runs on a schedule (e.g. every morning at 8:00) — GitHub Actions cron, or a small VPS / PC.

---

## 2. Components

### 2.1 Profile (one-time setup)
`profile.yaml` — everything a form asks for, written once:

```yaml
name: Awantha Imesh
email: awanthaimesh65@gmail.com
phone: "+94 7X XXX XXXX"
location: Sri Lanka
linkedin: https://linkedin.com/in/awantha
github: https://github.com/awantha2003
cv_file: cv/Awantha_Imesh_CV.pdf
roles: [Frontend Developer, React Developer, Junior Web Developer, Intern Software Engineer]
skills: [JavaScript, React, HTML, CSS, PHP, C, C++]
experience_years: 0-1
expected_salary: "negotiable"
notice_period: Immediate
work_type: [remote, hybrid, onsite-colombo]
visa_sponsorship_needed: false
blacklist_companies: []
```

The CV PDF is also converted to text once (`pdfplumber`) so the AI can read it.

### 2.2 Job collectors (`collectors/`)
Prefer sources that allow automated access first:

| Source | Method | Notes |
|---|---|---|
| Remotive, RemoteOK, Arbeitnow | Public JSON APIs | Easy, legal, remote jobs |
| Adzuna, Jooble | Free API key | Many countries |
| Greenhouse / Lever / Workable / Ashby job boards | Public job-board JSON | Company career pages; forms are simple to auto-fill |
| topjobs.lk, xpress.jobs, ikman jobs | Scrape listing pages (Playwright) | Many SL jobs say "send CV to hr@…" → email path |
| LinkedIn / Indeed | **Job-alert emails** read from Gmail | Do NOT bot-apply on LinkedIn/Indeed (see §5) |

Every job is normalised to one shape:
`{id, title, company, location, description, url, apply_type: form|email|external, apply_email?, posted_at, source}`
and de-duplicated by `hash(company + title)`.

### 2.3 Matcher (`matcher.py`)
1. **Hard filters** (fast, free): keywords in title, location/remote, experience level
   ("Senior", "5+ years" → skip), blacklist.
2. **AI score** (Claude API): send CV text + job description, get back JSON:
   `{score: 0-100, reasons: [...], missing_skills: [...], apply: true/false}`.
3. Keep jobs with `score >= 70` (configurable).

### 2.4 Content generator (`writer.py`)
For each matched job, the AI writes:
- a **short cover letter / email body** (120–180 words, mentions 2–3 real skills from the CV
  that match the job — never invents experience),
- answers for **custom form questions** ("Why do you want to join?", "Years of React?")
  using `profile.yaml` + CV only. Unknown answer → mark job as `needs_review`.

### 2.5 Applier
**a) Form applier (`apply_form.py`) — Playwright**
- One "adapter" per site type: `greenhouse.py`, `lever.py`, `workable.py`, `generic.py`.
- Fills name/email/phone/links, uploads `cv_file` with `set_input_files()`, pastes cover letter,
  answers questions, clicks submit, saves a **screenshot** as proof.
- CAPTCHA / login wall / unknown required field → stop, mark `needs_review` (no CAPTCHA bypassing).

**b) Email applier (`apply_email.py`) — Gmail API**
- Subject: `Application for <Job Title> – Awantha Imesh`
- Body: AI-written cover letter. Attachment: CV PDF.
- Sent through Gmail API (OAuth) or SMTP with a Gmail **App Password**
  (never store the real password; keep secrets in `.env` / GitHub Secrets).
- Never email the same address twice for the same job.

### 2.6 Tracker (`jobs.db` — SQLite)
Table `applications`: `job_id, title, company, source, url, score, status
(found|matched|applied|emailed|needs_review|skipped|failed), applied_at, screenshot, notes`.

Daily **summary email to myself**: "Applied 12, emailed 5, needs review 3 (links)".
Optional: export to Google Sheets so I can see everything on my phone.

---

## 3. Safety switches (important)
- **Review mode first** (`AUTO_SUBMIT=false`): system fills everything and sends me the list;
  I approve with one click/reply. Switch to full auto only after ~1 week of good results.
- **Daily cap**: max 20–30 applications/day, random delays between actions.
- **Dry run** flag for testing (`--dry-run` fills but never submits/sends).
- Never apply twice to the same company+role.

---

## 4. Tech stack
- **Python 3.11**, `playwright`, `requests`, `pdfplumber`, `pyyaml`, `sqlite3`
- **LLM for matching + writing** — behind one small `llm.py` wrapper so the provider can be
  swapped with an env var (`LLM_PROVIDER=gemini|groq|openrouter|ollama|claude`):
  - **Gemini** (Google AI Studio free tier) — recommended default for a free setup
  - **Groq** free tier (open models like Llama) — very fast, good backup when Gemini hits limits
  - **OpenRouter** `:free` models — many open models with one API key
  - **Ollama** (local: Llama, Qwen, Gemma, Mistral) — fully free/offline, needs a decent PC
  - **Claude** (paid) — best quality writing, if budget allows
  - Note: free tiers have daily rate limits, and some (e.g. Gemini free) may use prompts for
    training — CV data is personal, so check each provider's terms.
- **Gmail API** (`google-api-python-client`) or `smtplib`
- Scheduler: **GitHub Actions cron** (free) or `cron` on a Linux box
- Alternative no-code option: **n8n** workflow (Schedule → HTTP/RSS → AI → Gmail) for the
  email path only

### Folder layout
```
job-bot/
  profile.yaml
  cv/Awantha_Imesh_CV.pdf
  .env                      # ANTHROPIC_API_KEY, GMAIL creds (never commit)
  collectors/  remotive.py  adzuna.py  greenhouse.py  topjobs.py  gmail_alerts.py
  matcher.py
  writer.py
  appliers/    greenhouse.py  lever.py  workable.py  generic.py  email.py
  tracker.py
  main.py                   # collect → match → write → apply → report
  .github/workflows/daily.yml
```

---

## 5. Limits to know
- **LinkedIn, Indeed, Glassdoor ban accounts that use bots to apply.** Use their job-alert
  emails as a *source*, then apply on the company's own site or by email.
- CAPTCHAs and logins (Workday, Taleo) can't be reliably auto-filled → go to `needs_review`.
- Mass, low-quality applications hurt reputation — the score threshold + tailored letter matter
  more than volume.

---

## 6. Build order (milestones)
1. **Week 1** — `profile.yaml`, CV → text, Remotive + topjobs collectors, SQLite tracker.
2. **Week 1** — Claude matcher + cover-letter writer; print matches (no applying).
3. **Week 2** — Email applier (Gmail) in review mode → most SL jobs covered.
4. **Week 2–3** — Playwright form applier: Greenhouse + Lever + Workable adapters.
5. **Week 3** — GitHub Actions daily schedule + summary email.
6. **Week 4** — Turn on auto-submit with daily cap; add more sources.

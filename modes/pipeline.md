# Mode: pipeline — URL Inbox (Second Brain)

Process job URLs stored in `data/pipeline.md`. The user adds URLs at any time and then executes `/career-ops pipeline` to process them all.

## Workflow

1. **Read** `data/pipeline.md` → search for `- [ ]` items in the "Pending" section
1b. **Run Ashby-aware company guardrails before materialization**:
   - Normalize company names and inspect `job_applications.csv`.
   - Treat `Applied`, `Ready to Apply`, and `Needs Referral` as active application states.
   - Allow only 1 active application per company in a 14-day window.
   - If multiple pending jobs are from the same company, select only the strongest role and mark lower-ranked roles deferred.
   - For Ashby-hosted jobs, gate high-priority roles as `Needs Referral` instead of `Ready to Apply`.
2. **For each pending URL**:
   a. Calculate the next sequential `REPORT_NUM` (read `reports/`, take the highest number + 1)
   b. **Extract JD** using Playwright (browser_navigate + browser_snapshot) → WebFetch → WebSearch
   c. If the URL is not accessible → mark as `- [!]` with a note and continue
   d. **Execute full auto-pipeline**: Evaluation A-F → Report .md → PDF (if score >= 3.0) → Tracker
   e. **Move from "Pending" to "Processed"**: `- [x] #NNN | URL | Company | Role | Score/5 | PDF ✅/❌`
3. **If there are 3+ pending URLs**, launch agents in parallel (Agent tool with `run_in_background`) to maximize speed.
   - Do not give parallel workers write access to `job_applications.csv`, `data/pipeline.md`, or the active browser session.
   - Antigravity/Gemini bulk evaluator runs should receive only company-governor selected jobs unless the user explicitly asks to audit deferred jobs.
4. **At the end**, show summary table:

```
| # | Company | Role | Score | PDF | Recommended action |
```

## Format of pipeline.md

```markdown
## Pending
- [ ] https://jobs.example.com/posting/123
- [ ] https://boards.greenhouse.io/company/jobs/456 | Company Inc | Senior PM
- [!] https://private.url/job — Error: login required

## Processed
- [x] #143 | https://jobs.example.com/posting/789 | Acme Corp | AI PM | 4.2/5 | PDF ✅
- [x] #144 | https://boards.greenhouse.io/xyz/jobs/012 | BigCo | SA | 2.1/5 | PDF ❌
```

## Intelligent JD detection from URL

1. **Playwright (preferred):** `browser_navigate` + `browser_snapshot`. Works with all SPAs.
2. **WebFetch (fallback):** For static pages or when Playwright is unavailable.
3. **WebSearch (last resort):** Search in secondary portals that index the JD.

**Special cases:**
- **LinkedIn**: May require login → mark `[!]` and ask the user to paste the text
- **PDF**: If the URL points to a PDF, read it directly with the Read tool
- **`local:` prefix**: Read the local file. Example: `local:jds/linkedin-pm-ai.md` → read `jds/linkedin-pm-ai.md`

## Automatic numbering

1. List all files in `reports/`
2. Extract the number from the prefix (e.g., `142-medispend...` → 142)
3. New number = maximum found + 1

## Source synchronization

Before processing any URL, verify sync:
```bash
node cv-sync-check.mjs
```
If there is a desynchronization, warn the user before continuing.

## Ashby Readiness Rules

- Detect Ashby by `jobs.ashbyhq.com` URLs or Ashby API metadata.
- Keep generated PDFs text-extractable, preferably under 1MB, and never treat PDFs over 16MB or non-extractable PDFs as ready.
- Add exact-keyword and evidence-backed criteria notes to reports for Ashby roles.
- Treat non-DEI optional fields as required during manual form review.
- Legal, salary, authorization, privacy, demographics, and final submit decisions remain human-only.

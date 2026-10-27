# job-hunt-agent

A local foundation for organizing a job search, scoring saved job descriptions against factual profile data, and tracking application progress. It uses Python 3.12, Pydantic, SQLite, and Playwright. Browser control is limited to a persistent visible session and the harmless `https://example.com` test; no job-site automation is implemented.

## What it does and does not do

It validates a profile and approved answers, stores jobs locally, detects duplicate IDs, canonical URLs, and conservative company/title/location matches, provides explainable deterministic fit scores, tracks statuses, and checks proposed actions against explicit safety rules. The browser service can launch a persistent browser profile, inspect a page, and take a screenshot. It does not scrape job boards, fill or submit applications, or automate any job board or employer site.

It never stores passwords. It does not infer factual profile fields from a resume or job posting. Scoring is a first-pass heuristic; missing structured posting information is not proof that a candidate lacks that qualification.

## Windows installation

Requires Python 3.12.

```powershell
cd C:\Users\Acer\Desktop\JobHunt
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -e ".[dev]"
Copy-Item .env.example .env
```

Edit `.env` as needed. Configuration is validated through Pydantic Settings. Only a local SQLite URL is accepted. The default database is `data/job_hunt.db`.

To save a factual profile, create `data/profile.json` using the fields in `app/models/domain.py`, then run:

```powershell
python -m app.cli profile validate
python -m app.cli profile show
```

Jobs can be added and viewed with:

```powershell
python -m app.cli jobs add-linkedin --url "https://www.linkedin.com/jobs/view/123456789/"
python -m app.cli jobs list
python -m app.cli jobs score
python -m app.cli tracker summary
python -m app.cli safety check "fill application fields" --question "What is your work authorization?"
```

Supported external ID sources are `linkedin`, `indeed`, `bayt`, `naukri`, `naukri gulf`, `employer`, and `other` (mapped to `li-`, `in-`, `bayt-`, `nk-`, `ng-`, `em-`, and `x-`). For composite conventions such as `ng-<company-role>`, supply that composite as the identifier.

## Job discovery pipeline

`jobs add` and `jobs add-linkedin` are user-driven intake commands. They validate, normalize, deduplicate, score from the profile saved in SQLite, apply safety filters, show a preview, then wait for an explicit `Add to Tracker? [y/N]` confirmation. LinkedIn URLs are never visited. A URL-only job may be saved and is marked `Needs User` until its missing details are reviewed.

```powershell
python -m app.cli jobs add --source manual --external-id demo-001 --title "Junior Data Scientist" --company "Example Analytics LLC" --location "Dubai, UAE" --url "https://example.com/jobs/1" --description "Python, SQL, Machine Learning. Visa sponsorship available." --employment-type "Full-time" --required-skills "Python, SQL, Machine Learning"
python -m app.cli jobs add --source indeed --url "https://www.indeed.com/viewjob?jk=example"
python -m app.cli jobs paste
python -m app.cli jobs list
python -m app.cli jobs duplicates
```

Manual URL intake is supported for LinkedIn, Indeed, Bayt, NaukriGulf, and company career pages. `jobs paste` reads pasted text until an `END` line and only parses labeled fields or a clear title/company/location header. The optional local preview page runs on loopback: `python -m app.cli jobs intake-ui`, then open `http://127.0.0.1:8765/jobs/intake`. It analyzes only the information entered into the form; Add and Cancel are explicit actions.

Use `python -m app.cli jobs queries` to print profile-driven search suggestions for each source, and `python -m app.cli jobs workflow` for a suggested source order and tracking counts. These commands display suggestions only; they do not search job sites.

Duplicate records update only missing job information, append notes, and record the relationship without changing application progress. Original and normalized title, company, location, and URL values are retained separately. Exact source IDs and canonical URLs are checked before conservative company/title/location comparison.

Import a JSON array of manual records by running `python -m app.cli jobs pipeline --file data/manual_jobs.json`. The records use the `RawJob` field names in `app/jobs/models.py`. `jobs pipeline` without a file runs an empty manual batch and reports zero fetched records.

`python -m app.cli jobs search --query "Data Scientist"` searches saved local records only. Without `--query`, it prints suggestions derived from the stored profile; `jobs queries` groups the same suggestions by manual source. A request such as `python -m app.cli jobs search --source linkedin` returns `NOT_IMPLEMENTED`. No web search, scraping, or site access occurs.

### LinkedIn source status

LinkedIn automated discovery is disabled. Use manual intake. `jobs search --source linkedin` returns `NOT_IMPLEMENTED` without making a request. The LinkedIn source accepts only the URL and information you supply; there is no Playwright, Selenium, DOM, API, bookmarklet, or network access in its adapter. No job application is submitted.

Use `python -m app.cli jobs show <job_id>` to inspect a saved record and its current fit explanation without fetching it again.

Fit scoring uses only profile-stored roles, locations, skills, experience, education, work authorization, sponsorship preference, and exclusions. Add verified profile facts with `profile validate`; missing profile or eligibility information sends a job to `Needs User`. Unclear sponsorship, work authorization, or clearance language is not treated as approval. Closed posts, fees, citizenship restrictions, explicit exclusions, invalid URLs, and large experience gaps are skipped with a reason.

## Tests

```powershell
python -m pytest
```

## Browser Automation Setup

Use Python 3.12 for this step. In PowerShell, from the project folder:

```powershell
py -3.12 --version
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -m playwright install chromium
Copy-Item .env.example .env
```

Set `BROWSER_PROFILE_PATH=data/browser_profile`, `BROWSER_HEADLESS=false`, and optionally `BROWSER_CHANNEL=chromium` in `.env`. The default browser is visible. The profile directory is created on first launch and reused on later runs, so the browser can retain the user's manually established sessions. The application never reads or prints passwords, cookies, authentication tokens, or browser storage.

Run these commands in PowerShell:

```powershell
python -m app.cli browser check
python -m app.cli browser open-test
python -m app.cli browser screenshot
```

Both test commands navigate only to `https://example.com`; they print the title and main heading. The screenshot command saves `logs/browser/example-com.png`. Browser lifecycle and safety events are written to the existing SQLite `action_logs` table and the application log. Logged URLs are reduced to their origin so query strings and path tokens are not recorded.

The persistent browser profile contains sensitive session information, including cookies and local browser storage. Keep `data/browser_profile/` private. It is excluded by `.gitignore`; never commit, copy, or share it. The `.env` file and logs are also excluded.

This step does not automate LinkedIn, Indeed, Bayt, NaukriGulf, Naukri, Workday, Greenhouse, Lever, Ashby, or company career sites. It does not create accounts, upload CVs, submit applications, solve CAPTCHAs, or enter OTP codes. CAPTCHA, OTP, login walls, payment prompts, and suspicious security challenges produce `NEEDS_USER` and stop automated interaction. Complete such actions manually and rerun only when appropriate.

## Safety behavior

The safety checker returns an allowed decision, explanation, rule ID, whether a user is required, and severity. CAPTCHA, OTP, demographic/citizenship prompts, unknown legal/consent questions, payment, account creation, missing CV, and ambiguous questions stop with a user-required decision. Submission actions always require human approval. Work authorization and sponsorship are only allowed when the caller indicates an approved verified answer exists. These checks are not a substitute for human review, and must be integrated as a mandatory gate before any future browser action.

The profile stores only facts the user supplied. Application answers include `approved` and `last_verified`; the preparation question mapper uses explicit matches and requires human review. There is no application browser agent.

## Application preparation (Step 6)

Preparation is local and does not submit an application, contact an employer, upload a file, or use an external AI service. Put your real PDF or DOCX resume in the project-root `resumes/` directory. The repository contains only `resumes/.gitkeep`; it does not create or assume a resume. Resume files under `resumes/` are ignored by Git. Register an existing CV by its path and describe its focus; registration checks the file and extension but never reads or copies its contents:

```powershell
python -m app.cli cv add --path "resumes/data_scientist_uae.pdf" --role "Data Scientist" --skills "Python, SQL, Statistics"
python -m app.cli cv approve <cv_id>
python -m app.cli cv list
python -m app.cli cv show <cv_id>
```

Absolute local paths are also supported. Paths inside the project are stored relative to its root. An outside-project path triggers a warning and confirmation and is stored as an absolute local path; the file is not copied. Only `.pdf` and `.docx` files are accepted. `cv add --approved` is available when you want to approve the CV during registration. CV recommendations use only your role/skill emphasis metadata and require an approved, existing file. Missing or deleted CV files are excluded.

Generate a plan for a job already in the tracker, then paste application questions one per line and finish with `END`:

```powershell
python -m app.cli application plan <job_id>
python -m app.cli application questions <job_id>
python -m app.cli application draft-cover-letter <job_id>
```

Cover letters use only the job title/company, your saved name, and profile skills that also appear in the supplied job description. The default command prints an editable draft without saving it. Add `--save` to store it locally in the per-job preparation record. Question mappings show their evidence and confidence. Approved saved answers older than one year or missing a verification date are marked `NEEDS_CONFIRMATION`; work authorization, sponsorship, citizenship, demographic, disability, veteran, and criminal-history questions always require application-specific review. Unknown questions remain unanswered. A confirmed or edited response is stored only for that job's preparation; it does not change the global answer bank.

The existing local UI also serves application preparation:

```powershell
python -m app.cli jobs intake-ui
```

Open `http://127.0.0.1:8765/applications/prepare`, choose a tracked job, review its score and concerns, select an approved CV, paste questions, edit the letter, and mark checklist items. **Export plan locally** explicitly writes `data/application-plans/job-<id>.json`; an existing export is not overwritten. All checklist items must be marked by you before the preparation state becomes `Ready for Manual Application`. This is not the job's `Applied` status.

After you personally submit an application on the employer's site, record that report explicitly with its date:

```powershell
python -m app.cli application report-applied <job_id> --date 2026-10-27
```

The command updates the tracker and appends a note that the application was reported by you; it does not independently verify submission. The plan always states that preparation alone has not submitted anything. No browser form filling, email, CV upload, CAPTCHA/OTP handling, or protected-site interaction is part of this workflow. Current limitations: CV files are not parsed, recommendations rely on user-entered CV metadata, requirement extraction is conservative, and there is no automatic eligibility-answering or application-submission feature.

## Job source registry and manual intake

`python -m app.cli jobs sources` displays the configured starting points and their manual-only status. These are service or search starting pages, not guaranteed direct vacancy URLs or confirmed APIs:

| Source | Starting point |
| --- | --- |
| LinkedIn Jobs | [linkedin.com/jobs](https://www.linkedin.com/jobs/) |
| Indeed UAE | [ae.indeed.com](https://ae.indeed.com/) |
| Bayt UAE | [bayt.com/en/uae/jobs](https://www.bayt.com/en/uae/jobs/) |
| NaukriGulf | [naukrigulf.com](https://www.naukrigulf.com/) |
| Greenhouse candidate portal | [my.greenhouse.io](https://my.greenhouse.io/) |
| Lever | [lever.co](https://www.lever.co/) |
| Workday | [workday.com](https://www.workday.com/) |

For LinkedIn, open the site yourself, find a vacancy, copy its URL and description, then paste them through `python -m app.cli jobs add-linkedin` or the local page at `http://127.0.0.1:8765/jobs/intake`. Use `python -m app.cli jobs add --source greenhouse` (or `indeed`, `bayt`, `naukrigulf`, `lever`, `workday`, `company_career`, or `manual`) for other manually collected vacancies. `python -m app.cli jobs paste --source linkedin` accepts copied text and asks for missing fields. Each path previews duplicates, fit, missing information, safety concerns, and proposed tracker status before asking for confirmation. A missing description is called out because it lowers confidence in scoring.

No source adapter currently searches or scrapes a job site, and the registry does not assert that an API exists or that automation is permitted. In particular, LinkedIn is user-operated only; the app does not inspect LinkedIn DOM/APIs or use browser automation there. See LinkedIn's [Automated activity guidance](https://www.linkedin.com/help/linkedin/answer/a1341543), [Prohibition of scraping software](https://www.linkedin.com/help/linkedin/answer/a1341387/prohibition-of-scraping-software), and [User Agreement](https://www.linkedin.com/legal/user-agreement). Indeed, Bayt, NaukriGulf, Greenhouse, Lever, Workday, and company career pages also remain manual unless a documented authorized integration is added later.

## Planned browser architecture

Future browser work should be a separate Playwright adapter behind `app/browser`, with an explicit state machine and safety gate before each interaction. It should use a user-visible local Chrome profile, never persist credentials, pause for CAPTCHA/OTP/security checks and unknown or sensitive prompts, and require the user to confirm final submission. Site-specific adapters must be developed only after this foundation is validated; this project currently makes no requests to job sites.

## Database backup

Close any process using the database, then copy `data/job_hunt.db` to a protected backup location. SQLite's `.backup` command or Python `sqlite3.Connection.backup()` can also create a consistent copy while the application is open. Back up profile data and the database together. The database contains personal information, so protect backups with appropriate local disk permissions and encryption. Do not commit `.env`, profile data, or database files.

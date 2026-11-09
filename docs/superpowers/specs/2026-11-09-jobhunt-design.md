# Jobhunt: design

Date: 2026-11-09. Status: draft, waiting for review.

## Purpose

Jobhunt is a self-hosted job-search app. Each person installs their own copy. One copy serves one user. It finds job postings, tailors the user's resume for each job, and builds a study plan for each job that reaches the interview stage.

Success for v1:

1. The app fetches jobs from the v1 sources on a schedule and shows only jobs that match the user's preferences.
2. For a saved job, the app shows a fit analysis and produces a tailored resume as HTML and PDF.
3. A tailored resume contains no claim that is absent from the base profile.
4. A job moved to `interview` gets a study plan that ends on the interview date.

## Stack

- Go, one binary, cross-compiled for macOS, Linux and Windows.
- SQLite through `modernc.org/sqlite` (pure Go, no cgo).
- Server-rendered `html/template` pages with htmx. All assets are embedded with `embed`.
- Claude API through the official Go SDK. Each user enters their own API key. Default model: `claude-sonnet-5-5`, configurable.
- PDF export through `chromedp` when Chrome or Chromium is installed.

## Architecture

The binary runs a web server on `localhost:8080` (configurable) and keeps all data in one folder:

```
~/.jobhunt/                 data folder, override with --data or JOBHUNT_DATA
  jobhunt.db                SQLite database
  config.yaml               preferences, sources, settings
  profile.yaml              base resume, the source of truth
  template/resume.html      resume template, copied from the embedded default on first run
  output/                   generated resumes
```

| Package | Job | Depends on |
|---|---|---|
| `source` | `Adapter` interface and one adapter per job source. | HTTP |
| `store` | All SQLite access. Migrations run at startup. | SQLite |
| `ai` | All Claude calls and all prompts. | Claude API |
| `resume` | PDF import, profile model, source-map check, HTML render, PDF export. | `ai` |
| `scheduler` | Runs adapters on a timer, removes duplicates, applies preference filters. | `source`, `store` |
| `web` | HTTP handlers, pages, htmx fragments. | all packages |

Only `ai` calls Claude. Only `store` touches SQLite.

### Job status

`new → saved → applied → interview → offer`. Any state can move to `rejected` or `archived`. The scheduler sets `filtered` on jobs that fail the preference filter. The user can move a `filtered` job to `new` or `saved`. A move to `interview` requires an interview date and starts the study plan generation.

### Access

On `localhost` the app has no login. When `config.yaml` sets `listen` to a non-loopback address, the app refuses to start unless `auth.password_hash` is set. It then requires that password through a session cookie.

## Configuration

`config.yaml` example:

```yaml
listen: 127.0.0.1:8080
claude:
  api_key_env: ANTHROPIC_API_KEY
  model: claude-sonnet-5-5
preferences:
  roles: ["backend engineer", "php developer", "go developer"]
  seniority: ["senior", "staff"]
  remote: required          # required | preferred | any
  locations: ["Brazil", "Latin America", "Worldwide"]
  exclude_keywords: ["clearance", "on-site only"]
  min_salary_usd: 0         # 0 disables the filter; applies only when the posting states a salary
sources:
  fetch_interval: 6h
  greenhouse: ["stripe", "gitlab"]
  lever: ["netflix"]
  ashby: ["linear"]
  remotive: true
  remoteok: true
  weworkremotely: true
```

The settings page edits this file. The file stays the source of truth so a user can also edit it by hand.

## Job discovery

### Adapter interface

```go
type Job struct {
    Source      string
    ExternalID  string
    URL         string
    Company     string
    Title       string
    Location    string
    Remote      bool
    SalaryText  string
    Description string // plain text
    PostedAt    time.Time
}

type Adapter interface {
    Name() string
    Fetch(ctx context.Context) ([]Job, error)
}
```

### v1 adapters

| Adapter | Endpoint | Input from config |
|---|---|---|
| Greenhouse | `boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true` | board tokens |
| Lever | `api.lever.co/v0/postings/{company}?mode=json` | company slugs |
| Ashby | `api.ashbyhq.com/posting-api/job-board/{name}` | board names |
| Remotive | `remotive.com/api/remote-jobs` | on/off |
| RemoteOK | `remoteok.com/api` | on/off |
| We Work Remotely | RSS feed | on/off |
| Manual | paste a URL or the posting text | none |

The manual adapter covers LinkedIn, Gupy and every source with no API. For a URL, the app fetches the page and asks Claude to extract the `Job` fields. For pasted text, Claude extracts the fields from the text. The app does not scrape LinkedIn.

Confirm each endpoint and its response shape during planning. Record one real response per adapter as a test fixture.

### Pipeline

1. The scheduler runs every adapter at `fetch_interval`, and on demand from the sources page.
2. Each adapter runs in its own goroutine with a 30-second timeout.
3. Duplicates are removed by `(source, external_id)`, then by a hash of the normalized company, title and location. The second key catches one job posted on two boards.
4. The keyword filter applies `preferences`. It uses no AI, so it costs nothing.
5. Jobs that pass get status `new`. Jobs that fail are stored as `filtered`, hidden by default, and visible through a filter toggle. The user can then see why a job was excluded.

## Job profile

The fit analysis runs when the user moves a job to `saved`. It does not run on `new` jobs, to control API cost.

The analysis contains:

1. Fit score from 0 to 100. Each point of evidence is a pair: a requirement from the posting and the profile item that matches it.
2. Gap list. Each gap is marked `required` or `nice_to_have`.
3. Resume highlights: which profile items to put first, and which posting keywords match skills the user has.
4. Company research: product, size, tech stack, hiring process. This step uses the Claude web search tool. Each fact shows its source URL.

Claude returns structured JSON that matches a schema in the `ai` package. The app stores the analysis in SQLite.

A cover letter is generated only when the user clicks its button. The app shows a salary only when the posting states one.

## Resume

### Base profile

`profile.yaml` sections: `contact`, `summary`, `experience` (company, title, dates, bullets), `skills`, `education`, `projects`, `languages`. Each bullet and each skill has a stable ID, for example `exp.corephp.b3`.

Import: the user uploads their current PDF. Claude reads the PDF and returns the YAML. The app shows the result in an edit form. The user corrects it and saves it one time. After that, `profile.yaml` is the only source.

### Tailoring

1. The user clicks "Tailor resume" on a saved job.
2. Claude receives the base profile, the posting and the fit analysis. It returns a tailored profile in the same schema. Each output bullet carries `source_ids`, the base IDs it came from.
3. The source-map check runs:
   - Each tailored bullet must have at least one valid `source_id`.
   - Each skill in the tailored profile must exist in the base `skills`.
   - Company names, job titles and dates must equal the base values exactly.
4. The review page shows each tailored line next to its source lines. Any line that fails the check is marked red. The user can edit or remove any line before export.
5. The app does not export a resume that has a failed check.

### Output

The template renders the tailored profile to `output/profile-<company>-<job-title>-<YYYY-MM-DD>.html`. Company and title are lowercased, with spaces as hyphens and other characters removed. A second version on the same day gets a `-2` suffix.

PDF export uses `chromedp` when it finds Chrome or Chromium. Otherwise the app shows the HTML with a print stylesheet and tells the user to use "Print to PDF".

## Interview study

A move to `interview` requires the interview date and an optional stage name (for example "technical screen"). Generation starts at once. The study plan contains:

1. Likely interview format and stages for this company, from web search, with source URLs.
2. Technical topics to review, built from the gap list and the job's tech stack. Each topic has a priority and suggested resources.
3. Behavioral questions. Each question maps to a real profile item, with a STAR outline (situation, task, action, result) built from that item.
4. Questions for the user to ask the interviewer.
5. A day-by-day plan from today to the interview date. If the date is less than two days away, the plan is a single priority list.

Each topic and each day has a checkbox. The app stores progress. A job can have more than one interview stage. Each stage gets its own study plan.

## Error handling

- Adapter failure: log the error, store the last error and time per source, and show them on the sources page. Other adapters continue.
- Claude call failure: store the task as `failed` with the error text, and show a retry button. No automatic retry beyond the SDK's own retry on rate limits.
- Invalid JSON from Claude: retry the call one time, then mark the task `failed`.
- Missing API key: AI features show a message that links to settings. Discovery and the board still work.
- Long tasks (analysis, tailoring, study plan) run in background goroutines. The page polls the task status through htmx.

## Testing

- `source`: each adapter is tested with `httptest` against a recorded real response.
- `store`: tests run against a temporary SQLite file with real migrations.
- `resume`: the source-map check has table tests for valid output, an invented bullet, an invented skill and a changed date. Rendering has a golden-file test.
- `scheduler`: duplicate removal and preference filters have table tests.
- `ai`: tests use a fake client that implements the same interface. The fake checks prompt assembly and response parsing only. Production uses the real Claude API. A manual check with a real key runs before each release.
- `web`: handler tests for the status moves and the auth rule.

## Out of v1

More source adapters (Gupy, Indeed, Glassdoor), email-alert ingestion, multi-user support, automatic job applications, and salary estimates.

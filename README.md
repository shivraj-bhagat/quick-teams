# Quick Teams

Quick Teams is a multi-tenant Rails app for running a team: people, projects, schedules, timesheets, goals,
surveys and reports, all in one place. Each company signs up as its own account and pays through Stripe.

## Tech stack

| Area | Choice |
|---|---|
| Language / framework | Ruby 3.4.7, Rails 8.1 |
| Database | PostgreSQL 16 |
| Front end | Hotwire (Turbo + Stimulus), StimulusReflex, importmap, Propshaft, Tailwind CSS 4 (no Node needed) |
| Background jobs / websockets | Solid Queue, Solid Cable (no Redis needed) |
| Auth | Devise, devise_invitable, devise-pwned_password |
| Multi-tenancy | acts_as_tenant (one tenant per `Account`) |
| Billing | Pay + Stripe |
| Files | Active Storage on S3, image_processing |
| PDFs | wicked_pdf + wkhtmltopdf |
| Other | Draper decorators, Pagy pagination, rails-patterns, New Relic |

## Features

### Accounts, sign-in and billing
- Sign-up and sign-in, invitations by email, and a check that rejects passwords known to have leaked.
- Each company is an isolated account; the owner can delete it.
- Stripe subscriptions: checkout, billing portal, success and expired pages.
- User settings: profile, password, avatar and personal preferences.

### Account setup
Admins manage the lists the rest of the app uses: roles, disciplines, skills, jobs, clients, people tags and
statuses, project tags and statuses, survey question categories, ticket statuses and labels, and account-wide
preferences (including notifications).

### People
- Employee directory with filters, search and stats; invite, deactivate and restore people.
- Each profile has: about, schedules, timesheets, goals, todos, feedback, skills, documents, reports, surveys,
  KPIs (with stats), team, timeline and observed projects.
- Status, tags, skills, KPIs and feedback update in place without a page reload.

### Projects
- Create, edit, archive, restore and delete projects, with tags, statuses, clients, filters and stats.
- Each project has: about, schedules, notes, risks, feedback, timesheets, milestones, todos, skills, reports,
  documents, surveys, KPIs, clients, timeline and observers.

### Scheduling and occupancy
- Resource schedule across people and projects.
- Occupancy views for the whole account, by job and by role.

### Timesheets, goals and todos
- Time logging per person and per project, with stats and filters.
- Goals with comments; todos on people and projects.
- Home dashboard with goals and events.

### Spaces
- Discussion spaces with messages and comments; pin, archive and restore spaces.

### Tickets
- Ticket tracker with open and closed views, statuses, labels and comments.

### Surveys and assessments
- Build surveys from questions, options and categories; clone surveys; reusable templates with assignees.
- Assign surveys, take them, preview attempts and see participant and survey stats.
- Submit and download survey reports, including as PDF; quick assessments from a profile.

### Reports
Available under `/report`:
- Timesheets, employees, projects, risks, observers, skills, activities and roles.
- Goals (all and open), KPIs and a performance report.
- Schedules: available now and next month, no schedule, no projects, overburdened, shared, by role.
- Todos: my pending, pending, all pending, recently added, recently finished.
- Status reports with comments and a submit flow.

### Search and archive
- Search people, projects, users, surveys and skills.
- Archive pages for archived projects, deactivated people and archived spaces.

### Email and background jobs
- Notification emails for comments, feedback, goals, messages, schedules, spaces, tickets and todos.
- A weekly activity digest email.
- Cleanup jobs that delete archived projects, deactivated users and old timesheets (see `config/recurring.yml`).

## Local setup

Prerequisites: Ruby 3.4.7 (`.ruby-version` / `mise.toml`), Docker, and Google Chrome for system tests.

1. Start Postgres 16 (published on port **5433** so it doesn't clash with a local Postgres):

   ```bash
   docker compose up -d
   ```

2. Install gems, create and seed the database:

   ```bash
   bin/setup
   ```

3. Turn on the development cache (StimulusReflex needs it):

   ```bash
   bin/rails dev:cache
   ```

4. Run the web server and the Tailwind watcher together:

   ```bash
   foreman start -f Procfile.dev
   ```

   `bin/dev` starts only the Rails server, so CSS changes won't rebuild with it. The app runs at
   http://localhost:3000. In development, sent emails open at http://localhost:3000/letter_opener.

### Database settings

Development and test read these variables, with the defaults shown:

| Variable | Default |
|---|---|
| `DB_HOST` | `localhost` |
| `DB_PORT` | `5433` |
| `DB_USERNAME` | `postgres` |
| `DB_PASSWORD` | `postgres` |
| `DB_NAME` | `quickteams` (databases are `<DB_NAME>_development` and `<DB_NAME>_test`) |

Give each checkout or worktree its own `DB_NAME` so their databases don't collide.

## Running tests

Run the suites in parallel:

```bash
PARALLEL_WORKERS=8 bin/rails test
```

```bash
PARALLEL_WORKERS=8 bin/rails test test/system
```

Each worker gets its own database (`<DB_NAME>_test-N`). System tests use Selenium with Chrome.

Security checks:

```bash
bundle exec brakeman
```

```bash
bundle exec bundle-audit
```

## Deployment

The `Procfile` runs two processes:

- `web`: Puma
- `worker`: Solid Queue

`bin/render-build.sh` is the build script.

Production settings:

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Primary database |
| `QUEUE_DATABASE_URL` | Solid Queue database (falls back to `DATABASE_URL`) |
| `CABLE_DATABASE_URL` | Solid Cable database (falls back to `DATABASE_URL`) |
| `RAILS_MASTER_KEY` | Decrypts `config/credentials.yml.enc` (AWS keys and other secrets) |
| `RAILS_SERVE_STATIC_FILES`, `RAILS_LOG_TO_STDOUT` | Platform settings |
| `WEB_CONCURRENCY`, `RAILS_MAX_THREADS`, `JOB_CONCURRENCY` | Process and thread tuning |

Production sends email over SMTP and stores files on S3.

## Branches and upgrade status

- `main`: the live line. It also has the daily/weekly planner, job cards and the culture tab on profiles.
- `dev`: the older production line (Rails 7.1, Sidekiq + Redis, esbuild).
- `upgrade` / `upgrade-base`: the Rails 8.1 / Solid Queue / importmap migration.

# From Heroku to Kamal: Migrating a Multi-Tenant Rails App with AI

**Talk for Ruby Meetup Buenos Aires — March 2026**

---

## Talk Outline

### 1. Context: What is SafetyPanel? (3 min)

- Multi-tenant Rails app for safety management, training, and compliance
- Subdomain-based tenancy with `acts_as_tenant` (each client gets `acme.safetypanel.com.ar`)
- Started September 2020, ~1100 commits, 5 contributors
- Stack at the start of 2026: Rails 7.2, Heroku, Sidekiq + Redis, wicked_pdf + wkhtmltopdf, Webpack

### 2. Why Leave Heroku? (3 min)

- **Cost**: cheapest dyno + Redis add-on + managed Postgres was expensive for what we got
- **Constraints**: 512MB memory limit forced workarounds (Heroku 8MB upload limit → direct uploads PR #409)
- **Dependency bloat**: Redis only used for Sidekiq + Action Cable (which wasn't even active)
- **wkhtmltopdf**: unmaintained binary, tricky in containers, old WebKit rendering
- **Control**: wanted zero-downtime deploys, SSH access, simple `bin/kamal console`

### 3. The Plan: 9 Steps, One Spreadsheet (5 min)

Describe the brainstorm session and the resulting epic issue #430 with its phased approach:

| Phase | Steps | Goal |
|-------|-------|------|
| Foundation | 1: Rails 8 upgrade | Unlock Solid gems |
| Remove Redis | 2-5: Solid Cache, ActiveJob, Solid Queue, remove Redis | Only dependency = PostgreSQL |
| Simplify Stack | 6: wicked_pdf → Ferrum | Modern PDF engine |
| Infrastructure | 7-9: DNS, Dockerfile + Kamal, Deploy | Leave Heroku |

**Key decision**: upgrade Rails and swap internals *on Heroku first*, then migrate infrastructure. This way the VPS gets a clean stack from day one.

**The dependency chain**:
```
Step 1 (Rails 8)
  ├── Step 2 (Solid Cache)
  ├── Step 3 (ActiveJob conversion)
  │     └── Step 4 (Solid Queue) ─── Step 5 (Remove Redis)
  └── Step 6 (Ferrum) ─── Step 7 (DNS) ─── Step 8 (Dockerfile) ─── Step 9 (Deploy)
```

### 4. The Sprint: Feb 10-25 (10 min) — The core of the talk

#### Feb 10 — Six PRs in One Day

The first 5 steps of the plan, merged in sequence in a single day:

1. **Rails 7.2.3 → 8.0.4** (#440) — `config.load_defaults` stayed at 7.2 for safety
2. **Solid Cache** (#442) — same PostgreSQL, 256MB max, barely used cache → minimal risk
3. **Sidekiq workers → ActiveJob** (#443) — only 2 workers, 5 new tests added
4. **Sidekiq → Solid Queue** (#444) — adapter swap + Mission Control UI + `recurring.yml`
5. **Remove Redis** (#445) — Action Cable → async adapter, delete `redis` gem
6. Schema update (#441) — housekeeping

**Demo/slide idea**: show the git log for Feb 10 — six PRs, methodical progression from Rails 8 to Redis-free in hours.

#### Feb 11 — Framework Defaults + PDF Engine

- **Rails 8.0 framework defaults** (#457) — `config.load_defaults 8.0`, safe because zero usage of affected features
- **wicked_pdf → ferrum_pdf** (#446) — the biggest code change:
  - New "preview" actions that render PDF as HTML (also useful for debugging!)
  - `PdfTokenAuthenticatable` concern — signed tokens (30s expiry) for headless Chrome
  - Chrome navigates to the preview URL and captures the PDF
  - Required temporary Heroku dyno upgrade (512MB → 1GB for Chrome)
- **Solid Queue memory tuning** (#447) — first production issue: 843MB on 512MB dyno
  - `batch_size` 500 → 100, `polling_interval` 0.1s → 1s
  - Added jemalloc buildpack for additional 20-40% memory reduction

#### Feb 13-23 — Building the Dockerfile + Kamal Config

- PR #461 created Feb 13, merged Feb 24 (12 commits, 19 files, +1198 lines)
- Multi-stage Dockerfile: `ruby:3.4.4-slim` → build stage (Node, gems, assets) → runtime
- Key ingredients: jemalloc, YJIT, Chromium, Thruster (HTTP/2), non-root user
- `bin/provision-vps` script: 485 lines of Ubuntu 24.04 hardening
- `docs/operations/infrastructure.md`: 335-line ops guide

#### Feb 24 — Deploy Day

- PR #461 merged
- Immediate fix: deploy workflow needed `contents: read` permission (#469) — lesson: when you explicitly set `permissions` in GitHub Actions, all defaults are revoked
- VPS IP and sshd provisioning fix (`a626563`) — the provision script had a minor sshd config issue

#### Feb 25 — Go-Live + First Fires

- SES email delivery (`180a06a`) — replaced SMTP with AWS SES API
- Sentry profiling fix (#471) — `stackprof` gem was silently a no-op
- ActiveJob retry on error (#480) — jobs needed explicit retry config
- Multiple client-facing fixes merged same day

### 5. The Hard Parts (5 min)

#### SSL/TLS: Two-Leg Chain
```
Client → Cloudflare (edge cert) → kamal-proxy (Cloudflare origin cert) → Puma
```
- Cloudflare Full (Strict) mode
- Origin certificate stored as GitHub Actions secrets, injected via `.kamal/secrets`
- `config.assume_ssl = true` + `config.force_ssl = true` in production.rb
- `ssl_redirect: false` in Kamal (Cloudflare already does HTTP → HTTPS)

#### PDF Rendering Inside Docker
- Chrome inside Docker can't resolve `acme.safetypanel.com.ar` → localhost
- Solution: `--host-resolver-rules "MAP *.safetypanel.com.ar <host-ip>"`
- `KAMAL_HOST` env var (auto-set by Kamal) provides the server IP
- `--ignore-certificate-errors` because Chromium doesn't trust Cloudflare's CA

#### Multi-Tenant Wildcard Subdomains
```yaml
# config/deploy.yml
proxy:
  hosts:
    - safetypanel.com.ar
    - "*.safetypanel.com.ar"
```
- DNS: wildcard A/CNAME in Cloudflare → VPS IP
- Rails `config.hosts` with regex for wildcard subdomain validation
- Health check `/up` excluded from host authorization

### 6. The AI-Assisted Workflow (5 min)

The entire migration was pair-programmed with Claude Code:

- **Brainstorming**: the migration plan, phased approach, dependency chain
- **Issue creation**: structured 9-step issue sequence with acceptance criteria
- **PR descriptions**: detailed "what/why/how" format, test plans, follow-up items
- **VPS provisioning script**: 485 lines of server hardening
- **Debugging**: CI permissions, sshd config, memory tuning, PDF rendering in Docker
- **Post-migration fixes**: rapid iteration on production bugs

**Key insight**: the AI doesn't just write code — it helps think through the *sequence* and *dependencies* of a migration. The plan was the most valuable artifact.

### 7. Current State + What's Next (2 min)

**Current stack** (post-migration):

| Component | Before | After |
|-----------|--------|-------|
| Rails | 7.2.3 | 8.0.4 |
| Hosting | Heroku | DigitalOcean VPS + Kamal 2 |
| Background jobs | Sidekiq (Redis) | Solid Queue (PostgreSQL) |
| Caching | Memory store | Solid Cache (PostgreSQL) |
| PDF generation | wicked_pdf (wkhtmltopdf) | ferrum_pdf (Chromium) |
| DNS | Route 53 | Cloudflare |
| HTTP proxy | — | Thruster (HTTP/2) |
| Deploy | `git push heroku` | `kamal deploy` via GitHub Actions |
| Redis | Required | Removed entirely |

**Still open** (issue #430 remains open):
- Replace simple_form with Rails form helpers (#454)
- Replace Devise with custom auth (#455)
- Serve public Active Storage via direct S3 URLs (#492)

### 8. Q&A (5 min)

---

## Complete Reference Index

### Epic Issue

| # | Title | State | URL |
|---|-------|-------|-----|
| 430 | Rails App Simplification & Heroku → VPS (Kamal) Migration | OPEN | https://github.com/safetypanel/safetypanel/issues/430 |

### Step Issues (in order)

| # | Step | Title | Closed |
|---|------|-------|--------|
| 431 | 1 | Upgrade Rails 7.2.3 → 8.x | 2026-02-10 |
| 432 | 2 | Set up Solid Cache | 2026-02-10 |
| 433 | 3 | Convert legacy Sidekiq::Workers to ActiveJob | 2026-02-10 |
| 434 | 4 | Sidekiq → Solid Queue + Mission Control | 2026-02-12 |
| 435 | 5 | Remove Redis entirely | 2026-02-12 |
| 436 | 6 | wicked_pdf → Ferrum/FerrumPDF | 2026-02-12 |
| 437 | 7 | DNS — Route 53 → Cloudflare | 2026-02-20 |
| 438 | 8 | New Dockerfile + Kamal configuration | 2026-02-24 |
| 439 | 9 | Deploy to VPS + decommission Heroku | 2026-02-25 |

### Core Migration PRs (chronological)

| # | Title | Merged | Key Change |
|---|-------|--------|------------|
| 440 | feat(rails): upgrade Rails 7.2.3 to 8.0.4 | 2026-02-10 | Rails 8, `load_defaults` at 7.2 |
| 441 | chore(db): update schema.rb for Rails 8.0 format | 2026-02-10 | Schema housekeeping |
| 442 | feat(cache): add Solid Cache as database-backed cache store | 2026-02-10 | `solid_cache` gem, same DB |
| 443 | feat(jobs): convert legacy Sidekiq workers to ActiveJob | 2026-02-10 | 2 workers → ActiveJob, 5 new tests |
| 444 | feat(queue): replace Sidekiq with Solid Queue and Mission Control | 2026-02-10 | Queue adapter swap, Mission Control, `recurring.yml` |
| 445 | chore(deps): remove Redis dependency | 2026-02-11 | `redis` gem removed, Action Cable → async |
| 446 | feat(pdf): replace wicked_pdf with ferrum_pdf | 2026-02-11 | Chrome-based PDFs, preview actions |
| 447 | fix(queue): tune Solid Queue config to reduce memory | 2026-02-11 | batch_size 500→100, polling 0.1s→1s |
| 457 | feat(config): enable Rails 8.0 framework defaults | 2026-02-12 | `config.load_defaults 8.0` |
| 461 | feat(deploy): Dockerfile, Kamal 2, and GitHub Actions deploy | 2026-02-24 | 19 files, 12 commits, full infra |
| 469 | fix(ci): fix deploy workflow permissions and trigger | 2026-02-24 | `contents: read` + manual dispatch |

### Post-Migration Fix PRs

| # | Title | Merged | Issue |
|---|-------|--------|-------|
| 471 | chore(sentry): add stackprof and enable logs | 2026-02-25 | Profiling was silently a no-op |
| 475 | fix(visits): hide attachment filename in report PDF | 2026-02-25 | PDF rendering regression |
| 477 | fix(visits): support overnight visits crossing midnight | 2026-02-25 | Time handling edge case |
| 479 | fix(routes): redirect all legacy establishment URLs | 2026-02-25 | Old URL structure compatibility |
| 480 | fix(jobs): add retry on error for ActiveJob | 2026-02-25 | Jobs needed retry config |
| 481 | fix(routes): remove auth from legacy establishment redirects | 2026-02-26 | Redirect auth issue |
| 486 | fix(pdf): inline base64 images in certifications PDF | 2026-02-27 | Image rendering in Chrome PDF |
| 485 | fix(performance): resolve N+1 queries across controllers | 2026-02-27 | Post-deploy perf fixes |
| 489 | fix(employees): correct training share link domain | 2026-02-27 | Domain in share URLs |

### Foundation PRs (pre-migration)

| # | Title | Merged | Why It Mattered |
|---|-------|--------|-----------------|
| 400 | chore(rails): upgrade from Rails 7.1 to 7.2 | 2026-01-06 | Stepping stone to Rails 8 |
| 404 | Upgrade gems and prepare for Rails 8.0 | 2026-01-14 | 40+ gems, enum syntax, Pagy v43 |
| 409 | Fix training PDF uploads over ~8MB | 2026-01-15 | Direct uploads to bypass Heroku limit |
| 416 | refactor(linters): replace standard/erb_lint with rubocop-rails-omakase and herb | 2026-01-27 | Modern linting stack |
| 332 | Migrate from Webpack to esbuild | 2025-08-26 | 10-100x faster JS builds |
| 361 | Remove DashForge theme dependency | 2025-11-17 | Eliminated paid theme |
| 368 | chore(deps): upgrade Bootstrap from 4.5.2 to 5.3.3 | 2025-11-25 | Modern CSS framework |

### Older Foundation (for the long-arc narrative)

| Commit/PR | Title | Date | Significance |
|-----------|-------|------|-------------|
| #132 | upgrade rails to 6.1 | 2023-11-11 | Rails 5 → 6.1 |
| #137 | update ruby to 3.1 and node to 20.x | 2023-12-21 | Ruby modernization |
| #167 | Migrate to Turbo | 2024-08-10 | Turbolinks → Turbo |
| #184 | upgrade to rails 7 | 2024-09-15 | Rails 6.1 → 7 |

### Key Commits (not PRs)

| SHA | Date | Description |
|-----|------|-------------|
| `a626563` | 2026-02-24 | fix(deploy): set production VPS IP and fix sshd provisioning |
| `180a06a` | 2026-02-25 | feat(mail): replace SMTP with SES API for email delivery |
| `0ee2597` | 2026-02-25 | fix(mail): require aws-sdk-sesv2 before delivery method |

### Infrastructure Artifacts

| File | Purpose |
|------|---------|
| `Dockerfile` | Multi-stage: ruby:3.4.4-slim + Chromium + jemalloc + YJIT + Thruster |
| `config/deploy.yml` | Kamal 2 config: GHCR, Cloudflare SSL, wildcard proxy |
| `.github/workflows/deploy.yml` | GitHub Actions: manual dispatch, kamal deploy |
| `.github/workflows/test.yml` | CI: run tests on push |
| `.github/workflows/lint.yml` | CI: RuboCop + Herb on push/PR |
| `.kamal/secrets` | Secret bridge: env vars → Kamal containers |
| `bin/provision-vps` | 485-line Ubuntu 24.04 hardening script |
| `bin/docker-entrypoint` | Runs `db:prepare` before server start |
| `bin/thrust` | Thruster HTTP/2 proxy wrapper |
| `docs/operations/infrastructure.md` | 335-line architecture + ops guide |

### DNS Migration PR (closed, used as runbook)

| # | Title | State |
|---|-------|-------|
| 463 | docs(dns): Route 53 to Cloudflare migration tutorial | CLOSED |

### Future Simplification Issues

| # | Title | State |
|---|-------|-------|
| 454 | chore(deps): replace simple_form with Rails form helpers | OPEN |
| 455 | chore(deps): replace Devise with custom authentication | OPEN |
| 492 | perf: Serve public Active Storage assets via direct S3 URLs | OPEN |

---

## How I Steered the AI at Every Step

> **Source**: Reconstructed from conversation session logs (`~/.claude/projects/` JSONL files), git history, and GitHub PRs/issues. Quotes are verbatim from the session logs. Sessions before Feb 19 are not stored locally (the Feb 10-15 sprint sessions were likely before conversation logging was enabled or were cleaned up).

### Phase 0: Foundation PRs (months before the migration)

| When | What changed | PR |
|------|-------------|-----|
| Aug 2025 | Webpack → esbuild | #332 |
| Nov 2025 | Remove DashForge theme | #361 |
| Nov 2025 | Bootstrap 4.5.2 → 5.3.3 | #368 |
| Jan 2026 | Rails 7.1 → 7.2 | #400 |
| Jan 2026 | Upgrade 40+ gems, fix enum deprecations for Rails 8 compat | #404 |
| Jan 2026 | Direct uploads to bypass Heroku 8MB limit | #409 |
| Jan 2026 | Replace linters (standard → rubocop-rails-omakase, erb_lint → herb) | #416 |

**[FILL IN]**: Were these deliberate preparation for leaving Heroku, or did the migration idea come later? What was the trigger?

### Phase 1: The Brainstorm and Plan (Feb 10)

*(Session not stored locally — this was before conversation logging was enabled for this project.)*

Issue #430 created with the 9-step plan, Current → Target stack table, and 6 "Key Decisions." PRs #440-#445 all merged the same day.

**[FILL IN]**: The Feb 10 sessions are missing. How did the brainstorm and sprint actually go?

### Phase 2: DNS Migration — Step by Step with Verification (Feb 20)

**Session `d7c6de3b`** — The user sat with Route 53, Cloudflare, and DonWeb all open simultaneously:

> "I have aws route 53, cloudflare, and donweb open. Guide me step by step on how to migrate the dns."

The session shows methodical, verify-at-every-step steering:

> "I already made the changes at nic.ar. How can I check if cloudflare is managing the traffic?"

Hit a snag with Cloudflare proxy breaking the site:

> "I disabled the proxy on all records (3) and now the site is accessible."

Steered toward Cloudflare origin certificates instead of Let's Encrypt:

> "I would like to migrate to the full strict mode. But I want the cloudflare cert. The one I generate via let's encrypt expires every 3 months."

Insisted on verifying email delivery before signing off — pasted full email headers to check DKIM/SPF/DMARC:

> "Can we test the mails before signing off?"

Only then:

> "delete the route 53 hosted zone"
> "Do I have a copy of all the records just in case?"

### Phase 3: Production Data — "NO CHANGES IN THE CODE!" (Feb 20)

**Session `3d4bf4fb`** — During the establishments-to-clients migration, a production data issue surfaced. Claude suggested a code change to fix it. The user escalated hard:

> "I cannot wait for the next PR. It is in production."

> "no changes in code. IT IS IN PRODUCTION"

> "NO CHANGES IN THE CODE!!!!! What if we change the name of the existing conflicting clients? Adding a prefix or suffix? Then the slug will be available for the new clients that replace the establishments"

This is a key steering moment: the user wanted **operational fixes via console**, not code PRs, for production data issues. After running the fix:

> "I ran all the steps and this is the new report" *(showed mappings.unmapped=0)*

> "THe four establishments seem to be from test or abandoned accounts."

### Phase 4: Local VM Testing — "Does it make sense?" (Feb 24)

**Session `4a20942b`** (13MB, the biggest session) — This is the session you mentioned. The user proposed testing on a local VM before going to production:

> "Now, is there a way to test the deploy locally? Of course we will not be testing the github action. I was thinking of spinning up a VM in this computer. Deploy to it. And test everything works. Does it make sense?"

**Architecture decisions made during this session:**

On Solid Queue process isolation:
> "I don't want to run solid queue within Puma. The app initially is going to be deployed to one host, but with two vcpu. So I want two different processes."

On SSL:
> "The DNS and the SSL certificates are handled by Cloudflare. And we use an origin certificate."

On the provision script — the user provided a complete ~500-line hardening script as a starting point:
> "I would like to add provision script so that I can copy and paste to set up vps."

On keeping things lean:
> "If the sample hooks are not going to be used, I would rather remove them."
> "I do not want the plan committed"
> "We use an S3 bucket for active storage. I do not think that we will need a volume for it."

On CI/CD, referencing another project as a model:
> "About the deploy from github: I have this working in another project" *(referenced boxful-engine deploy workflow)*
> "I want the deploy action for this project to be as lean & simple as possible."

**VM setup and bugs discovered:**

The user used Multipass for the local VM. During testing:

1. **Docker permission error**: User pasted `permission denied while trying to connect to the docker API`

2. **SSH lockout from overly aggressive hardening**:
   > "I waited several minutes and I still get Connection refused. Is it possible that there was too much hardening in the vps setup?"

3. **Subdomain routing**: The app uses subdomains, and the VM didn't resolve them:
   > "Remember that the app uses subdomains. When I visit admin.safetypanel.com.ar I get production"
   *(Needed `/etc/hosts` entries pointing subdomains to the VM)*

4. **Login broken** (self-signed cert + Turbo interaction):
   > "It doesn't work" / "I get no error. It just doesn't log in"

5. **Professional form bug**: User pasted a 500 error:
   > `ActionView::Template::Error (Cannot get a signed_id for a new record)`

6. **PDF rendering failure**: Chromium inside the Docker container couldn't resolve subdomain URLs. User showed a screenshot of a PDF with a 404 in the body. This led to the `--host-resolver-rules` solution.

7. **SSL comment inaccuracy** — user caught a wrong comment in `production.rb`:
   > "Reviewing config/environments/production.rb I see [comment about Cloudflare]. But cloudflare is not terminating the ssl. Right?"
   > "I mean, the config is correct but the comment is wrong?"
   > "But isn't kamal-proxy the one that terminates the ssl with the origin certificate?"

**Sign-off checklist** — user created their own before closing:
> "Before we sign off there are some things to tackle: 1. reset my dev env 2. Did we consider softening the vps security? I am not a linux expert and I would like to NOT get locked out of my VPS. 2.1 What would be the sudo password for the linux user? 3. See the WAF exception for PDF generation -- should we still do it? 4. I would like to add a docs/operation/infrastructure.md"

### Phase 5: Deploy Day — "Come on... the app is down!" (Feb 25)

**Session `a7f86a2b`** (1.4MB) — The most stressful session. User deployed to the production DigitalOcean VPS.

**VPS provisioning:**
> "I am about to tackle issue 439 (Step 9: Deploy to VPS + decommission Heroku)"
> "I created the Digital Ocean project. Still have to create the droplet and managed db."
> "The ip is: 161.35.62.142. My ssh key and the github actions one are already set."

Provision script hit a real-world issue:
> *(pasted)* `ERROR: Invalid sshd config, skipping SSH restart`

**First deploy:**
> "Shouldn't we run kamal setup or something like that first?"

TLS error after deploy — user pasted `curl` output:
> `curl: (35) error:0A000438:SSL routines::tlsv1 alert internal error`

Then pasted full kamal proxy logs showing healthcheck failures, then eventual success.

**Database migration — the high-stress moment:**

The user set maintenance mode, dumped from Heroku, and started restoring. Then everything went wrong:

- `db:drop` failed: protected environment error
- `DISABLE_DATABASE_ENVIRONMENT_CHECK=1` syntax error
- `pg_restore` timed out: managed DB only accessible from VPC, not from local machine
- Had to SCP the dump to the server, then run docker commands remotely
- Docker run syntax kept failing on the remote shell

The frustration escalated:

> "Come on... the app is down! Think harder"
> "Cooooooooooooooooooooooooooooooooooooooooooooomme onnnnnn"

Then:
> "It looks like it worked."

**Verification:**
> "They match. Should I check something about the attachments (active storage?)"

User verified on both sides — pasted Heroku and DO console output showing `ActiveStorage::Blob.count => 421966` and matching S3 URLs.

**SMTP blocked by DigitalOcean — the pivot to SES:**

> "How can I trigger an email?"

Email timed out: `execution expired (Net::OpenTimeout)`. User diagnosed it:

> *(ran)* `ssh deploy@161.35.62.142 "timeout 5 bash -c '</dev/tcp/email-smtp.us-east-2.amazonaws.com/587' && echo OK || echo BLOCKED"` → BLOCKED

> "give me the text I should send to DO"

DigitalOcean responded: *"we are restricting SMTP traffic across the DigitalOcean platform... We recommend utilizing alternative ports such as port 2525. You can also use REST API"*

> "465 is also blocked. Everything else works"

The user made the pivot decision on the spot:
> "As an alternative, is there a gem to use aws ses via API?"
> "Both services, the s3 bucket and aws ses, are under the same user. Is that what you mean?"

After deploying the SES fix:
> "Tested the email. It works"

**Heroku decommission:**
> "yes, scale it down"
> "Should I scale down the worker as well?"
> "yes, update it. Include a comment with all the relevant details from this conversation"

### Phase 6: Post-Migration — User Reports from Production (Feb 25-27)

**Session `47c6ab40`** — User wrote a changelog for the app owner:
> "/changelog since commit 096ac9e. In Spanish for a non-technical audience (app owner). It should mention that the app is running on the new server (Digital Ocean) and that we are keeping the old one (Heroku) until 2026-03-03 just in case."

**Session `f162792a`** — Legacy establishment links broken. User received bug reports from real users:

> "I am receiving a report that users can not use the legacy link https://tetrapak.safetypanel.com.ar/visitor/establishments/victoria/trainings"

> "I am receiving reports that the account calsalanus has the same problem"

User ran production diagnostics themselves (pasted console output and kamal logs), then made a judgment call about a secondary reported issue:

> "Before we apply these changes, I think that there is no scenario where a client or a user will reach this link directly."
> "should we care if someone visits this link directly? Or should we only worry about handling the scenarios we encourage?"

Then asked for the explanation in Spanish for the app owner:
> "Give me this explanation in Spanish so that I can share it with the app owner"

### Phase 7: CI Fix — User Diagnosed Root Cause (Feb 24)

**Session `8c50bd29`** — After the deploy PR merged, CI failed. The user diagnosed it themselves:

> "I got ActiveSupport::MessageEncryptor::InvalidMessage and the CI failed"
> "Regarding the master key, I found the problem. In my .env I had the production master key because I was doing some kamal deploy tests."

### The Pattern (from actual session evidence)

Across all sessions, the consistent pattern was:

1. **User decided the approach**: Solid Queue as separate process, Cloudflare origin cert (not Let's Encrypt), local VM testing before production, SES API (not SMTP), operational fixes for production data (not code PRs)
2. **User provided starting material**: the ~500-line provision script, the boxful-engine deploy workflow as a reference, the full error output for every failure
3. **User verified at every step**: email headers for DNS, `ActiveStorage::Blob.count` on both sides for data migration, sign-off checklist before closing sessions
4. **User diagnosed root causes**: master key contamination, SMTP port blocking, SSL comment inaccuracy, production slug conflicts
5. **User course-corrected hard when needed**: "NO CHANGES IN THE CODE!!!!", "Come on... the app is down! Think harder", catching wrong comments in production.rb
6. **User triaged post-migration reports**: ran production diagnostics, made judgment calls about which issues to fix vs. which to ignore, communicated with the app owner in Spanish

### Future Simplification (still open)

- **#454** — Replace simple_form with Rails form helpers (42 forms)
- **#455** — Replace Devise with custom authentication (69 files)
- **#492** — Serve public Active Storage via direct S3 URLs

---

## Talk Timing Guide

| Section | Minutes | Cumulative |
|---------|---------|-----------|
| Context: What is SafetyPanel? | 3 | 3 |
| Why Leave Heroku? | 3 | 6 |
| The Plan: 9 Steps | 5 | 11 |
| The Sprint: Feb 10-25 | 10 | 21 |
| The Hard Parts | 5 | 26 |
| The AI-Assisted Workflow | 5 | 31 |
| Current State + What's Next | 2 | 33 |
| Q&A | 5 | 38 |

**Total: ~38 minutes** (adjust sections 4 and 5 for shorter slots)

---

## Suggested Slides / Demos

1. **Before/After stack table** — the Current → Target table from issue #430
2. **Git log for Feb 10** — show the six-PR sequence in the terminal
3. **Dependency diagram** — the step dependency chain
4. **Dockerfile walkthrough** — highlight multi-stage build, jemalloc, YJIT, Chromium
5. **`config/deploy.yml`** — explain the Kamal config with SSL + wildcard proxy
6. **Architecture diagram** — Internet → Cloudflare → kamal-proxy → Thruster → Puma
7. **`bin/kamal console` demo** — show how easy ops are now
8. **GitHub Actions deploy workflow** — the full CI/CD pipeline
9. **The SSL chain diagram** — two-leg TLS with Cloudflare origin certs
10. **Chrome `--host-resolver-rules`** — the PDF rendering hack explained

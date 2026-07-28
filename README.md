# Camp 8

Georgia high school football recruiting intelligence platform.
Operated by ASA Professional Services. Live at **[fbintel-central.com](https://fbintel-central.com)**.

> **Working branch is `redesign2`, not `main`.** All development happens there.

---

## What this is

Camp 8 gives high school football athletes an AI-driven athlete rating, a college coaches directory with an AI email writer, a camp calendar, NIL guidance, and a health and wellness module.

The athlete is the customer — not coaches, not fans, not media. That distinction drives most product decisions.

**The core IP** is the Camp 8 Athlete Rating: an 8-factor deterministic score on a 300–850 scale, with profile-driven and market-driven weight modifiers. See [Scoring engine](#scoring-engine).

---

## Quick start

```bash
gh repo clone SidneySwanCPT/camp8
cd camp8
git checkout redesign2
```

It's a static site — no build step. Open `index.html` directly or serve it with Live Server.

To test the serverless functions (AI chat, scoring) you need the Netlify CLI and the environment variables:

```bash
npm install -g netlify-cli
netlify dev
```

**Environment variables are not in this repo.** Request them from the project owner.

---

## Architecture

```
Local  →  GitHub (redesign2)  →  Netlify  →  fbintel-central.com
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            Netlify Functions                  AWS EC2
       (chat, score, scrape, ingest)     (Puppeteer + cron jobs)
                    │                               │
                    └───────────┬───────────────────┘
                                │
                          Supabase (Postgres + Auth)
```

| Layer | Service | Notes |
|---|---|---|
| Hosting | Netlify | Auto-deploys ~60s after every push to `redesign2` |
| Serverless | Netlify Functions | Holds all API keys and proprietary logic |
| Database + Auth | Supabase | Postgres with Row Level Security, PKCE auth |
| Backend | AWS EC2 t2.micro | `18.222.223.212:3001`, PM2-managed, runs scrapers and cron jobs |
| AI | Anthropic Claude | Accessed only through `netlify/functions/chat.js` |
| Maps | Leaflet | CDN |
| Mobile | Capacitor 8.3 → Xcode | Bundle ID `com.asaprofessional.camp8` |

---

## Repo layout

### Pages — public

| File | Purpose |
|---|---|
| `index.html` | Home. Hero, AI panel, feature bar, camp teasers, countdown |
| `login.html` | The only public entry point to gated content |
| `nil-central.html` | NIL hub — rules and deals tracker |
| `ncaa-rules.html` | NCAA recruiting rules reference |
| `powerhouse.html` | Georgia program power rankings + map |
| `recruits.html` | Prospect hero; actions gate behind login |
| `commitments.html` | Commitment tracker |
| `transfer-portal.html` | Transfer portal tracker |
| `camps.html` | 755-camp database — map, list, calendar views |
| `privacy.html` · `terms.html` | Legal. Required for App Store and Meta API review |
| `404.html` | Branded error page, wired in `netlify.toml` |

### Pages — login required

| File | Purpose |
|---|---|
| `dashboard.html` | Athlete home. Profile, verified badge, tabs, calendar, score visuals |
| `onboarding.html` | 8-step setup wizard. Runs once after signup |
| `nil-rating.html` | Conversational NIL evaluator with a 6-factor progress tracker |
| `combine-eval.html` | Conversational combine grader, 7 factors |
| `coaches.html` | Coaches directory + AI coach-email writer |
| `health.html` | Goals, AI workout builder, macro calculator, wellness coach |
| `chat.html` | AI mentor, 7 topic modes, accepts `?topic=` |
| `athlete-hub.html` · `by-school.html` · `nil-sponsors.html` | Secondary athlete tools |
| `admin.html` | Owner-only. News queue, CRUD, stat-dispute escalation |

### Core logic

| File | Purpose |
|---|---|
| `nav.js` | **Universal navigation.** One file controls the nav on every page |
| `auth-guard.js` | **Session gate.** Public-page allow-list, login and onboarding redirects |
| `shared.js` | Utilities — escaping, formatting, toasts, loading states, debounce |
| `supabase-client.js` | Supabase client **singleton** with PKCE config |
| `ai-chat.js` | Shared chat UI engine used by every AI tool |
| `login-splash.js` | Motivational message shown after login |
| `dashboard-widgets.js` | Calendar, radar chart, score history graph |
| `*-map.js` | Leaflet renderers for camps, recruits, programs, schools |
| `*-data.js` | Static seed data — coaches, camps, recruits, programs, NIL, sponsors |

### Serverless functions

| File | Purpose |
|---|---|
| `netlify/functions/chat.js` | Anthropic proxy. Holds the API key, rate limits 30 req/min per IP |
| `netlify/functions/score.js` | **Proprietary scoring engine.** Deterministic, weights read from Supabase |
| `netlify/functions/scrape.js` | Bridge to the EC2 backend for MaxPreps scraping |
| `netlify/functions/signal-ingest.js` | Stores incoming athlete signals |
| `netlify/functions/pattern-detect.js` | Correlation analysis feeding the learning layer |
| `netlify/functions/event-reminders.js` | Calendar prep reminders |

### Build artifacts — do not hand-edit

`ios/` and `public/` are generated by `npx cap sync ios`.

---

## Scoring engine

Deterministic by design: **the same inputs plus the same trend data must always produce the same score.** The AI collects data conversationally; it never computes the score.

| # | Factor | Max |
|---|---|---|
| 01 | Recruiting Profile | 150 |
| 02 | Combine Performance | 120 |
| 03 | NIL Market Value | 100 |
| 04 | Academic Standing | 100 |
| 05 | Film & Exposure | 90 |
| 06 | Position Value | 80 |
| 07 | Health & Wellness | 70 |
| 08 | Character & Leadership | 60 |
| | **Raw total** | **770** → mapped to 300–850 |

**Bands:** 300–499 Developing · 500–619 Rising · 620–719 Prospect · 720–779 Scholarship · 780–850 Elite

**Weights are data, not code.** They live in the `formula_weights` table and are read at runtime, so they can be tuned without a deploy. Every score row records `formula_version` and `modifier_version` so any historical score can be reproduced.

**Dynamic modifiers** adjust weights per athlete — elite measurables reduce the camp/exposure requirement, rural markets get a reduced exposure penalty, unverified data carries a confidence discount, and weekly market trends shift position value. This is what lets a 6'7" 250lb tight end with no stats score accurately.

---

## Database

Supabase Postgres. Run `schema.sql` first, then `schema_v2.sql`.

`athletes.user_id` needs a `UNIQUE` constraint before `stats_disputes` will accept its foreign key:

```sql
ALTER TABLE athletes ADD CONSTRAINT athletes_user_id_unique UNIQUE (user_id);
```

**Core:** `athletes` · `athlete_combine_scores` · `athlete_nil_scores` · `athlete_starred_camps` · `athlete_offers` · `athlete_goals` · `athlete_calendar`

**Admin:** `camps_admin` · `commitments_admin` · `nil_deals_admin` · `coaches_admin` · `pending_news`

**Scoring:** `athlete_scores` · `athlete_signals` · `formula_weights` · `trend_weights` · `detected_patterns` · `athlete_outcomes`

**Integrity:** `stats_disputes` — escalates to admin review after 2 disputes in 30 days

Row Level Security is on for athlete-facing tables. Athletes read and write only their own rows. Admin operations use the service role key, which **must never reach the browser**.

---

## Workflows

### Deploy to web

```bash
git pull                 # always pull first
git branch               # confirm * is on redesign2
git status && git diff   # review before staging
git add .
git commit -m "describe the change"
git push                 # Netlify deploys automatically
```

### Deploy to iOS

```bash
git push                 # push web changes first
npx cap sync ios         # copies web files into the iOS bundle
npx cap open ios         # opens Xcode
# Cmd + R to run in simulator
# Product > Archive to submit
```

### Database changes

Add a new `schema_vN.sql` file — never edit an existing one. Use `IF NOT EXISTS` so it's re-runnable. Add RLS policies for any new athlete-facing table. Commit the SQL file.

### EC2 backend

```bash
ssh -i camp8-key.pem ubuntu@18.222.223.212
pm2 logs camp8-backend --lines 50    # first stop when debugging
pm2 restart camp8-backend
curl http://18.222.223.212:3001/health
```

Scheduled jobs: news scanner (daily 8am UTC) · coach checker (Mondays 9am UTC) · stale detector (daily 7am UTC).

---

## Conventions

- **Branch** — all work goes to `redesign2`. Don't merge to `main` without asking.
- **Navigation** — add or move nav links only in `nav.js`. Never hardcode a nav bar in a page.
- **Gating** — new gated pages must be handled in `auth-guard.js`. Test the logged-out path every time.
- **Styling** — use the CSS variables in `styles.css` (`--gold`, `--bg-card`, `--text-secondary`). Never hardcode the gold hex.
- **Escaping** — run all user and scraped data through `esc()` from `shared.js` before injecting into HTML.
- **Secrets** — any code touching an API key belongs in `netlify/functions/` or on EC2. No exceptions.
- **Mobile** — test every change at 375px, 390px, and 768px.
- **Supabase client** — keep it a singleton. Duplicate `createClient` calls cause `GoTrueClient` warnings and break auth in the iOS app.

---

## Compliance

**This platform serves minors.** Confirm with the project owner before shipping any change that touches minors' data, authentication, or third-party scraping.

| Area | What it means |
|---|---|
| COPPA (under 13) | Verifiable parental consent required. Penalties up to $50k per violation |
| State minor privacy (13–17) | CA, VA, CO, CT extend protections. Parental consent for social linking is best practice |
| Apple App Store | Requires live privacy + terms URLs, Sign in with Apple, and a genuine native feature |
| Meta / X APIs | Use official APIs for organization data. Scraping personal accounts violates their terms |
| MaxPreps ToS | Only scrape when the athlete supplies their own URL and consents |
| Disclaimers | The coaching-changes notice on `coaches.html` and the no-guarantee language in `terms.html` must stay |

---

## Open items

**Launch blockers:** Google Login · Sign in with Apple · PWA manifest and service worker · full mobile QA pass · resolve the Capacitor `GoTrueClient` duplicate-instance warning

**Next:** X/Twitter API ingestion · Meta Graph API (2–4 week review, start early) · YouTube Data API · live MaxPreps scoring · 247Sports/On3 ingestion · measurables override and geographic exposure scoring

**Then:** partner submission portal · featured camp listings · push notifications · TestFlight and App Store · outcome tracking and learning layer

**Security debt:** EC2 port 3001 is open to `0.0.0.0/0` — restrict to Netlify's outbound ranges. Backend traffic is plain HTTP — set up `api.fbintel-central.com` with TLS.

---

## Contact

Questions on architecture, scoring, or anything touching user data — ask the project owner before shipping.

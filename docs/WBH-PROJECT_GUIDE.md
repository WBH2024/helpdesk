# Frappe Helpdesk — Developer Project Guide (WBH)

A reference for developers who extend, add features to, or fix bugs in this repository.

- **Snapshot:** branch `WBH-9-2026`, app version `1.30.1` (`helpdesk/__init__.py`), last commit `1c3361c` "chore(release): Bumped to Version 1.30.1" (2026-09-03). The history contains no WBH-specific commits yet, so this checkout matches upstream `frappe/helpdesk` at that release (checked with `git log`).
- **How to read the citations:** every claim names the file(s) behind it in backticks, with `:line` where useful. Line numbers are from the snapshot above and will drift.
- **"Unverified"** marks anything inferred from names, comments, or partial reads that was not confirmed by reading the code path end to end or by running it. Nothing in this guide was confirmed by running the app, because no bench or site exists in the environment where it was written.
- **Companion files:** `AGENTS.md` (the project's binding coding rules; this guide summarises them in §5 but does not replace them) and `CLAUDE.md` (a short command-and-convention cheat sheet).

## Contents

1. [Purpose and main features](#1-purpose-and-main-features)
2. [Tech stack, setup, and how to run it](#2-tech-stack-setup-and-how-to-run-it)
3. [Folder and module map](#3-folder-and-module-map)
4. [Architecture and data flow](#4-architecture-and-data-flow)
5. [Where to start, and the conventions to follow](#5-where-to-start-and-the-conventions-to-follow)
6. [Known issues, TODOs, and fragile areas](#6-known-issues-todos-and-fragile-areas)

---

## 1. Purpose and main features

**Purpose.** Frappe Helpdesk is an open-source (AGPLv3) ticket-management and customer-service application. It is a **Frappe app**: it cannot run alone and is installed into a Frappe "bench" site next to the `frappe` framework and the `telephony` app (`README.md`, `helpdesk/hooks.py:1-9`, `pyproject.toml` `[tool.bench.frappe-dependencies]`). The UI is a Vue single-page app served at `/helpdesk` (`helpdesk/hooks.py:49-54`, `pyproject.toml` `[tool.bench.assets]`).

**Main features** (each row names where it lives):

| Feature | What it does | Where it lives |
|---|---|---|
| Agent portal and customer portal | One SPA with two "roots": agents work tickets; customers (portal users with the `HD Customer`/`HD Customer Manager` roles) raise and follow their own tickets | `README.md` "Key Features"; roles set up in `helpdesk/setup/install.py:214-247`; frontend roots in `desk/src/roots/` |
| Tickets | The core record (`HD Ticket`), created from email, the portal, or the agent UI, with status, priority, type, team, assignment, and a timeline of emails, comments, and activities | `helpdesk/helpdesk/doctype/hd_ticket/`, `helpdesk/api/ticket.py` |
| Email-to-ticket and replies | Incoming mail on a support Email Account becomes a ticket or a reply; agents reply by email from the ticket | `helpdesk/overrides/email_account.py` (overrides the `Email Account` class via `helpdesk/hooks.py:123-127`), `helpdesk/api/settings/email.py`, `helpdesk/templates/emails/` |
| Configurable statuses | Custom ticket statuses grouped into categories (Open / Paused / Resolved); a separate customer-facing label is supported | `helpdesk/helpdesk/doctype/hd_ticket_status/`; defaults in `helpdesk/setup/install.py:386-424` |
| SLAs | Response and resolution targets per priority, working hours, and holiday lists; SLA status is refreshed hourly | `helpdesk/helpdesk/doctype/hd_service_level_agreement/`, `hd_service_level_priority/`, `hd_service_day/`, `hd_service_holiday_list/`, `hd_holiday/`; scheduler in `helpdesk/hooks.py:43-45`; default "Standard" SLA in `helpdesk/setup/install.py:64-134` |
| Assignment rules and teams | Auto-assign tickets through Frappe's `Assignment Rule` ("Round Robin" and "Load Balancing" rules get Helpdesk-specific handling in `helpdesk/overrides/assignment_rule.py:41`), per team or site-wide | `helpdesk/helpdesk/doctype/hd_team/`, `helpdesk/overrides/assignment_rule.py`, `helpdesk/extends/assignment_rule.py`, `helpdesk/api/assignment_rule.py` |
| Knowledge base | Articles in categories, article feedback, and article suggestions while a customer writes a ticket (`README.md` screenshots section) | `helpdesk/helpdesk/doctype/hd_article/`, `hd_article_category/`, `hd_article_feedback/`, `helpdesk/api/knowledge_base.py`, `helpdesk/api/article.py` |
| Search | Full-text search built on Frappe's SQLite search framework, plus stopwords and synonyms | `helpdesk/search_sqlite.py` (registered at `helpdesk/hooks.py:33`), `helpdesk/search.py`, `hd_stopword/`, `hd_synonym/`, `hd_synonyms/` |
| Saved replies | Reusable reply templates, optionally scoped to teams | `helpdesk/helpdesk/doctype/hd_saved_reply/`, `hd_saved_reply_team/`, `helpdesk/api/saved_replies.py` |
| Customer feedback | Ratings and feedback options after resolution, including an email feedback web form | `hd_ticket_feedback_option/`, `hd_email_feedback/`, `helpdesk/helpdesk/web_form/email_feedback/`, `helpdesk/setup/ticket_feedback.py` |
| Saved views | Per-user or public list views (columns, filters, sort) for tickets and other lists, with standard views shipped on install | `helpdesk/helpdesk/doctype/hd_view/`, `helpdesk/setup/default_views.py` |
| Ticket templates, field layout, form scripts | Admins choose which fields a ticket form shows (templates, layouts) and can attach client-side scripts to forms | `hd_ticket_template/`, `hd_ticket_template_field/`, `hd_field_layout/`, `hd_form_script/` |
| Customers and contacts | `HD Customer` organisations linked to Frappe `Contact` records; customer managers may also see their organisation's tickets (`_is_customer_manager` check in `helpdesk/helpdesk/doctype/hd_ticket/hd_ticket.py:1340`) | `helpdesk/helpdesk/doctype/hd_customer/`, `hd_customer_member/`, `helpdesk/api/customer.py`, `helpdesk/api/contact.py` |
| Agents and presence | Agent records, availability status (Active / Away / Unavailable), @-mentions, in-app notifications | `hd_agent/`, `hd_agent_status/`, `hd_notification/`, `helpdesk/mixins/mentions.py`, `helpdesk/api/agent.py` |
| Live collaboration | Who-is-viewing and who-is-typing indicators on a ticket, over Socket.IO | `realtime/handlers.js` |
| Dashboards and analytics | Agent home page, ticket statistics, and analytics charts; Frappe desk reports | `helpdesk/api/dashboard.py`, `helpdesk/api/ticket_analytics.py`, `helpdesk/api/ticket_stats.py`, `helpdesk/api/agent_home/`, `helpdesk/helpdesk/report/` |
| Onboarding and invitations | Setup checklist for new sites; invite agents or customers by email through Frappe's `User Invitation` | `helpdesk/api/onboarding.py`, `helpdesk/hooks.py:56-74`, `helpdesk/overrides/user_invitation.py`, `helpdesk/helpdesk/hooks/user_invitation.py` |
| ERPNext integration (optional) | Two-way sync of `HD Customer` ↔ ERPNext `Customer`, and mirroring of their `User Permission` and `DocShare` records, when ERPNext is installed and the integration is enabled | `helpdesk/integrations/erpnext/`, `helpdesk/helpdesk/doctype/erpnext_hd_settings/`, wired by `doc_events` in `helpdesk/hooks.py:76-103` |
| Automatic housekeeping | Daily auto-close of tickets after N days; search index build and NLP corpus download | `helpdesk/hooks.py:24-46` |
| Translations | About 30 locales, synced with Crowdin | `helpdesk/locale/*.po`, `crowdin.yml`, `.github/workflows/generate-pot-file.yml` |

## 2. Tech stack, setup, and how to run it

### 2.1 Tech stack

| Layer | Technology and version | Source |
|---|---|---|
| Backend language | Python ≥ 3.10 (CI uses 3.12) | `pyproject.toml` `requires-python`; `.github/workflows/server-tests.yml` |
| Backend framework | Frappe Framework `>=15.116.1,<17.0.0` (so v15 and v16) | `pyproject.toml` `[tool.bench.frappe-dependencies]`; `README.md` compatibility matrix |
| Required sibling app | `telephony` (`>=0.0.1,<1.0.0`) | `helpdesk/hooks.py:9`; `pyproject.toml` |
| Optional sibling app | ERPNext (turns on the customer-sync integration; CI installs it) | `helpdesk/integrations/erpnext/utils.py:222-225`; `.github/helpers/install.sh` |
| Python dependency | `textblob==0.18.0.post0`; NLTK corpora are downloaded at runtime | `pyproject.toml`; `helpdesk/search.py:377-387` |
| Database / cache / queue | MariaDB (CI 10.6, Docker 10.8), Redis | `.github/workflows/server-tests.yml`; `docker/docker-compose.yml` |
| Realtime | Frappe's Socket.IO server, plus Helpdesk handlers in `realtime/handlers.js`. That Frappe loads this file automatically by its path is **Unverified**. | `realtime/handlers.js` |
| Frontend | Vue `^3.5.14`, `frappe-ui` `1.0.0-beta.24` (patched with patch-package), Pinia `^2.0.33`, vue-router `^4.2.2`, `@vueuse/core` `^13.8.0`, Tiptap `^3.26.0`, `socket.io-client` `^4.7.2`, Tailwind `^3.4.15`, TypeScript `^5.0.2` | `desk/package.json`; `desk/patches/frappe-ui+1.0.0-beta.24.patch` |
| Frontend build | Vite `^4.4.9` with the frappe-ui Vite plugin (Frappe proxy, Lucide icons, Jinja boot data), `@framework/ui`, Vue JSX, and a PWA plugin; output goes to `helpdesk/public/desk`, and the HTML entry to `helpdesk/www/helpdesk/index.html` | `desk/vite.config.js:12-93`; `pyproject.toml` `[tool.bench.assets]` |
| Python packaging | `flit_core` | `pyproject.toml` `[build-system]` |

> **Setup gotcha: `@framework/ui`.** `desk/package.json:13` declares `"@framework/ui": "link:../../frappe/ui"`, and `desk/vite.config.js:1,45` imports and uses it. The frontend therefore expects `frappe-bench/apps/frappe/ui` to exist. Which Frappe versions ship that `ui/` folder is **Unverified**; if `yarn dev` or `yarn build` fails to resolve `@framework/ui`, check that path first.
>
> **The `frappe-ui` git submodule is not checked out** in this clone (`git submodule status` shows `-a3d141c…`). It is only needed if you want to develop against a local frappe-ui: `desk/vite-helpers.ts:29-41` switches to `../frappe-ui` in development mode only when `frappe-ui/node_modules` exists.

### 2.2 Setup options

**A. Local bench (recommended for WBH work).** These steps come from `README.md` "Local". Replace the upstream URL with your fork's URL or a local path so that you run *this* code:

```bash
# 1. Install bench and create a bench (Frappe docs: frappeframework.com/docs/user/en/installation)
bench init frappe-bench --frappe-branch version-15   # or a v16 branch; see pyproject.toml
cd frappe-bench
bench new-site helpdesk.test
bench --site helpdesk.test add-to-hosts
bench get-app https://github.com/frappe/telephony      # required app (helpdesk/hooks.py:9)
bench get-app <your-fork-url-or-local-path-to-this-repo>
bench --site helpdesk.test install-app helpdesk          # runs helpdesk.setup.install.after_install
bench build --app helpdesk
bench start                                              # web on :8000, plus workers, scheduler, socketio
# Open http://helpdesk.test:8000/helpdesk
```

`install-app` runs `after_install` (`helpdesk/hooks.py:24`, `helpdesk/setup/install.py:17-39`). It creates the custom fields, the default statuses (Open, Replied, Resolved, Closed), the agent statuses, the default article category and article, the priorities (Urgent, High, Medium, Low), a "Standard" SLA with Mon–Fri 10:00–18:00 hours and a "Default" holiday list, the teams "Billing" and "Product Experts", role permissions, the customer roles, a default assignment rule, the default ticket template and ticket types, the standard views, a welcome ticket, and the full-text indexes on `HD Ticket.subject` and `HD Ticket.description`.

**B. Frontend hot reload.** From `README.md` "For Frontend Development":

```bash
cd frappe-bench/apps/helpdesk/desk
yarn install        # its postinstall runs patch-package (desk/package.json scripts)
yarn dev            # Vite dev server; README says it is reachable at http://helpdesk.test:8080
```

The frappe-ui plugin runs with `frappeProxy: true` (`desk/vite.config.js:22`), which proxies API calls to the bench web server. That the proxy targets port 8000 by default is **Unverified** (it is the frappe-ui plugin's behaviour, not code in this repo). The root `package.json` only forwards to `desk/`: `yarn dev` and `yarn build` there run `cd desk && yarn dev` and `cd desk && yarn build`.

**C. Docker.** `docker/docker-compose.yml` starts MariaDB, Redis, and a `frappe/bench` container that runs `docker/init.sh`. That script creates the site `helpdesk.localhost` (admin password `admin`), installs telephony and helpdesk, and turns on `developer_mode`, `mute_emails`, and `server_script_enabled` (`docker/init.sh:28-38`). **Caveat for WBH:** `docker/init.sh:26` runs `bench get-app helpdesk --branch main`, which fetches the app from upstream rather than from your checkout. The compose file mounts your folder at `/workspace` (`docker/docker-compose.yml:25`), but `init.sh` never installs from it. That `bench get-app helpdesk` resolves to `github.com/frappe/helpdesk` is **Unverified**, but nothing in the script points it at `/workspace`.

**D. Production.** Use Frappe Cloud or the `easy-install.py` flow with the `ghcr.io/frappe/helpdesk` image (`README.md` "Production Setup"). The image is built by `.github/workflows/build.yml` from `frappe_docker` with Frappe `version-15`, telephony `develop`, and helpdesk `main`.

### 2.3 Day-to-day commands

| Task | Command | Source |
|---|---|---|
| Run the whole backend test suite | `bench --site <site> run-tests --app helpdesk` | `.github/workflows/server-tests.yml` ("Run Tests") |
| Run one test module | `bench --site <site> run-tests --module helpdesk.helpdesk.doctype.hd_ticket.test_hd_ticket` | Standard Frappe CLI; **Unverified** in this repo |
| Test-site prerequisites | CI installs **erpnext, telephony, and helpdesk** into the test site. `before_tests` resets HD Settings, creates an SLA, holiday list, email account, and the ERPNext `hd_customer` field, and runs the ERPNext setup wizard if ERPNext is present | `.github/helpers/install.sh`; `helpdesk/hooks.py:143`; `helpdesk/test_utils.py:22-35,594-606` |
| Enable tests on a site | If `run-tests` refuses to run, `bench --site <site> set-config allow_tests true` (standard Frappe behaviour; **Unverified** for your Frappe version) | — |
| Build the frontend | `cd desk && yarn build` (or `bench build --app helpdesk`) | `desk/package.json`; `README.md` |
| Python lint/format | `pre-commit run --all-files` runs black 22.6.0, flake8 5.0.4 (with `.github/helpers/.flake8_strict` and bugbear, comprehensions, and simplify plugins), isort (black profile), and prettier 2.6.2 for JS, Vue, and HTML | `.pre-commit-config.yaml` |
| Extra CI lint | Frappe semgrep rules plus `r/python.lang.correctness`, and commitlint on PR commit titles | `.github/workflows/lint.yml` |
| Frontend lint | Root `.eslintrc.js` (vue3-recommended, TypeScript, Tailwind, Prettier) with `.lintstagedrc` running `eslint --fix`. Neither `package.json` declares ESLint or lint-staged as a dependency, so how these are installed is **Unverified**. | `.eslintrc.js`; `.lintstagedrc`; `package.json`; `desk/package.json` |
| Frontend tests | None configured: `desk/package.json` has no test script and no test runner dependency | `desk/package.json` |
| Debug script | Write `helpdesk/debug.py` with `def execute():`, then run `bench --site <site> execute helpdesk.debug.execute` | `AGENTS.md` "Debugging Workflow" |
| Translations | `bench generate-pot-file --app helpdesk` regenerates `helpdesk/locale/main.pot` (CI does this weekly) | `.github/helpers/update_pot_file.sh`; `.github/workflows/generate-pot-file.yml` |

### 2.4 Branching, CI, and release (upstream conventions)

- **Branches.** Upstream develops on `develop` and ships from `main` (stable, where Mergify auto-closes outside PRs) and `main-hotfix`. Backports are applied through labels (`.mergify.yml`). WBH work in this fork happens on feature branches such as `WBH-9-2026`.
- **Commits.** Use Conventional Commits with the types `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, `test` (`commitlint.config.mjs`). CI enforces this on PRs to `main`, `main-hotfix`, and `develop` (`.github/workflows/lint.yml`).
- **CI on PRs.** Server tests run on MariaDB 10.6 with Frappe `version-15` and ERPNext `version-15` (`.github/workflows/server-tests.yml`, `.github/helpers/install.sh`). Changes that touch only `.js`, `.css`, `.md`, `.html`, or `.csv` files skip the server tests (`paths-ignore` in `server-tests.yml`). `.vue` and `.ts` files are **not** in that list, so frontend-only changes still trigger them.
- **Release.** semantic-release runs on `main` and rewrites the version in `helpdesk/__init__.py` (`.releaserc`, `.github/workflows/on_release.yml`). A container image is built on pushes to `main` and on tags (`.github/workflows/build.yml`).

## 3. Folder and module map

_Section in progress._

## 4. Architecture and data flow

_Section in progress._

## 5. Where to start, and the conventions to follow

_Section in progress._

## 6. Known issues, TODOs, and fragile areas

_Section in progress._

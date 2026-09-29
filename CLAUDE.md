# CLAUDE.md — Frappe Helpdesk (WBH fork)

Frappe Helpdesk is a ticketing and customer-service app with a Frappe (Python) backend in `helpdesk/` and a Vue 3 SPA in `desk/`, served at `/helpdesk`. It requires the `telephony` app. ERPNext is optional.

- **Binding coding rules:** `AGENTS.md`. Follow them.
- **Full developer reference:** `docs/WBH-PROJECT_GUIDE.md`, covering the module map, data model, ticket lifecycle, permissions, recipes, and known issues. Read the relevant section on demand; do not load the whole file.

## Layout

```
helpdesk/                      Frappe app (Python package)
  hooks.py                     wiring: scheduler, doc_events, permission hooks, overrides, routes
  helpdesk/doctype/<dt>/       36 DocTypes: <dt>.json schema, <dt>.py controller, test_<dt>.py
  helpdesk/doctype/hd_ticket/  core model; api.py holds new/get_one/merge/split endpoints
  api/                         @frappe.whitelist endpoints for the SPA (doc.py = generic list engine)
  utils.py                     is_agent / agent_only / agent_manager_only / publish_event
  overrides/  extends/         core-doctype class overrides / doc_events function hooks
  integrations/erpnext/        optional HD Customer <-> ERPNext Customer sync
  search.py / search_sqlite.py KB search (Redisearch) / ticket search (SQLite FTS)
  setup/install.py             after_install seeding; patches/ + patches.txt = migrations
  test_utils.py                ALL shared test helpers live here
  www/helpdesk/index.py        serves the SPA shell + boot data
desk/                          Vue 3 + Vite SPA (built into helpdesk/public/desk)
  src/main.js, src/router/index.ts, src/roots/   bootstrap, routes/guards, agent vs customer shell
  src/pages/  src/components/  src/stores/  src/composables/
  patches/                     patch-package patch for frappe-ui 1.0.0-beta.24
realtime/handlers.js           Socket.IO handlers (ticket viewers, typing)
```

## Commands

Run these from a Frappe bench (`frappe-bench/`). Replace `<site>` with your site name, for example `helpdesk.test`.

```bash
# Install into a site (runs helpdesk.setup.install.after_install)
bench get-app telephony && bench get-app <this-repo-url-or-path>
bench --site <site> install-app helpdesk
bench start                                   # web :8000, workers, scheduler, socketio

# Frontend (from apps/helpdesk/desk)
yarn install                                  # postinstall applies desk/patches via patch-package
yarn dev                                      # Vite dev server, default :8080, proxies API to bench
yarn build                                    # or: bench build --app helpdesk

# Backend tests (CI installs erpnext + telephony + helpdesk into the test site)
bench --site <site> run-tests --app helpdesk
bench --site <site> run-tests --module helpdesk.helpdesk.doctype.hd_ticket.test_hd_ticket
# if refused: bench --site <site> set-config allow_tests true

# Lint / format (black, isort, flake8 strict config, prettier)
pre-commit run --all-files

# Migrations and debugging
bench --site <site> migrate                   # runs helpdesk/patches.txt
bench --site <site> execute helpdesk.debug.execute   # after writing helpdesk/debug.py with def execute()

# Translations
bench generate-pot-file --app helpdesk        # updates helpdesk/locale/main.pot
```

- There are **no** frontend tests, and nothing runs lint or typecheck scripts in `desk/package.json`.
- `desk` builds must run inside a bench:
  - `desk/src/socket.ts` imports `sites/common_site_config.json`.
  - `@framework/ui` links to `../../frappe/ui`.
- `docker/init.sh` installs helpdesk from upstream GitHub (`--branch main`), **not** from this checkout.

## Backend conventions

- **Controller hooks only call methods.** Keep the logic in methods on the class. Mutate everything, then call `.save()` once, in the hook or in the method that owns the mutation. Never save inside a helper. Never fetch the same doc twice in one hook.
- **Whitelisted functions must type-annotate their parameters** (`require_type_annotated_api_methods = True` in `hooks.py`).
- **Guard agent-only endpoints** with `@agent_only` or `@agent_manager_only` from `helpdesk/utils.py`.
- **Prefer `frappe.qb.get_query(..., ignore_permissions=False)`.** Plain `frappe.qb` and `frappe.get_all` **skip** the HD Ticket `permission_query` and team restrictions.
- **"Admin" means three different things:**
  - `utils.is_admin()` is true only for the literal `Administrator` user.
  - `api.auth.get_user().is_admin` is true for System Managers.
  - `is_agent_manager()` accepts Agent Manager, System Manager, or Administrator.

  Pick the one you mean deliberately.
- **Realtime:** use `publish_event(event, room=get_doc_room(doctype, name), data=...)`. It publishes after commit.
- **New defaults** go in `setup/install.py` (new sites) **and** a patch in `patches.txt` (existing sites). Patches must be idempotent.
- **Tests:**
  - Use `FrappeTestCase`.
  - Put every helper in `helpdesk/test_utils.py`, with a docstring: `make_<doctype>` for factories, `enable_<x>`/`disable_<x>` for toggles.
  - Check `test_utils.py` for an existing helper before writing one.
- **Translations:** use `_()`. Never wrap an f-string in `_()`; use `_("... {0}").format(x)`.
- **Spec docs:** if a feature has one, update it in the same change (AGENTS.md rule 2).

## Frontend conventions

- **Components:** `<script setup lang="ts">`, Composition API, PascalCase files.
- **Data fetching:** frappe-ui `createResource`, `createListResource`, `createDocumentResource`, with `url: "helpdesk.api.<module>.<fn>"`. Never use `useFetch`.
- **Icons:** Lucide only (`<LucideX />` auto-registered, or `import LucideX from "~icons/lucide/x"`). Do not add new `FeatherIcon` uses.
- **Styling:** Tailwind semantic tokens (`text-ink-gray-*`, `bg-surface-*`, `border-outline-gray-*`, `text-p-*`), gray shades only.
- **i18n:** every user-visible string goes through `__()`. Do not pass template literals to `__()`.
- **State:** Pinia for shared state; `ref`/`computed` locally; `@vueuse/core` before custom helpers.
- **Reuse before building:**
  - `ListViewBuilder.vue` (list pages)
  - `layouts/SettingsLayoutBase.vue` (settings tabs)
  - `LayoutHeader.vue`, `Section.vue`, `EmptyState.vue`, `ConfirmDialog.vue`
  - `TicketField.vue`
  - `editor/config.ts` (editors)
- **Routes:** customer-portal routes need `meta: { public: true }`. Routing is not security; enforce access on the server.
- **Socket listeners:** remove them with the same handler (`$socket.off(event, handler)`). A bare `off(event)` removes everyone's listeners.

## Workflow rules

- **Never write to a Frappe site** (insert, update, delete, migrate) unless the user explicitly asks. Read-only queries are fine (AGENTS.md rule 1).
- **Commits:** Conventional Commits (`feat:`, `fix:`, `docs:`, …), enforced by commitlint in CI.
- **Where work goes:** WBH work goes on `WBH-*` branches. Upstream accepts PRs only on `develop`.
- **Only comments that explain *why*.**
- **This file is listed in `.gitignore`.** Commit changes to it with `git add -f CLAUDE.md`. Once tracked, later edits show up normally.

## Known drift between AGENTS.md and the code

- **TypeScript strictness.** AGENTS.md says TS is "not strict". `desk/tsconfig.json` extends the frappe-ui base and adds strict flags, but nothing runs `tsc`.
- **`desk/components.d.ts`** is generated and gitignored, so it is absent from the repo.
- **`helpdesk.api.ticket.assign_ticket_to_agent`** (the AGENTS.md example) is **not** whitelisted. Use `HDTicket.assign_agent`.
- **`docs/erpnext-customer-sync.md`**, cited by AGENTS.md as a spec, does not exist.

Details and the full list of known bugs and fragile areas are in `docs/WBH-PROJECT_GUIDE.md` §6.

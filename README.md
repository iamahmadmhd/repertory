# Repertory

A collaborative movie discovery and watchlist app. Authenticated members browse and search films (sourced
from [TMDB](https://www.themoviedb.org/)), save titles privately, and build watchlists that
other members can collaborate on in real time — with per-list Owner / Editor / Viewer roles
and changes syncing live across everyone viewing the same list.

Built serverless on React 19 + TypeScript and AWS Amplify Gen 2. See [Tech stack](#tech-stack)
below for the full list.

## Screens

### Real-time collaboration

A film added by one member appears for everyone with the list open — an AppSync subscription scoped to that single watchlist, not a global firehose. Writes reconcile by server id, so your own optimistic insert is replaced rather than duplicated when its event echoes back. (FR-SYNC-1, FR-SYNC-2)
![Two browser windows side by side, signed in as different members. A film added to a watchlist in the left window appears in the right window's list without a page reload.](https://github.com/user-attachments/assets/99897d78-cd89-4917-a277-245520aa3e36)

### Watchlists with per-list roles

Owner, Editor and Viewer are enforced server-side — the badges here are presentation only; AppSync rejects the operations regardless of what the client renders. Member colours are derived from a hash of the user id, so nobody has to assign them. (FR-LIST-5, FR-MEM-3, NFR-SEC-1)
![A watchlist detail screen. Three members are listed with coloured avatar initials and role badges reading OWNER, EDITOR and VIEWER.](https://github.com/user-attachments/assets/d2866446-2bee-4816-a8e9-5715fd3c29c5)

## Documents (source of truth)

- [`docs/SRS.md`](docs/SRS.md) — what the system must do (requirement IDs: `FR-*`, `NFR-*`, `V-*`)
- [`docs/SYSTEM-DESIGN.md`](docs/SYSTEM-DESIGN.md) — how it's built (stack, module structure, ADRs)
- [`CLAUDE.md`](CLAUDE.md) — working rules distilled from the above, for AI-assisted development

If you're changing behavior, check the SRS/System Design first — they're the spec this app is
built against, and the rest of this README just summarizes them for a quick start.

## Getting started

Requires [pnpm](https://pnpm.io/) and AWS credentials for the sandbox backend.

```bash
pnpm install
pnpm sandbox        # starts a local Amplify backend sandbox (needs AWS credentials)
pnpm dev             # in a second terminal
```

You'll need a TMDB credential before `tmdb-proxy` works. Get a **Read Access Token** (the
bearer token, not the legacy v3 API key) from <https://www.themoviedb.org/settings/api> and
store it via `npx ampx sandbox secret set TMDB_ACCESS_TOKEN` — SSM-backed, never in `.env`
(FR-TMDB-1).

## Scripts

| Command                | What it does                                                    |
| ---------------------- | --------------------------------------------------------------- |
| `pnpm dev`             | Vite dev server                                                 |
| `pnpm sandbox`         | Local Amplify backend sandbox (needs AWS credentials)           |
| `pnpm build`           | Production build                                                |
| `pnpm build:analyze`   | Production build + bundle visualizer report (`dist/stats.html`) |
| `pnpm test`            | Unit/integration tests (Vitest)                                 |
| `pnpm test:e2e`        | End-to-end tests (Playwright)                                   |
| `pnpm lint`            | ESLint, including the Feature-Sliced Design boundaries check    |
| `pnpm format`          | Prettier write + ESLint fix                                     |
| `pnpm check`           | Prettier check only (no writes)                                 |
| `pnpm generate-routes` | Regenerate `src/routeTree.gen.ts` outside of dev/build          |

## Tech stack

React 19 + TypeScript (strict) on Vite, routed with TanStack Router and cached with TanStack
Query, styled with Tailwind CSS v4 over Base UI primitives. Forms run on react-hook-form + Zod.
The backend is AWS Amplify Gen 2 (Cognito auth, AppSync/DynamoDB data, a handful of Lambda
functions for the operations generated resolvers can't express). Full rationale for each
choice, including what was deliberately left out and why, is in
[`docs/SYSTEM-DESIGN.md`](docs/SYSTEM-DESIGN.md) §2.1 and its ADRs.

## Project structure

Feature-Sliced Design: `app > pages > features > entities > shared`, each layer importing
only from layers strictly below it (enforced by `eslint-plugin-boundaries`, not convention).

```text
src/
  app/        providers, router (file-based routes), layouts, global styles
  pages/      discover, search, movie-detail, saved, watchlists, watchlist-detail, settings, auth
  features/   save-movie, manage-list-items, manage-members, toggle-watched,
              create-watchlist, filter-discovery, claim-username, edit-profile,
              delete-account, sign-out
  entities/   movie, watchlist, member (each with ui/ model/ api/)
  shared/     ui (Base UI wrappers), lib, config

amplify/
  auth/       Cognito user pool + triggers
  data/       schema, auth rules, custom operations
  functions/  post-confirmation, tmdb-proxy, membership, claim-username, permission-fanout,
              watchlist-item, delete-account, image-proxy
```

See [`CLAUDE.md`](CLAUDE.md) for the full naming and layering rules, including the one
framework-mandated exception (`src/app/router/`, where TanStack Router owns filenames).

## Status

The application is built and functional end to end: registration and passwordless sign-in,
discovery, search, movie detail, saved films, watchlists with collaborators and per-list
roles, watched tracking, real-time sync on an open list, theming, and account deletion.
All eight backend functions are implemented.

Two gaps are worth knowing before picking something up:

- **The authorization test matrix is not filled in.** `tests/auth-matrix/watchlist-permissions.test.ts`
  encodes SRS §6.1 as `it.todo(...)` cells. Authorization is this project's principal
  correctness claim, so this is the largest outstanding piece of work. Each cell must be
  asserted against the API, never the UI.
- **Three specified features are not built**: renaming a list (FR-LIST-2/3), drag
  reordering items (FR-ITEM-5), and the avatar half of FR-AUTH-5.

System Design §11 carries the full status table, including the screen elements that are
deliberately absent because no requirement or data backs them.

## Conventions

Two rules the codebase enforces rather than suggests:

- **Layering.** `app > pages > features > entities > shared`, checked by
  `eslint-plugin-boundaries`. A slice never imports a sibling in the same layer.
- **Comments explain mechanism; documents record decisions.** Rationale, alternatives
  considered, and requirement traceability belong in `docs/`, not in source comments. A
  comment in the code should say what a non-obvious line does or why it is load-bearing,
  and stop there.

## Constraints

Built to a six-week budget (System Design C-5). The "Known Limitations" (§9) and "Out of
Scope" (SRS §7) sections exist because of it — check those two lists before adding scope.

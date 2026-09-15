# Repertory

[![Live app](https://img.shields.io/badge/live-repertory-0ea5e9?style=flat-square)](https://www.repertory.iamahmadmhd.com)
[![React 19](https://img.shields.io/badge/React-19-61dafb?style=flat-square&logo=react&logoColor=black)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![AWS Amplify](https://img.shields.io/badge/AWS-Amplify_Gen_2-ff9900?style=flat-square&logo=awsamplify&logoColor=white)](https://docs.amplify.aws)

A collaborative movie discovery and watchlist app. Authenticated members browse and search films (sourced
from [TMDB](https://www.themoviedb.org/)), save titles privately, and build watchlists that
other members can collaborate on in real time - with per-list Owner / Editor / Viewer roles
and changes syncing live across everyone viewing the same list.

Built serverless on React 19 + TypeScript and AWS Amplify Gen 2. See [Tech stack](#tech-stack)
below for the full list.

## Screens

### Real-time collaboration

A film added by one member appears for everyone with the list open - an AppSync subscription scoped to that single watchlist, not a global firehose. Writes reconcile by server id, so your own optimistic insert is replaced rather than duplicated when its event echoes back. (FR-SYNC-1, FR-SYNC-2)
![Two browser windows side by side, signed in as different members. A film added to a watchlist in the left window appears in the right window's list without a page reload.](https://github.com/user-attachments/assets/99897d78-cd89-4917-a277-245520aa3e36)

### Watchlists with per-list roles

Owner, Editor and Viewer are enforced server-side - the badges here are presentation only; AppSync rejects the operations regardless of what the client renders. Member colours are derived from a hash of the user id, so nobody has to assign them. (FR-LIST-5, FR-MEM-3, NFR-SEC-1)
![A watchlist detail screen. Three members are listed with coloured avatar initials and role badges reading OWNER, EDITOR and VIEWER.](https://github.com/user-attachments/assets/d2866446-2bee-4816-a8e9-5715fd3c29c5)

### Discovery

TMDB-sourced, but every request is issued server-side through a caching Lambda - the API credential never reaches the browser, and responses are normalised into application types before they do. Genre and page live in the URL as Zod-validated search params, so any view is linkable. (FR-DISC-1, FR-DISC-3, FR-TMDB-1, FR-TMDB-3)
![A grid of film posters under the heading "Trending this week", with a genre filter dropdown open and pagination controls below the grid.](https://github.com/user-attachments/assets/c4cde20f-2b9c-41d9-8c7d-3e6a0d221782)

### Saved films are private

Not a watchlist with one member - a separate model with no read path for anyone else, and no coupling to lists in either direction. Saving a film adds it to nothing. (FR-SAVE-3, FR-SAVE-4, FR-SAVE-5)
![A grid of saved film posters under the heading "Saved films", with the subheading "12 films · private to you".](https://github.com/user-attachments/assets/a33e6269-e14b-48a0-945a-0d44d72d0c78)

### Passwordless by design

One screen for both sign-up and sign-in: with no password, signing in is just proving you own the email again, so the API response decides which path you're on - not the visitor picking a tab. Nothing to store, nothing to leak, no reset flow to build. (FR-AUTH-1, FR-AUTH-6, ADR-011/012)
![A sign-in screen with a single email address field and a continue button. There is no password field and no "forgot password" link.](https://github.com/user-attachments/assets/4724ea66-3964-4bd4-bc04-87d88894f407)

### Settings

Account lifecycle, finished. Usernames are claimed atomically - there is no window in which two members hold the same one. Deleting an account cascades across seven tables and every step is idempotent, so a partial failure is safe to retry. (FR-AUTH-3, FR-AUTH-4, FR-LIST-4)
![A settings screen with a light/dark theme toggle, a claimed username shown as @handle, sign out and sign out everywhere actions, and a delete account action in red.](https://github.com/user-attachments/assets/1b95de82-7e68-468d-b812-6872b1239a6a)

### Mobile Watchlist

Built for both, not reflowed into one. Desktop and mobile are separate trees behind a single breakpoint - a sidebar and dense grid on one, a bottom tab bar and stacked rows on the other - because collapsing a six-column poster grid into a phone never produces the layout you'd have designed for it.
![The same watchlist on a narrow phone-width screen, with a bottom tab bar showing Discover, Saved, Lists and Settings.](https://github.com/user-attachments/assets/cd93d5a7-fc24-4578-93bf-d5b4f325b54f)

### Watchlist detail - light

One token system, two themes. Colour is defined once as custom properties and resolved per scheme with CSS light-dark() - including the generated member colours, which are OKLCH pairs rather than two hand-maintained palettes. No component knows which theme is active.
![The watchlist detail screen from image 2, rendered in light mode with the same content and layout.](https://github.com/user-attachments/assets/7024db52-0309-401a-b2d3-e9b1f3d346f7)

## Documents (source of truth)

- [`docs/SRS.md`](docs/SRS.md) - what the system must do (requirement IDs: `FR-*`, `NFR-*`, `V-*`)
- [`docs/SYSTEM-DESIGN.md`](docs/SYSTEM-DESIGN.md) - how it's built (stack, module structure, ADRs)
- [`CLAUDE.md`](CLAUDE.md) - working rules distilled from the above, for AI-assisted development

If you're changing behavior, check the SRS/System Design first - they're the spec this app is
built against, and the rest of this README just summarizes them for a quick start.

## Getting started

**Try it: [https://www.repertory.iamahmadmhd.com](https://www.repertory.iamahmadmhd.com/)** — sign in with any email address; a one-time
code arrives in a few seconds. There is no password.

To run it locally against your own backend:
Requires [pnpm](https://pnpm.io/) and AWS credentials for the sandbox backend.

```bash
pnpm install
pnpm sandbox        # starts a local Amplify backend sandbox (needs AWS credentials)
pnpm dev             # in a second terminal
```

You'll need a TMDB credential before `tmdb-proxy` works. Get a **Read Access Token** (the
bearer token, not the legacy v3 API key) from <https://www.themoviedb.org/settings/api> and
store it via `npx ampx sandbox secret set TMDB_ACCESS_TOKEN` - SSM-backed, never in `.env`
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
Scope" (SRS §7) sections exist because of it - check those two lists before adding scope.

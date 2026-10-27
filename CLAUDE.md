# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack and layout

- **Frontend:** Next.js (App Router) with TanStack Query, in `client/` (planned, not created yet).
- **Backend:** NestJS 12 in `server/`, the only code so far.

One git repo at the root covers both apps, with a single root `.gitignore`. `server/docs/` is deliberately ignored. Run the commands below from `server/`.

### Frontend–backend contract (planned)

- Next rewrites `/api/:path*` to Nest, and Nest uses `app.setGlobalPrefix('api')`. The browser only ever talks to the Next origin, so the auth cookies are first-party. The Google callback URL points at the Next origin.
- Public pages (tutor listings and profiles) are server components that fetch public data without cookies.
- Logged-in pages are client components that use TanStack Query and a shared `api()` fetch helper (`credentials: 'include'`, retries once after `POST /api/auth/refresh` on a 401). Don't gate auth in `proxy.ts`/middleware: the `refresh_token` cookie is scoped to `/api/auth`, so the Next server never sees it on page requests.

## Commands

```bash
npm run start:dev        # watch mode on PORT (default 3000)
npm run build            # nest build -> dist/ (output dir is wiped on each build)
npm run lint             # oxlint --type-aware over src/ and test/ (not ESLint)
npm run format           # prettier: single quotes, trailing commas

npm test                                   # unit tests: *.spec.ts under src/
npm test -- src/app.controller.spec.ts     # a single test file
npm test -- -t 'should return'             # tests matching a name
npm run test:e2e                           # *.e2e-spec.ts under test/ (config: test/jest-e2e.json)
```

The Jest scripts run through `node --experimental-vm-modules`, so call them via `npm test`, not by running `npx jest` directly. Unit config is in `jest.config.ts`, which reads `paths` aliases from `tsconfig.json`. Add new aliases in tsconfig only, not in the Jest config.

## Conventions

- TypeScript uses `module`/`moduleResolution: nodenext` with `strict` on (`strictPropertyInitialization` off). Check that new dependencies, especially generated clients like Prisma, import cleanly under nodenext.
- oxlint treats `no-floating-promises` as an error. Mark intentionally unawaited promises with `void`, as `main.ts` does with `void bootstrap()`.

## Current state and planned direction

`src/` still holds only the Nest starter (`AppModule`, `AppController`, `AppService`). There is no database, config module or auth yet.

`server/docs/oauth-login-plan.md` is the agreed design for authentication. Read it before you work on auth, users or persistence. Its main decisions:
- Prisma + Postgres with `User`, `Account` and `RefreshToken` models, and a global `PrismaService`.
- Google OAuth through the server-side authorization code flow (`passport-google-oauth20`).
- A 15-minute access JWT and a 30-day opaque refresh token, both in `httpOnly` cookies and never in `localStorage`. Refresh tokens are stored as hashes, rotated on every use, and a reused token revokes its whole family.
- A global JWT guard with a `@Public()` opt-out, plus `@Roles()` and `@CurrentUser()` decorators.
- `role` (`STUDENT` | `TUTOR`) starts as null and is set once during onboarding. `ADMIN` is never self-assigned.
- CORS and an `Origin` check are locked to `FRONTEND_URL`.

The plan refers to a placeholder in-memory `users` feature that does not exist in `src/` right now.

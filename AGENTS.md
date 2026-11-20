# AGENTS.md

Instructions for coding agents working in this repository. Humans: start with `README.md`.

## What this is

A small slice of **LeeTKOB** (CSC13114 teaching project): a TA's review queue and the
*Publish grades* endpoint. It is a Session 7 exercise about making a CI gate that actually
fails. Some of what you find here is broken **on purpose**; the student fixes it step by step.
Do only the step you are asked to do.

## Setup and commands

Node **22.13+** (uses the built-in `node:sqlite`). No framework, no build step, ES modules.

```bash
npm ci                  # install exactly what the lockfile says; never `npm install`
npm start               # http://localhost:3007, in-memory DB reseeded on every start
npm run lint            # ESLint
npm run format:check    # Prettier (singleQuote, printWidth 100; *.md is ignored)
npm test                # unit tests (node --test), test/unit/
npm run contract        # contract tests, test/contract/
npm run check           # lint + format + unit + contract: this is the gate
npm run e2e             # Playwright, needs `npx playwright install chromium`
```

Run `npm run check` before you say a change is done, and report its real output. If it is
red, say so; do not describe red as green.

## Layout

| Path | Role |
|---|---|
| `docs/contract/publish.md` | the contract for `POST /api/labs/{labId}/publish`. **It is correct.** |
| `src/app.js` | all routes and status codes, plain `node:http` |
| `src/grades.js` | pure logic, no DB or HTTP; unit tested |
| `src/http.js` | `send`, `problem` (RFC 9457 problem details), `readJson` |
| `src/db.js` | schema and seed data (users, tokens, classes, labs, submissions) |
| `test/helpers.js` | `startApp()`: in-process server on a fresh DB, `as(user)`, `publishedAt(lab)` |
| `test/unit/`, `test/contract/`, `e2e/` | the three layers of the test pyramid |
| `.github/workflows/ci.yml` | the CI gate; job name `gates` |

Seed tokens are `tok-<name>` with `_` as `-` (e.g. `ta_a` → `tok-ta-a`). See the comment at
the top of `src/db.js` for which lab can and cannot be published.

## Rules

1. **The contract wins.** When code and `docs/contract/publish.md` disagree, fix `src/`.
   Never edit the contract to match the code.
2. **Never weaken a test to get green.** Do not delete or loosen assertions, add `skip`/`todo`,
   wrap assertions in `try/catch`, or lower the expected count. Making a test *stricter* is
   fine. Removing a `try/catch` that swallows failures is fine.
3. **Nothing may swallow a failure in CI.** No `|| true`, no `continue-on-error`, no
   `if: always()` that hides a failing step. The workflow must run `npm run check`, on every
   pull request, with the Node version from `package.json` `engines`, using `npm ci`.
4. **Stay inside the step's scope.** If the brief says "do not touch `src/` or `test/`", don't,
   even to fix something you noticed. Mention it instead.
5. **A refused call changes nothing.** Any `4xx` from publish must leave `published_at` as it
   was. Authorization and the pending-review check come before any write.
6. **Errors are problem details**: `application/problem+json` with `type`, `title`, `status`,
   built with `problem()` in `src/http.js`. Do not hand-roll error bodies.
7. **Keep `src/grades.js` pure.** No imports from `db.js` or `http.js`.
8. **No new dependencies** unless asked. The no-framework design is deliberate: every status
   code should be visible in `src/app.js`.

## Tests

- Contract tests use `startApp()` and must assert on status, `content-type`, problem `type`,
  and, for refusals, on `app.publishedAt(lab)` read **after** the call.
- Each test gets a fresh database (`beforeEach`), so tests must not depend on order.
- The e2e suite is one journey; keep it to one or two. Prefer contract tests for rules.

## Commits and pull requests

- Small commits, one concern each. Test changes go in their own commit, separate from
  `src/` fixes, so `git diff -- test/` can be reviewed on its own.
- Never push to `main`; open a pull request. `main` is protected and requires `gates`.
- In the PR description, say which checks you ran and paste the failing line if any failed.

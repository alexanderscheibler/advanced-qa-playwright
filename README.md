# Advanced QA Framework for Playwright E2E

A Claude skill that acts as a senior QA - reviewing test files, identifying gaps, and writing production-quality Playwright tests using a consistent, opinionated architecture.

Built from real knowledge. Tested against JavaScript and TypeScript specs of varying quality, then refined based on what it consistently missed.

Made for small projects that need a e2e test suite implemented quickly.

---

## What it does

It works two ways.

### Review Playwright specs
Hand it an existing Playwright spec and it gives you a prioritised gap analysis before touching any code. Then, if you want, it writes the fixes. This is the default, and it stays in lane: it won't rewrite your whole framework when you asked about one file, and it won't invent endpoints or schemas you haven't shown it.

### Build a framework from scratch
Or start from zero. Ask it to set up a framework for a small project. It scaffolds the real thing - config, folder structure, Page Objects, fixtures, a first working spec, and a GitHub Actions workflow - right-sized to what the project actually needs.

Both modes intend to cover the full QA surface: UI flows, API testing, database verification, CI/CD, and flaky test diagnosis.

---

## For developers and SDETs

The skill enforces a layered architecture that keeps tests maintainable as codebases grow:

- **Page Objects** own locators and UI interactions — no assertions inside them
- **Fixtures** inject dependencies — no `new SomePage(page)` scattered through tests
- **Specs** own all assertions, wrapped in `test.step()` for readable reports
- **`data/`** holds typed scenario files — no hardcoded test data in specs

If Playwright MCP is connected, it can use `browser_generate_locator` to resolve locators directly from the live accessibility tree.

---

## For QA

The skill follows a four-stage process on every review, even when you ask for "a quick look":

1. **Collect requirements** — it asks what the feature is actually supposed to do before writing anything
2. **Map the test space** — happy path and unhappy path, UI and data integrity, across a consistent set of commonly missed areas (boundary values, state transitions, permission gaps, concurrent operations)
3. **Implement** — gap analysis first, code second
4. **Evaluate** — does each test catch a silent regression, or does it just pass?

The review checklist covers coverage gaps, assertion quality, locator reliability, and data completeness. For any data-changing operation, it can look for DB verification before and after - not just UI.

---

## Stack

TypeScript · Playwright · Page Object Model · GitHub Actions · PostgREST (DB verification) · Playwright MCP (optional, for locator resolution)

---

## Version

`1.0.0` · MIT · [alexanderscheibler](https://github.com/alexanderscheibler)
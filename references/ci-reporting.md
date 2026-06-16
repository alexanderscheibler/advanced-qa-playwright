# CI/CD and Reporting Reference

## GitHub Actions — Sharded E2E

Runs on every push to `main` / `feature/**` and on every PR. Tests are sharded across parallel jobs; HTML reports are merged at the end.

```yaml
# .github/workflows/e2e.yml
name: E2E Regression Suite

on:
  push:
    branches: [main, 'feature/**']
  pull_request:
    branches: [main]

jobs:
  test:
    name: 'Playwright (shard ${{ matrix.shard }}/4)'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Cache Playwright browsers
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          restore-keys: playwright-${{ runner.os }}-

      - run: npx playwright install --with-deps chromium

      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: npx playwright test --shard=${{ matrix.shard }}/4
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_USERNAME: ${{ secrets.TEST_USERNAME }}
          TEST_PASSWORD: ${{ secrets.TEST_PASSWORD }}
          CI: 'true'

      - name: Upload blob report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report/
          retention-days: 1

      - name: Upload JUnit results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: junit-results-${{ matrix.shard }}
          path: test-results/junit.xml
          retention-days: 30

      - name: Upload failure traces
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: traces-shard-${{ matrix.shard }}
          path: test-results/
          retention-days: 30

  merge-reports:
    name: Merge Playwright Reports
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Download all blob reports
        uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true

      - name: Merge into single HTML report
        run: npx playwright merge-reports --reporter html ./all-blob-reports

      - name: Upload merged HTML report
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

---

## Reporter Configuration

In `playwright.config.ts`, use blob reporter for CI (enables sharded merging) and HTML locally:

```typescript
reporter: process.env.CI
  ? [
      ['blob'],
      ['list'],
      ['junit', { outputFile: 'test-results/junit.xml' }],
    ]
  : [
      [
        'html',
        {
          outputFolder: "playwright-report",
          open: process.env.CI ? "never" : "on-failure",
        }, 
      ],
      ['list'],
    ],
```

---

## Docker (local or self-hosted CI)

```dockerfile
FROM mcr.microsoft.com/playwright:v1.48.0-noble
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npx", "playwright", "test"]
```

```yaml
# docker-compose.yml for local full-stack testing
services:
  playwright:
    build: .
    environment:
      BASE_URL: http://app:3000
      TEST_USERNAME: testuser
      TEST_PASSWORD: testpass123
    depends_on: [app, db, postgrest]

  postgrest:
    image: postgrest/postgrest:v14
    ports: ['3001:3000']
    environment:
      PGRST_DB_URI: postgres://user:password@db:5432/mydb
      PGRST_DB_SCHEMA: public
      PGRST_DB_ANON_ROLE: readonly_role
    depends_on: [db]
```

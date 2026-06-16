# Architecture Reference

Full code patterns for the Playwright E2E framework layers.

## Table of Contents
1. [Configuration](#configuration)
2. [Pages Layer](#pages-layer)
3. [Interaction Resilience](#interaction-resilience)
4. [Utils — API](#utils--api)
5. [Utils — Database](#utils--database)
6. [Fixtures](#fixtures)
7. [Data-Driven Tests](#data-driven-tests)
8. [Spec Example with DB Verification](#spec-example-with-db-verification)

---

## Configuration

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

import dotenv from 'dotenv';
import path from 'path';
dotenv.config({
  quiet: true,
  path: path.resolve(__dirname, '.env'),
});

const BASE_URL = process.env.BASE_URL ?? 'http://localhost';

export default defineConfig({
  testDir: './tests',
  expect: { timeout: 10_000 },
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
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
  use: {
    baseURL: BASE_URL,
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'retain-on-failure',
    navigationTimeout: 60_000,
    actionTimeout: 30_000,
  },
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: 'fixtures/auth/storageState.json' },
      dependencies: ['setup'],
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'], storageState: 'fixtures/auth/storageState.json' },
      dependencies: ['setup'],
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'], storageState: 'fixtures/auth/storageState.json' },
      dependencies: ['setup'],
    },
  ],
});
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "rootDir": ".",
    "baseUrl": ".",
    "paths": {
      "@pages/*": ["tests/pages/*"],
      "@utils/*": ["tests/utils/*"],
      "@fixtures": ["tests/fixtures/*"]
    }
  },
  "include": ["tests/**/*", "utils/**/*", "playwright.config.ts"],
  "exclude": ["node_modules", "dist", "playwright-report", "test-results"]
}
```

### utils/config.ts

```typescript
export interface AppConfig {
  baseUrl: string;
  credentials: { username: string; password: string };
  api: { timeout: number };
}

export const config: AppConfig = {
  baseUrl: (process.env.BASE_URL ?? 'http://localhost:3000').replace(/\/+$/, ''),
  credentials: {
    username: process.env.TEST_USERNAME ?? 'testuser',
    password: process.env.TEST_PASSWORD ?? 'testpass123',
  },
  api: { timeout: Number(process.env.API_TIMEOUT) || 30_000 },
};
```

**.gitignore must include:**
```
tests/fixtures/auth/storageState.json
playwright-report/
test-results/
blob-report/
.env
```

---

## Pages Layer

Page classes own locators and UI interactions. They **never** contain `expect()`.

### BasePage

```typescript
// pages/BasePage.ts
import { type Page } from '@playwright/test';

export class BasePage {
  readonly page: Page;
  constructor(page: Page) { this.page = page; }
  async navigate(path: string): Promise<void> { await this.page.goto(path); }
}
```

### LoginPage

```typescript
// pages/LoginPage.ts
import { type Page } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
  constructor(page: Page) { super(page); }

  // Arrow functions ensure fresh locator evaluation on every call
  emailInput    = () => this.page.getByLabel('Email');
  passwordInput = () => this.page.getByLabel('Password');
  submitBtn     = () => this.page.getByRole('button', { name: 'Log in' });
  errorMessage  = () => this.page.locator('.error-message');

  async goto(): Promise<void> { await this.page.goto('/login'); }

  async login(username: string, password: string): Promise<void> {
    await this.emailInput().fill(username);
    await this.passwordInput().fill(password);
    await this.submitBtn().click();
    await this.page.waitForURL('**/dashboard**', { timeout: 30_000 });
  }

  /** Returns error text so the spec can assert on it */
  async getErrorText(): Promise<string> {
    return this.errorMessage().textContent() ?? '';
  }
}
```

### Barrel Export

```typescript
// pages/index.ts
export { BasePage } from './BasePage';
export { LoginPage } from './LoginPage';
```

---

## Interaction Resilience

For complex UIs (autocompletes, async re-renders) where a just-opened dropdown can be torn down before the click lands. This is **not** a workaround for flakiness — it's an atomic retry for inherently racy interactions.

```typescript
// Inside a Page class

/**
 * Retry a UI interaction until it succeeds or the timeout elapses.
 * Use when an async re-render can invalidate a just-opened dropdown.
 * Each attempt uses Playwright's own action-level auto-wait — no sleep inside.
 */
private async retryInteraction(
  action: () => Promise<void>,
  timeoutMs = 15_000,
): Promise<void> {
  const deadline = Date.now() + timeoutMs;
  let lastError: unknown;
for (;;) {
  try { await action(); return; }
  catch (error) {
    lastError = error;
    if (Date.now() >= deadline) break;
  }
}
throw lastError instanceof Error ? lastError : new Error(String(lastError));
}

// Usage
async addProductLine(search: string, exactLabel: string): Promise<void> {
  await this.page.getByRole('button', { name: 'Add a line' }).click();
  await this.retryInteraction(async () => {
    await this.page.getByPlaceholder('Search a product').fill(search);
    await this.page
      .getByRole('option', { name: exactLabel })
      .first()
      .click({ timeout: 2_000 });
  });
}
```

---

## Utils — API

```typescript
// utils/api/ApiHelper.ts
import { type APIRequestContext } from '@playwright/test';

export class ApiHelper {
  constructor(private request: APIRequestContext, private baseURL: string) {}

  async get<T>(endpoint: string, headers?: Record<string, string>): Promise<T> {
    const response = await this.request.get(`${this.baseURL}${endpoint}`, { headers });
    return response.json() as Promise<T>;
  }

  async post<T>(endpoint: string, data: unknown, headers?: Record<string, string>): Promise<T> {
    const response = await this.request.post(`${this.baseURL}${endpoint}`, { data, headers });
    return response.json() as Promise<T>;
  }

  async put<T>(endpoint: string, data: unknown, headers?: Record<string, string>): Promise<T> {
    const response = await this.request.put(`${this.baseURL}${endpoint}`, { data, headers });
    return response.json() as Promise<T>;
  }

  async delete(endpoint: string, headers?: Record<string, string>): Promise<number> {
    const response = await this.request.delete(`${this.baseURL}${endpoint}`, { headers });
    return response.status();
  }
}
```

### Domain API Example

```typescript
// utils/api/AuthApi.ts
import { type APIRequestContext } from '@playwright/test';
import { ApiHelper } from './ApiHelper';
import { config } from '../config';

export class AuthApi {
  private api: ApiHelper;
  constructor(request: APIRequestContext) {
    this.api = new ApiHelper(request, config.baseUrl);
  }

  async login(username: string, password: string): Promise<{ token: string }> {
    return this.api.post('/api/auth/login', { username, password });
  }

  async register(email: string, password: string): Promise<{ userId: string }> {
    return this.api.post('/api/auth/register', { email, password });
  }

  async refreshToken(token: string): Promise<{ token: string }> {
    return this.api.post('/api/auth/refresh', {}, { Authorization: `Bearer ${token}` });
  }
}
```

---

## Utils — Database

Expose the database read-only over HTTP (e.g. [PostgREST](https://postgrest.org) for PostgreSQL).
The UI saying "Done" is necessary but not sufficient — **the database is the source of truth**.

```typescript
// utils/db/client.ts
export interface DbConfig {
  baseUrl: string;
  token?: string;
  timeoutMs: number;
}

export class DbClient {
  constructor(private config: DbConfig) {}

  async query<T>(path: string, params?: Record<string, string>): Promise<T[]> {
    const url = new URL(`${this.config.baseUrl}${path}`);
    if (params) for (const [k, v] of Object.entries(params)) url.searchParams.set(k, v);
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), this.config.timeoutMs);
    try {
      const res = await fetch(url.toString(), {
        signal: controller.signal,
        headers: {
          Accept: 'application/json',
          ...(this.config.token ? { Authorization: `Bearer ${this.config.token}` } : {}),
        },
      });
      if (!res.ok) throw new Error(`DB query failed: ${res.status} ${res.statusText} — ${url}`);
      return res.json() as Promise<T[]>;
    } finally {
      clearTimeout(timer);
    }
  }

  /** Throws if not exactly one row */
  async querySingle<T>(path: string, params?: Record<string, string>): Promise<T> {
    const rows = await this.query<T>(path, params);
    if (rows.length !== 1) throw new Error(`Expected 1 row from ${path}, got ${rows.length}`);
    return rows[0];
  }

  /** Returns null when nothing matches */
  async queryMaybeSingle<T>(path: string, params?: Record<string, string>): Promise<T | null> {
    const rows = await this.query<T>(path, { ...params, limit: '1' });
    return rows.length > 0 ? rows[0] : null;
  }
}
```

### DB Config

```typescript
// utils/db/config.ts
export function resolveDbConfig(overrides: Partial<DbConfig> = {}): DbConfig {
  return {
    baseUrl: (overrides.baseUrl ?? process.env.POSTGREST_URL ?? 'http://localhost:3000')
      .replace(/\/+$/, ''),
    token: overrides.token ?? process.env.POSTGREST_TOKEN,
    timeoutMs: overrides.timeoutMs ?? Number(process.env.POSTGREST_TIMEOUT_MS) || 10_000,
  };
}
```

### PostgREST Docker Sidecar

```yaml
# docker-compose.yml (excerpt)
services:
  api:
    image: postgrest/postgrest:v14
    ports: ['3000:3000']
    environment:
      PGRST_DB_URI: postgres://user:password@db:5432/mydb
      PGRST_DB_SCHEMA: public
      PGRST_DB_ANON_ROLE: readonly_role
    depends_on: [db]
```

### DB Before/After Snapshot Pattern

```typescript
// Snapshot BEFORE
const rowsBefore = await db.query<{ quantity: number }>('/stock_quant', {
  product_id: `eq.${productId}`,
  select: 'quantity',
});
const qtyBefore = rowsBefore.reduce((sum, r) => sum + Number(r.quantity ?? 0), 0);

// ... UI flow ...

// Snapshot AFTER
const record = await db.querySingle<{ state: string }>('/operations', {
  reference: `eq.${ref}`,
  select: 'state',
});
const rowsAfter = await db.query<{ quantity: number }>('/stock_quant', {
  product_id: `eq.${productId}`,
  select: 'quantity',
});
const qtyAfter = rowsAfter.reduce((sum, r) => sum + Number(r.quantity ?? 0), 0);

// All assertions in the spec:
expect(record.state, 'record state in DB').toBe('done');
expect(qtyAfter - qtyBefore, 'on-hand delta in DB').toBe(expectedQty);
```

### Stable Map Comparison

```typescript
/** Sorted [key, value] pairs for stable deep equality comparisons */
const normalize = (m: Map<number, number>): Array<[number, number]> =>
    [...m.entries()].sort((a, b) => a[0] - b[0]);

expect(normalize(actualByProduct)).toEqual(normalize(expectedByProduct));
```

---

## Fixtures

### Auth Setup (runs once, saves storageState)

```typescript
// fixtures/auth/auth.setup.ts
import { test as setup } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';
import { config } from 'utils/config';
import path from 'path';

const AUTH_FILE = path.join(__dirname, 'storageState.json');

setup('authenticate', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login(config.credentials.username, config.credentials.password);
  await page.context().storageState({ path: AUTH_FILE });
});
```

### Test Fixtures

```typescript
// fixtures/test-fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';
import { DbClient } from '@utils/db/client';
import { resolveDbConfig } from '@utils/db/config';

type TestFixtures = {
  loginPage: LoginPage;
  db: DbClient;
};

export const test = base.extend<TestFixtures>({
  loginPage: async ({ page }, use) => { await use(new LoginPage(page)); },
  db: async ({}, use) => { await use(new DbClient(resolveDbConfig())); },
});

export { expect } from '@playwright/test';
```

---

## Data-Driven Tests

### Typed scenario files

Scenario data lives in `data/` as typed TypeScript exports. No test logic, no assertions — only shape definitions and values.

```typescript
// data/receipt-scenarios.ts
export interface ReceiptLine {
  product: string;
  search?: string;   // shorter search term when full label doesn't match
  quantity: number;
  defaultCode?: string;  // DB key for integrity verification
}

export interface ReceiptScenario {
  id: string;    // stable ID shown in report, e.g. "TC-01"
  title: string;
  lines: ReceiptLine[];
}

export const receiptScenarios: ReceiptScenario[] = [
  {
    id: 'TC-01',
    title: 'Receive 5 units of Product A from vendor',
    lines: [{ product: 'Product A', quantity: 5, defaultCode: 'PROD-A' }],
  },
  {
    id: 'TC-02',
    title: 'Receive multiple products in one shipment',
    lines: [
      { product: 'Product A', quantity: 3, defaultCode: 'PROD-A' },
      { product: 'Product B', quantity: 7, defaultCode: 'PROD-B' },
    ],
  },
];
```

### External validation files (CSV / XLSX)

When test scenarios are driven by external files, always validate the file exists and fully await the parse before any test runs. A stream callback is async — returning from `beforeAll` without awaiting it means tests run against an empty dataset.

```typescript
// ✅ Correct — beforeAll truly waits for the stream to finish
const messages: ValidationRow[] = [];

test.beforeAll(async () => {
  if (!fs.existsSync(validationFilePath)) {
    throw new Error(`Validation file not found: ${validationFilePath}`);
  }
  await new Promise<void>((resolve, reject) => {
    fs.createReadStream(validationFilePath)
      .pipe(csvParser())
      .on('data', (row: ValidationRow) => messages.push(row))
      .on('end', resolve)
      .on('error', reject);
  });
});

// ❌ Wrong — beforeAll returns before the stream ends; messages is empty when tests run
test.beforeAll(async () => {
  fs.createReadStream(validationFilePath)
    .pipe(csvParser())
    .on('data', (row) => messages.push(row)); // no await, no Promise
});
```

### Test name includes scenario metadata

When looping over files or scenarios, put the scenario type and expectation in the test title — not just the filename:

```typescript
// ✅ Clear in the HTML report
test(`[${scenario.toUpperCase()}] ${filename} — ${expectedMessage}`, async ({ page }) => { ... });

// ❌ Opaque — you have to open the test to know what it tests
test(`Upload and Verify ${filename}`, async ({ page }) => { ... });
```

---

## Spec Example with DB Verification

```typescript
// tests/01-receive-inventory.spec.ts
import { test, expect } from '@fixtures/test-fixtures';
import { receiptScenarios } from '@data/receipt-scenarios';

const normalize = (m: Map<number, number>): Array<[number, number]> =>
  [...m.entries()].sort((a, b) => a[0] - b[0]);

test.describe('Receive Inventory into Warehouse — Happy Path', () => {
  for (const scenario of receiptScenarios) {
    test(`${scenario.id} | ${scenario.title}`, async ({ inventoryPage, db }) => {
      const { page } = inventoryPage;

      let qtyBefore = 0;
      await test.step('Snapshot warehouse on-hand before receiving', async () => {
        const rows = await db.query<{ quantity: number }>('/stock_quant', {
          location_id: 'eq.5',
          select: 'quantity',
        });
        qtyBefore = rows.reduce((sum, r) => sum + Number(r.quantity ?? 0), 0);
      });

      await test.step('Open the Inventory app', async () => {
        await inventoryPage.open();
        await expect(page).toHaveURL(/\/inventory/);
      });

      await test.step('Navigate to Receipts', async () => {
        await inventoryPage.goToReceipts();
        await expect(page).toHaveURL(/\/receipts/);
        await expect(page.getByRole('button', { name: 'New' })).toBeVisible();
      });

      let reference = '';
      await test.step('Create receipt and capture reference', async () => {
        reference = await inventoryPage.createReceipt(scenario.lines);
      });

      await test.step('Receipt starts in Draft state', async () => {
        await expect(
          page.getByRole('radio', { name: 'Draft' }),
          'A freshly saved receipt should be in Draft',
        ).toBeChecked();
      });

      await test.step('Validate the receipt', async () => {
        await inventoryPage.validateReceipt();
      });

      await test.step('Receipt is in Done state', async () => {
        await expect(
          page.getByRole('radio', { name: 'Done' }),
          'Receipt should be Done after validation',
        ).toBeChecked();
      });

      await test.step('Database reflects the validated receipt', async () => {
        const picking = await db.querySingle<{ state: string; location_dest_id: number }>(
          '/stock_picking',
          { name: `eq.${reference}`, select: 'state,location_dest_id' },
        );
        expect(picking.state, 'picking state in DB').toBe('done');

        const rows = await db.query<{ quantity: number }>('/stock_quant', {
          location_id: 'eq.5',
          select: 'quantity',
        });
        const qtyAfter = rows.reduce((sum, r) => sum + Number(r.quantity ?? 0), 0);
        const expectedDelta = scenario.lines.reduce((sum, l) => sum + l.quantity, 0);
        expect(qtyAfter - qtyBefore, 'on-hand delta in DB').toBe(expectedDelta);
      });
    });
  }
});
```
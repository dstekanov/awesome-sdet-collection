# Conventions Reference

Naming, structure, configuration, and style rules for Playwright TypeScript projects.

---

## File Naming

| Artifact | Convention | Example |
|----------|-----------|---------|
| Spec file | `{feature}-{action}.spec.ts` | `login-success.spec.ts`, `checkout-payment.spec.ts` |
| Page Object | `{feature}.page.ts` | `login.page.ts`, `checkout.page.ts` |
| Component PO | `{name}.component.ts` | `header.component.ts`, `modal.component.ts` |
| Fixture file | `fixtures.ts` | `tests/fixtures.ts` |
| Auth setup | `auth.setup.ts` | `tests/auth.setup.ts` |
| Test data file | `{name}.data.ts` | `users.data.ts`, `products.data.ts` |

---

## Directory Structure

### Flat (small projects)

```
tests/
  login.spec.ts
  checkout.spec.ts
  pages/
    login.page.ts
    checkout.page.ts
  fixtures.ts
  auth.setup.ts
playwright.config.ts
```

### Feature-based (medium/large projects)

```
tests/
  auth/
    login.spec.ts
    logout.spec.ts
    pages/
      login.page.ts
  checkout/
    payment.spec.ts
    shipping.spec.ts
    pages/
      checkout.page.ts
  shared/
    pages/
      header.component.ts
    fixtures.ts
    auth.setup.ts
playwright.config.ts
```

### Domain-based (enterprise / multi-product)

```
e2e/
  admin/
    users/
      create-user.spec.ts
      delete-user.spec.ts
    pages/
      users-list.page.ts
      user-form.page.ts
  customer/
    onboarding/
      registration.spec.ts
    pages/
      registration.page.ts
  shared/
    components/
      header.component.ts
    fixtures.ts
    auth.setup.ts
playwright.config.ts
```

---

## Test Describe / It Naming

```typescript
// describe: "Feature: <FeatureName>" or "<PageName>"
test.describe('Feature: User Management', () => {

  // test: action + expected outcome (no "should")
  test('creates user with valid data', ...);              // ✅
  test('shows validation error for duplicate email', ...); // ✅
  test('should create user', ...);                         // ❌ avoid "should"
  test('create user test', ...);                           // ❌ avoid "test" suffix
});
```

---

## Playwright Config — Recommended Defaults

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['list'],
  ],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: '.auth/user.json',
      },
      dependencies: ['setup'],
    },
    {
      name: 'firefox',
      use: {
        ...devices['Desktop Firefox'],
        storageState: '.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

---

## Environment Variables

Never hardcode URLs or credentials. Use `.env` / `process.env`:

```typescript
// playwright.config.ts
use: {
  baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
}

// In tests — via environment helpers
const email = process.env.TEST_EMAIL ?? 'test@example.com';
const password = process.env.TEST_PASSWORD ?? 'password';
```

`.env.example` (commit this, not `.env`):
```
BASE_URL=http://localhost:3000
TEST_EMAIL=test@example.com
TEST_PASSWORD=password
```

---

## Tags and Annotations

Use `tag` and `annotation` options for filtering and linking:

```typescript
// Smoke test
test('login smoke', { tag: '@smoke' }, async ({ page }) => { ... });

// Regression
test('checkout full flow', { tag: '@regression' }, async ({ page }) => { ... });

// Link to issue
test('fix: empty cart shows message', {
  annotation: { type: 'issue', description: 'https://github.com/org/repo/issues/42' },
}, async ({ page }) => { ... });

// Multiple tags
test('payment with 3DS', { tag: ['@smoke', '@payment'] }, async ({ page }) => { ... });
```

Run by tag:

```bash
npx playwright test --grep @smoke
npx playwright test --grep-invert @slow
```

---

## Timeouts

```typescript
// playwright.config.ts — global defaults
export default defineConfig({
  timeout: 30_000,        // per test
  expect: {
    timeout: 5_000,       // per assertion
  },
});

// Per test override
test('slow operation', async ({ page }) => {
  test.setTimeout(60_000);
  // ...
});

// Per assertion override
await expect(page.getByTestId('result')).toBeVisible({ timeout: 15_000 });
```

---

## Locator Naming in Page Objects

| Element type | Naming convention | Example |
|-------------|-----------------|---------|
| Input field | `{name}Input` | `emailInput`, `passwordInput` |
| Button | `{name}Button` | `submitButton`, `cancelButton` |
| Link | `{name}Link` | `dashboardLink`, `logoutLink` |
| Heading | `{name}Heading` | `pageHeading`, `sectionHeading` |
| Table row | `{name}Row` (method) | `userRow(name: string)` |
| Modal/Dialog | `{name}Dialog` | `confirmDialog`, `editModal` |
| Generic container | `{name}Container` or `{name}Section` | `statsSection` |
| Error message | `{name}Error` or `errorMessage` | `emailError`, `errorMessage` |
| Loading spinner | `loadingSpinner` or `{name}Spinner` | `submitSpinner` |

---

## Test Data

Never use magic strings inline. Extract to constants or data files:

```typescript
// tests/data/users.data.ts
export const TEST_USERS = {
  admin: {
    email: 'admin@example.com',
    password: 'AdminPass1!',
    role: 'Admin',
  },
  viewer: {
    email: 'viewer@example.com',
    password: 'ViewerPass1!',
    role: 'Viewer',
  },
} as const;
```

```typescript
// In spec
import { TEST_USERS } from '../data/users.data';

test('admin can delete users', async ({ page }) => {
  await loginPage.login(TEST_USERS.admin.email, TEST_USERS.admin.password);
  // ...
});
```

---

## `expect` Best Practices

```typescript
// ✅ Auto-retry built into expect — no manual waits needed
await expect(page.getByText('Success')).toBeVisible();

// ❌ Avoid manual timeouts/sleeps — they make tests flaky
await page.waitForTimeout(2000); // BAD

// ✅ Wait for state, not time
await expect(page.getByTestId('spinner')).not.toBeVisible();

// ✅ Use polling for conditions outside the DOM
await expect.poll(() => fetchOrderStatus(orderId), {
  timeout: 30_000,
}).toBe('completed');

// ✅ Soft assertions — continue on failure, report all issues at the end
await expect.soft(page.getByTestId('title')).toHaveText('My Page');
await expect.soft(page.getByTestId('subtitle')).toBeVisible();
```

---

## TypeScript Strict Mode

Always enable strict TypeScript for Playwright projects:

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2020",
    "module": "commonjs",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "outDir": "./dist",
    "rootDir": "./",
    "baseUrl": "."
  },
  "include": ["tests/**/*.ts", "playwright.config.ts"]
}
```

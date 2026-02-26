---
name: generate-playwright-test
description: Generate Playwright TypeScript e2e test files from test scenarios. Use when a QA engineer or developer has a BDD use case, classic test case, or free-form description and needs a Playwright test file that follows best practices.
compatibility: Generic TypeScript. Works with any web application. Assumes Playwright Test (@playwright/test) with Page Object Model pattern.
license: MIT
metadata:
  author: dstekanov
  version: "1.0"
---

# Generate Playwright Test

## When to use this skill

Use when:
- A QA engineer or developer has a test scenario and needs a Playwright TypeScript test file
- You need to create a new e2e test for a feature without automated coverage
- You need to translate a manual test case into automated Playwright spec
- You need to add a test to an existing Playwright project

## Interaction protocol

When invoked:

1. **Accept the test scenario** — accept a BDD use case (Given/When/Then), classic test case (steps + expected results), Jira/TestRail description, or free-form text.

2. **Similarity analysis** — Before generating anything, search the project for existing tests (`tests/`, `e2e/`, `playwright/`, or wherever `*.spec.ts` files live) that match or overlap the requested scenario. Present findings:
   - Link to each similar test file
   - Estimated similarity % (based on matching steps and assertions)
   - What the existing test covers that the new scenario also needs
   - What is **missing** in the existing test vs. the new scenario
   - Whether any existing test **fully covers** the scenario
   - Only proceed after the user confirms generation is needed.

3. **Confirm placement** — Infer the test file path from the scenario and project structure, then ask to confirm:
   - **Feature area:** auth, checkout, dashboard, profile, admin, etc.
   - **Test file name:** `feature-action.spec.ts` pattern
   - **Page Object** required: yes / no / reuse existing
   - Present inferred values and ask "Is this correct?"

4. **Ask for missing details** — If the scenario is underspecified, ask for:
   - **URL / route** being tested
   - **Selector strategy** preference (`data-testid`, ARIA role, label text, etc.)
   - **Auth state** (does the test start logged in? use `storageState`?)
   - Any environment-specific config (`baseURL`, credentials, etc.)

5. **Generate the test** — Create the spec file (and Page Object if needed) at the confirmed path.

6. **Offer to run and fix** — After generating, ask if the user wants to run the test. If yes, execute `npx playwright test <file>` and iterate on failures.

## Generation rules

### Spec file skeleton

Every generated test MUST follow this structure:

```typescript
import { test, expect } from '@playwright/test';
import { FeaturePage } from '../pages/feature.page';

test.describe('Feature: {{feature_name}}', () => {
  let featurePage: FeaturePage;

  test.beforeEach(async ({ page }) => {
    featurePage = new FeaturePage(page);
    await featurePage.goto();
  });

  test('{{test_description}}', async ({ page }) => {
    // Arrange
    // Act
    // Assert
  });
});
```

When **no Page Object** is needed (simple, one-off test):

```typescript
import { test, expect } from '@playwright/test';

test.describe('{{feature_name}}', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/your-path');
  });

  test('{{test_description}}', async ({ page }) => {
    await page.getByRole('button', { name: 'Submit' }).click();
    await expect(page.getByText('Success')).toBeVisible();
  });
});
```

### Locator strategy — priority order

Always prefer in this order (most resilient → least resilient):

1. **`getByRole`** — semantic ARIA role (best)
   ```typescript
   page.getByRole('button', { name: 'Sign in' })
   page.getByRole('textbox', { name: 'Email' })
   page.getByRole('heading', { name: 'Dashboard' })
   ```

2. **`getByLabel`** — form fields via label
   ```typescript
   page.getByLabel('Password')
   ```

3. **`getByTestId`** — `data-testid` attribute (stable, framework-agnostic)
   ```typescript
   page.getByTestId('submit-button')
   ```

4. **`getByText`** — visible text (use for non-interactive elements)
   ```typescript
   page.getByText('Welcome back')
   ```

5. **`getByPlaceholder`** — input placeholder
   ```typescript
   page.getByPlaceholder('Enter your email')
   ```

6. **CSS / XPath** — last resort only, avoid when alternatives exist
   ```typescript
   page.locator('.error-message')  // only if no ARIA/testid option
   ```

### Action patterns

```typescript
// Navigation
await page.goto('/login');

// Filling forms
await page.getByLabel('Email').fill('user@example.com');
await page.getByLabel('Password').fill('secret');

// Clicking
await page.getByRole('button', { name: 'Sign in' }).click();

// Selecting from dropdown
await page.getByRole('combobox', { name: 'Country' }).selectOption('Ukraine');

// Checking checkbox
await page.getByRole('checkbox', { name: 'Remember me' }).check();

// Uploading a file
await page.getByLabel('Upload').setInputFiles('path/to/file.pdf');

// Keyboard
await page.getByRole('textbox').press('Enter');

// Hovering
await page.getByRole('button', { name: 'Menu' }).hover();

// Waiting for navigation
await Promise.all([
  page.waitForURL('**/dashboard'),
  page.getByRole('button', { name: 'Submit' }).click(),
]);
```

### Assertion patterns

Always use `expect` from `@playwright/test` — assertions have built-in auto-retry:

```typescript
// Visibility
await expect(page.getByText('Welcome')).toBeVisible();
await expect(page.getByRole('dialog')).not.toBeVisible();

// Text content
await expect(page.getByRole('heading')).toHaveText('Dashboard');
await expect(page.getByTestId('status')).toContainText('Active');

// URL
await expect(page).toHaveURL('/dashboard');
await expect(page).toHaveURL(/.*dashboard/);

// Input value
await expect(page.getByLabel('Email')).toHaveValue('user@example.com');

// Count
await expect(page.getByRole('listitem')).toHaveCount(5);

// Attribute
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled();
await expect(page.getByRole('button', { name: 'Loading' })).toBeDisabled();

// Checkbox
await expect(page.getByRole('checkbox')).toBeChecked();

// Page title
await expect(page).toHaveTitle('My App');
```

### Network interception

```typescript
// Mock API response
await page.route('**/api/users', async (route) => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify([{ id: 1, name: 'Alice' }]),
  });
});

// Wait for specific request
const responsePromise = page.waitForResponse('**/api/submit');
await page.getByRole('button', { name: 'Submit' }).click();
const response = await responsePromise;
expect(response.status()).toBe(200);

// Abort a request (simulate error)
await page.route('**/api/data', (route) => route.abort());
```

### Authentication with storageState

When the test requires an authenticated user, use a fixture with saved auth state instead of logging in every test:

```typescript
// playwright.config.ts — define project with storageState
// projects: [{ name: 'authenticated', use: { storageState: '.auth/user.json' } }]

// In test — use authenticated fixture
test.use({ storageState: '.auth/user.json' });

test('shows user dashboard', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
});
```

### Custom fixtures

When multiple tests share setup, extract a fixture:

```typescript
// fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from './pages/login.page';

type Fixtures = {
  loginPage: LoginPage;
};

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await use(loginPage);
  },
});

export { expect } from '@playwright/test';
```

### Test annotations and tags

```typescript
// Skip a test
test.skip('reason for skip', async ({ page }) => { ... });

// Mark as slow (triples timeout)
test.slow();

// Annotate with issue link
test('checkout flow', {
  annotation: { type: 'issue', description: 'https://github.com/org/repo/issues/42' },
}, async ({ page }) => { ... });

// Tag for filtering
test('smoke: login', { tag: '@smoke' }, async ({ page }) => { ... });
// Run with: npx playwright test --grep @smoke
```

### Handling dynamic content

```typescript
// Wait for element to appear (auto-retry built in)
await expect(page.getByTestId('spinner')).not.toBeVisible();

// Wait for network idle after action
await page.waitForLoadState('networkidle');

// Poll for a condition (when expect retry is not enough)
await expect.poll(async () => {
  const response = await page.request.get('/api/status');
  return (await response.json()).status;
}, { timeout: 30_000 }).toBe('ready');
```

### Handling assertions with no established project pattern

When the scenario requests assertions for fields/behaviors with no existing test coverage:

1. **Scaffold the action + locator** with the most likely selector strategy
2. **Add `// TODO` comments** with the expected assertion and a note to update the selector
3. **Flag to the user** in the summary which assertions are TODOs and why

```typescript
// TODO: Replace selector with actual data-testid once confirmed with dev team
// await expect(page.getByTestId('order-status')).toHaveText('Confirmed');
await expect(page.locator('[data-status]')).toBeVisible(); // placeholder
```

## File placement logic

Infer file paths from the project's existing structure. Common conventions:

| Project layout | Spec path | Page Object path |
|---------------|-----------|-----------------|
| `tests/` flat | `tests/feature-action.spec.ts` | `tests/pages/feature.page.ts` |
| `e2e/` nested | `e2e/feature/action.spec.ts` | `e2e/pages/feature.page.ts` |
| `playwright/` | `playwright/specs/feature.spec.ts` | `playwright/pages/feature.page.ts` |
| `src/__tests__/` | `src/__tests__/e2e/feature.spec.ts` | `src/__tests__/pages/feature.page.ts` |

**File naming rules:**
- Spec: `{feature}-{action}.spec.ts` (e.g., `login-success.spec.ts`, `checkout-payment.spec.ts`)
- Page Object: `{feature}.page.ts` (e.g., `login.page.ts`, `checkout.page.ts`)
- Fixtures: `fixtures.ts` in the root of the tests directory

## When to create a Page Object

**Create a Page Object when:**
- The page/component is reused across 2+ tests
- The page has many interactable elements (forms, tables, modals)
- The page has complex navigation or multi-step workflows

**Skip the Page Object when:**
- It's a single, simple test with 2–3 interactions
- The page is already covered by an existing Page Object

## Specific tasks

- **Common test patterns** → [references/test-patterns.md](references/test-patterns.md)
- **Page Object Model patterns** → [references/page-object-patterns.md](references/page-object-patterns.md)
- **Naming, tagging, config conventions** → [references/conventions.md](references/conventions.md)

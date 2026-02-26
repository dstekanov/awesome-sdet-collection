# Test Patterns Reference

Common, ready-to-use Playwright TypeScript test patterns. Copy-adapt as needed.

---

## 1. Login / Authentication

```typescript
import { test, expect } from '@playwright/test';

test.describe('Login', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('successful login redirects to dashboard', async ({ page }) => {
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('correctpassword');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  });

  test('invalid credentials shows error message', async ({ page }) => {
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('wrongpassword');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page.getByText('Invalid email or password')).toBeVisible();
    await expect(page).toHaveURL('/login');
  });

  test('empty form shows validation errors', async ({ page }) => {
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page.getByText('Email is required')).toBeVisible();
    await expect(page.getByText('Password is required')).toBeVisible();
  });
});
```

---

## 2. Form Submit (Create / Update)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Create User Form', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/users/new');
  });

  test('creates user with valid data', async ({ page }) => {
    await page.getByLabel('First name').fill('John');
    await page.getByLabel('Last name').fill('Doe');
    await page.getByLabel('Email').fill('john.doe@example.com');
    await page.getByRole('combobox', { name: 'Role' }).selectOption('Admin');
    await page.getByRole('checkbox', { name: 'Active' }).check();

    await page.getByRole('button', { name: 'Create user' }).click();

    await expect(page.getByText('User created successfully')).toBeVisible();
    await expect(page).toHaveURL(/\/users\/\d+/);
  });

  test('shows inline validation errors', async ({ page }) => {
    await page.getByLabel('Email').fill('not-an-email');
    await page.getByRole('button', { name: 'Create user' }).click();

    await expect(page.getByText('Enter a valid email address')).toBeVisible();
  });
});
```

---

## 3. Data Table / List

```typescript
import { test, expect } from '@playwright/test';

test.describe('Users list', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/users');
  });

  test('displays list of users', async ({ page }) => {
    const rows = page.getByRole('row').filter({ hasNot: page.getByRole('columnheader') });
    await expect(rows).toHaveCount(10); // expected number of rows
  });

  test('filters users by search query', async ({ page }) => {
    await page.getByRole('searchbox', { name: 'Search users' }).fill('Alice');
    await page.keyboard.press('Enter');

    const rows = page.getByRole('row').filter({ hasNot: page.getByRole('columnheader') });
    await expect(rows).toHaveCount(1);
    await expect(rows.first()).toContainText('Alice');
  });

  test('sorts by name column', async ({ page }) => {
    await page.getByRole('columnheader', { name: 'Name' }).click();

    const firstCell = page.getByRole('row').nth(1).getByRole('cell').first();
    await expect(firstCell).toHaveText('Aaron');
  });

  test('paginates to next page', async ({ page }) => {
    await page.getByRole('button', { name: 'Next page' }).click();

    await expect(page).toHaveURL(/page=2/);
    const rows = page.getByRole('row').filter({ hasNot: page.getByRole('columnheader') });
    await expect(rows).toHaveCount(10);
  });
});
```

---

## 4. Navigation / Routing

```typescript
import { test, expect } from '@playwright/test';

test.describe('Navigation', () => {
  test('sidebar links navigate to correct pages', async ({ page }) => {
    await page.goto('/dashboard');

    await page.getByRole('link', { name: 'Settings' }).click();
    await expect(page).toHaveURL('/settings');
    await expect(page.getByRole('heading', { name: 'Settings' })).toBeVisible();

    await page.getByRole('link', { name: 'Profile' }).click();
    await expect(page).toHaveURL('/profile');
  });

  test('browser back/forward works correctly', async ({ page }) => {
    await page.goto('/dashboard');
    await page.getByRole('link', { name: 'Settings' }).click();
    await expect(page).toHaveURL('/settings');

    await page.goBack();
    await expect(page).toHaveURL('/dashboard');

    await page.goForward();
    await expect(page).toHaveURL('/settings');
  });
});
```

---

## 5. Modal / Dialog

```typescript
import { test, expect } from '@playwright/test';

test.describe('Delete confirmation dialog', () => {
  test('confirms deletion and removes item', async ({ page }) => {
    await page.goto('/items');

    await page.getByRole('row', { name: /Item A/ }).getByRole('button', { name: 'Delete' }).click();

    const dialog = page.getByRole('dialog');
    await expect(dialog).toBeVisible();
    await expect(dialog.getByText('Are you sure you want to delete Item A?')).toBeVisible();

    await dialog.getByRole('button', { name: 'Confirm' }).click();

    await expect(dialog).not.toBeVisible();
    await expect(page.getByText('Item A deleted')).toBeVisible();
    await expect(page.getByRole('row', { name: /Item A/ })).not.toBeVisible();
  });

  test('cancels deletion and keeps item', async ({ page }) => {
    await page.goto('/items');

    await page.getByRole('row', { name: /Item A/ }).getByRole('button', { name: 'Delete' }).click();

    const dialog = page.getByRole('dialog');
    await dialog.getByRole('button', { name: 'Cancel' }).click();

    await expect(dialog).not.toBeVisible();
    await expect(page.getByRole('row', { name: /Item A/ })).toBeVisible();
  });
});
```

---

## 6. Multi-step Wizard / Checkout

```typescript
import { test, expect } from '@playwright/test';

test.describe('Checkout wizard', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/checkout');
  });

  test('completes checkout in 3 steps', async ({ page }) => {
    // Step 1 — Shipping
    await expect(page.getByRole('heading', { name: 'Shipping details' })).toBeVisible();
    await page.getByLabel('Full name').fill('Jane Smith');
    await page.getByLabel('Address').fill('123 Main St');
    await page.getByRole('button', { name: 'Continue' }).click();

    // Step 2 — Payment
    await expect(page.getByRole('heading', { name: 'Payment' })).toBeVisible();
    await page.getByLabel('Card number').fill('4111111111111111');
    await page.getByLabel('Expiry').fill('12/26');
    await page.getByLabel('CVV').fill('123');
    await page.getByRole('button', { name: 'Continue' }).click();

    // Step 3 — Review & confirm
    await expect(page.getByRole('heading', { name: 'Order summary' })).toBeVisible();
    await expect(page.getByText('Jane Smith')).toBeVisible();
    await page.getByRole('button', { name: 'Place order' }).click();

    // Success
    await expect(page).toHaveURL(/\/order-confirmation/);
    await expect(page.getByText('Order placed successfully')).toBeVisible();
  });

  test('can go back to previous step', async ({ page }) => {
    await page.getByLabel('Full name').fill('Jane Smith');
    await page.getByRole('button', { name: 'Continue' }).click();

    await expect(page.getByRole('heading', { name: 'Payment' })).toBeVisible();
    await page.getByRole('button', { name: 'Back' }).click();

    await expect(page.getByRole('heading', { name: 'Shipping details' })).toBeVisible();
    await expect(page.getByLabel('Full name')).toHaveValue('Jane Smith');
  });
});
```

---

## 7. API Mocking

```typescript
import { test, expect } from '@playwright/test';

test.describe('Dashboard with mocked API', () => {
  test('shows user list from API', async ({ page }) => {
    await page.route('**/api/users', async (route) => {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify([
          { id: 1, name: 'Alice', role: 'Admin' },
          { id: 2, name: 'Bob', role: 'Viewer' },
        ]),
      });
    });

    await page.goto('/dashboard');

    await expect(page.getByText('Alice')).toBeVisible();
    await expect(page.getByText('Bob')).toBeVisible();
  });

  test('shows error state when API fails', async ({ page }) => {
    await page.route('**/api/users', async (route) => {
      await route.fulfill({ status: 500 });
    });

    await page.goto('/dashboard');

    await expect(page.getByText('Something went wrong')).toBeVisible();
    await expect(page.getByRole('button', { name: 'Retry' })).toBeVisible();
  });

  test('intercepts and validates outgoing request body', async ({ page }) => {
    let capturedBody: unknown;

    await page.route('**/api/submit', async (route) => {
      capturedBody = route.request().postDataJSON();
      await route.fulfill({ status: 200, body: '{}' });
    });

    await page.goto('/form');
    await page.getByLabel('Name').fill('Alice');
    await page.getByRole('button', { name: 'Submit' }).click();

    expect(capturedBody).toMatchObject({ name: 'Alice' });
  });
});
```

---

## 8. File Upload

```typescript
import { test, expect } from '@playwright/test';
import path from 'path';

test.describe('File upload', () => {
  test('uploads a file and shows preview', async ({ page }) => {
    await page.goto('/upload');

    const filePath = path.join(__dirname, 'fixtures/sample.pdf');
    await page.getByLabel('Choose file').setInputFiles(filePath);

    await expect(page.getByText('sample.pdf')).toBeVisible();
    await page.getByRole('button', { name: 'Upload' }).click();

    await expect(page.getByText('Upload successful')).toBeVisible();
  });

  test('rejects files exceeding size limit', async ({ page }) => {
    await page.goto('/upload');

    const largeFilePath = path.join(__dirname, 'fixtures/large-file.zip');
    await page.getByLabel('Choose file').setInputFiles(largeFilePath);

    await expect(page.getByText('File size exceeds the 5MB limit')).toBeVisible();
  });
});
```

---

## 9. Accessibility (a11y)

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
  test('login page has no critical a11y violations', async ({ page }) => {
    await page.goto('/login');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze();

    expect(results.violations).toEqual([]);
  });
});
```

---

## 10. Visual Regression (Screenshot)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Visual regression', () => {
  test('dashboard matches snapshot', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page.getByRole('main')).toBeVisible();

    await expect(page).toHaveScreenshot('dashboard.png', {
      maxDiffPixelRatio: 0.02, // allow 2% pixel diff
    });
  });

  test('button states match snapshots', async ({ page }) => {
    await page.goto('/components');

    await expect(page.getByRole('button', { name: 'Primary' })).toHaveScreenshot('btn-primary.png');
    await expect(page.getByRole('button', { name: 'Disabled' })).toHaveScreenshot('btn-disabled.png');
  });
});
```

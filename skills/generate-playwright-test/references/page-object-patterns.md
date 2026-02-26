# Page Object Model Patterns

Reference for creating and using Page Objects in Playwright TypeScript projects.

---

## Basic Page Object

```typescript
// pages/login.page.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectErrorMessage(message: string) {
    await expect(this.errorMessage).toContainText(message);
  }
}
```

Usage in spec:

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

test('login with valid credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await expect(page).toHaveURL('/dashboard');
});
```

---

## Page Object with Navigation (returns next Page Object)

A clean pattern for multi-page flows — each action returns the next page object:

```typescript
// pages/login.page.ts
import { type Page } from '@playwright/test';
import { DashboardPage } from './dashboard.page';

export class LoginPage {
  constructor(readonly page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async loginAs(email: string, password: string): Promise<DashboardPage> {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign in' }).click();
    await this.page.waitForURL('/dashboard');
    return new DashboardPage(this.page);
  }
}
```

```typescript
// pages/dashboard.page.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class DashboardPage {
  readonly heading: Locator;
  readonly userMenu: Locator;

  constructor(readonly page: Page) {
    this.heading = page.getByRole('heading', { name: 'Dashboard' });
    this.userMenu = page.getByRole('button', { name: 'User menu' });
  }

  async expectLoaded() {
    await expect(this.heading).toBeVisible();
  }
}
```

Usage in spec:

```typescript
test('login flow reaches dashboard', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  const dashboardPage = await loginPage.loginAs('user@example.com', 'pass');
  await dashboardPage.expectLoaded();
});
```

---

## Page Object with Components (nested Page Objects)

For pages with complex, reusable UI components (header, sidebar, data table):

```typescript
// components/header.component.ts
import { type Page, type Locator } from '@playwright/test';

export class HeaderComponent {
  readonly logo: Locator;
  readonly navLinks: Locator;
  readonly userMenuButton: Locator;

  constructor(private readonly page: Page) {
    this.logo = page.getByRole('link', { name: 'Home' });
    this.navLinks = page.getByRole('navigation').getByRole('link');
    this.userMenuButton = page.getByRole('button', { name: 'User menu' });
  }

  async logout() {
    await this.userMenuButton.click();
    await this.page.getByRole('menuitem', { name: 'Sign out' }).click();
  }
}
```

```typescript
// pages/app.page.ts — base page with shared components
import { type Page } from '@playwright/test';
import { HeaderComponent } from '../components/header.component';

export class AppPage {
  readonly header: HeaderComponent;

  constructor(protected readonly page: Page) {
    this.header = new HeaderComponent(page);
  }
}
```

```typescript
// pages/dashboard.page.ts — extends AppPage
import { type Locator, expect } from '@playwright/test';
import { AppPage } from './app.page';

export class DashboardPage extends AppPage {
  readonly statsCard: Locator;

  constructor(page: import('@playwright/test').Page) {
    super(page);
    this.statsCard = page.getByTestId('stats-card');
  }

  async expectStatsCount(count: number) {
    await expect(this.statsCard).toHaveCount(count);
  }
}
```

---

## Page Object with Dynamic Locators

For tables, lists, or rows that need parameterized locators:

```typescript
// pages/users-list.page.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class UsersListPage {
  readonly searchInput: Locator;
  readonly addUserButton: Locator;

  constructor(readonly page: Page) {
    this.searchInput = page.getByRole('searchbox', { name: 'Search users' });
    this.addUserButton = page.getByRole('button', { name: 'Add user' });
  }

  async goto() {
    await this.page.goto('/users');
  }

  userRow(name: string): Locator {
    return this.page.getByRole('row', { name: new RegExp(name) });
  }

  deleteButtonFor(name: string): Locator {
    return this.userRow(name).getByRole('button', { name: 'Delete' });
  }

  editButtonFor(name: string): Locator {
    return this.userRow(name).getByRole('button', { name: 'Edit' });
  }

  async search(query: string) {
    await this.searchInput.fill(query);
    await this.page.keyboard.press('Enter');
  }

  async expectUserVisible(name: string) {
    await expect(this.userRow(name)).toBeVisible();
  }

  async expectUserNotVisible(name: string) {
    await expect(this.userRow(name)).not.toBeVisible();
  }
}
```

---

## Fixture-based Page Object Injection

The most scalable pattern — Page Objects injected via Playwright fixtures:

```typescript
// fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from './pages/login.page';
import { DashboardPage } from './pages/dashboard.page';
import { UsersListPage } from './pages/users-list.page';

type Pages = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  usersListPage: UsersListPage;
};

export const test = base.extend<Pages>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
  usersListPage: async ({ page }, use) => {
    await use(new UsersListPage(page));
  },
});

export { expect } from '@playwright/test';
```

Usage in spec (no manual instantiation):

```typescript
import { test, expect } from '../fixtures';

test('user can access user list', async ({ loginPage, usersListPage }) => {
  await loginPage.goto();
  await loginPage.login('admin@example.com', 'password');

  await usersListPage.goto();
  await expect(usersListPage.addUserButton).toBeVisible();
});
```

---

## Authenticated Fixture

Persist login session across tests using `storageState`:

```typescript
// auth.setup.ts — run once before all tests
import { test as setup } from '@playwright/test';

const authFile = '.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },
    {
      name: 'authenticated',
      use: { storageState: '.auth/user.json' },
      dependencies: ['setup'],
    },
  ],
});
```

---

## Page Object Checklist

Before generating a Page Object, verify:

- [ ] Constructor accepts `page: Page` and assigns all locators as `readonly` class properties
- [ ] Locators use `getByRole` / `getByLabel` / `getByTestId` — no raw CSS unless unavoidable
- [ ] Actions (click, fill, navigate) return `Promise<void>` or the next Page Object
- [ ] Assertions are wrapped in `expect*` helper methods (e.g., `expectLoaded()`, `expectErrorVisible()`)
- [ ] No hardcoded test data inside Page Objects — pass via parameters
- [ ] Dynamic locators (per-row, per-item) are methods that return `Locator`, not stored as properties

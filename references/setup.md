# Setup Reference — Playwright + Test Agents

## Prerequisites

- Node.js >= 18
- npm or yarn
- A running web application (local or staging URL)
- Test user credentials (if the app is behind auth)

---

## Capture: Video, Trace, and Screenshots (READ THIS FIRST)

Before configuring anything else, understand what Playwright captures on every test run. This skill enforces these defaults — do not turn them off.

### Three artifacts, three purposes

| Artifact | File | What it gives you | Default in this skill |
|---------|------|-------------------|----------------------|
| **Video** | `video.webm` | Continuous browser recording. Watch what the user "saw". | `on-first-retry` (only on failures) |
| **Trace** | `trace.zip` | Interactive timeline: every action + DOM snapshot + network + console log. **The primary debugging tool.** | `on-first-retry` |
| **Screenshot** | `test-failed-1.png` | Single image at the moment of failure. Easy to embed in reports. | `only-on-failure` |

### Why traces beat plain video

`expect-cli` and similar tools record video. Playwright records video **and** trace. The trace is dramatically more useful because:

- **Click any step in the timeline → see the DOM at that exact moment.** Inspect elements as they were rendered, not as they are now.
- **Network log is timestamped against actions.** See exactly which request fired during which click.
- **Console log + errors are inline.** No digging through DevTools after the fact.
- **The trace is portable.** A single `.zip` file. Send it in Slack, attach to a bug report, upload to CI artifacts. Anyone with `npx playwright show-trace trace.zip` can reproduce your debugging session.

### Opening a trace

```bash
# Most recent failed test
npx playwright show-trace test-results/*/trace.zip

# Specific trace file
npx playwright show-trace path/to/trace.zip

# Or open the HTML report and click any failed test → Trace tab
npx playwright show-report
```

The trace viewer opens in a browser. Use the timeline at the bottom — every action is a clickable step.

### CI artifact upload

The GitHub Actions workflow in `ci-cd.md` already uploads `test-results/` and `playwright-report/` as artifacts on every run. After a CI failure, download the artifact, unzip, and run `show-trace` locally — same debugging experience as if you'd run the test on your machine. Future Claude sessions can also download these artifacts and reason about failures from another conversation.

### When to override the defaults

- **Local debugging of a flaky test:** temporarily set `trace: 'on'` and `video: 'on'` to capture every run, including passes.
- **CI for high-traffic suites:** keep defaults — capturing only failures saves storage.
- **Never disable trace.** It's the single most valuable artifact this skill produces.

---

## Step 1: Install Playwright

```bash
# Initialize Playwright in your project
npm init playwright@latest

# This will:
# - Install @playwright/test
# - Create playwright.config.ts
# - Create a tests/ directory with example
# - Optionally install browsers
```

When prompted:
- Language: **TypeScript**
- Tests folder: **tests**
- GitHub Actions: **Yes** (if you want CI)
- Install browsers: **Yes**

## Step 2: Initialize Agents

```bash
# For Claude Code (recommended for this skill)
npx playwright init-agents --loop=claude

# For VS Code Copilot
npx playwright init-agents --loop=vscode

# For OpenCode
npx playwright init-agents --loop=opencode
```

This creates `.github/` folder with agent definitions (instructions + MCP tools). **Regenerate whenever Playwright is updated.**

## Step 3: Install Browsers

```bash
# Install Chromium only (faster, usually sufficient)
npx playwright install --with-deps chromium

# Or install all browsers
npx playwright install --with-deps
```

## Step 4: Configure playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  
  /* Maximum time per test */
  timeout: 30_000,
  
  /* Assertion timeout */
  expect: { timeout: 5_000 },
  
  /* Run tests in parallel */
  fullyParallel: true,
  
  /* Fail the build on CI if test.only left in source */
  forbidOnly: !!process.env.CI,
  
  /* Retry failed tests */
  retries: process.env.CI ? 2 : 0,
  
  /* Parallel workers */
  workers: process.env.CI ? 1 : undefined,
  
  /* Reporter */
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results/results.json' }],
    ['list'],
  ],

  /* Shared settings for all projects */
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    
    /* Collect trace on failure */
    trace: 'on-first-retry',
    
    /* Screenshot on failure */
    screenshot: 'only-on-failure',
    
    /* Video on failure */
    video: 'on-first-retry',
    
    /* Sensible defaults */
    actionTimeout: 10_000,
    navigationTimeout: 15_000,
  },

  projects: [
    /* Setup project for authentication */
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    
    {
      name: 'chromium',
      use: { 
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    /* Uncomment for multi-browser */
    // {
    //   name: 'firefox',
    //   use: { ...devices['Desktop Firefox'] },
    //   dependencies: ['setup'],
    // },
    
    /* Mobile viewport */
    // {
    //   name: 'mobile-chrome',
    //   use: { ...devices['Pixel 5'] },
    //   dependencies: ['setup'],
    // },
  ],

  /* Run local dev server before starting tests */
  // webServer: {
  //   command: 'npm run dev',
  //   url: 'http://localhost:3000',
  //   reuseExistingServer: !process.env.CI,
  // },
});
```

## Step 5: Authentication Setup

Create `tests/auth.setup.ts` for apps that require login:

```typescript
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  // Navigate to login
  await page.goto('/login');
  
  // Fill credentials
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL || 'test@example.com');
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD || 'testpassword');
  
  // Submit
  await page.getByRole('button', { name: 'Log in' }).click();
  
  // Wait for auth to complete
  await page.waitForURL('/dashboard');
  
  // Save signed-in state
  await page.context().storageState({ path: authFile });
});
```

### Multi-role auth (recommended for any app with permissions)

Most real apps have at least 2-3 roles (admin, user, guest). Set up one auth file per role and parameterize tests:

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';

const roles = [
  { name: 'admin',        email: 'admin@test.com',        password: 'test1234' },
  { name: 'profesional',  email: 'profesional@test.com',  password: 'test1234' },
  { name: 'superadmin',   email: 'superadmin@test.com',   password: 'test1234' },
];

for (const role of roles) {
  setup(`authenticate as ${role.name}`, async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(role.email);
    await page.getByLabel('Password').fill(role.password);
    await page.getByRole('button', { name: /log in|iniciar sesión/i }).click();
    await page.waitForURL(/\/dashboard|\/superadmin/);
    await page.context().storageState({ path: `playwright/.auth/${role.name}.json` });
  });
}
```

Then in `playwright.config.ts`, define one project per role:

```typescript
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'admin',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/admin.json' },
    dependencies: ['setup'],
    testMatch: /.*\.admin\.spec\.ts/,
  },
  {
    name: 'profesional',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/profesional.json' },
    dependencies: ['setup'],
    testMatch: /.*\.profesional\.spec\.ts/,
  },
  // ... etc
],
```

### Auth strategies — pick the one that matches your stack

Three patterns cover ~95% of real apps. Use semantic locators (`getByLabel`, `getByRole`) so the same code works regardless of framework.

#### Pattern A — Session cookie + CSRF (server-rendered apps)

Applies to most server-rendered frameworks: Laravel, Rails, Django, Symfony, ASP.NET MVC, Phoenix, Express + sessions, etc. Login submits a form, server sets a session cookie, subsequent requests carry it automatically.

```typescript
setup('authenticate (session + CSRF)', async ({ page }) => {
  // Always start clean — stale CSRF tokens are the #1 cause of flaky auth
  await page.context().clearCookies();

  await page.goto('/login');

  // Semantic locators work across i18n and framework variants
  await page.getByLabel(/email|user|correo/i).fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel(/password|contraseña|mot de passe/i).fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: /log in|sign in|iniciar sesión|entrar/i }).click();

  // Wait for the post-login redirect — adjust the regex to your app
  await page.waitForURL(/\/dashboard|\/home|\/$/);

  // Sanity check: a session-related cookie should exist
  const cookies = await page.context().cookies();
  const hasSession = cookies.some(c =>
    /session|auth|jwt|sid/i.test(c.name) || /xsrf|csrf/i.test(c.name)
  );
  if (!hasSession) {
    throw new Error('Login appeared to succeed but no session cookie was set');
  }

  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

#### Pattern B — Token-based (SPA + REST API)

Applies to React/Vue/Angular SPAs that store a JWT or API token in `localStorage` after login. Faster than UI auth — call the API directly.

```typescript
setup('authenticate (API token)', async ({ request, page }) => {
  const response = await request.post('/api/auth/login', {
    data: {
      email: process.env.TEST_USER_EMAIL,
      password: process.env.TEST_USER_PASSWORD,
    },
  });
  if (!response.ok()) throw new Error(`Login failed: ${response.status()}`);

  const { token } = await response.json();

  await page.goto('/');
  await page.evaluate(t => localStorage.setItem('auth_token', t), token);

  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

#### Pattern C — OAuth / SSO (third-party identity provider)

Applies to apps that delegate auth to Google, Microsoft, Auth0, Okta, etc. Three options ranked by reliability:

1. **Best:** bypass the OAuth flow entirely with a test-only login endpoint that issues a session for a test user. Add `/test-login?user=qa@test.com` guarded by `APP_ENV === 'testing'`.
2. **OK:** use the provider's test mode if it offers one (Auth0 has this).
3. **Last resort:** automate the provider's login UI. Fragile — providers change their HTML often.

### Common auth failures (framework-agnostic)

| Symptom | Cause | Fix |
|---------|-------|-----|
| `waitForURL` times out | Login failed silently — credentials wrong | Check `.env` values; try logging in manually first |
| `419` / `403` mid-test | CSRF token expired or session reused across runs | `clearCookies()` at start of `auth.setup.ts` |
| `401 Unauthenticated` after auth setup passes | `storageState` path mismatch between setup and project config | Use the exact same path string in both places |
| Auth works locally, fails in CI | `BASE_URL` / cookie domain mismatch | Set `BASE_URL` env in CI to the deployed app URL |
| Session expires mid-test | Long test exceeded server session lifetime | Shorten the test, or extend session lifetime in test env |
| Flaky pass/fail every other run | Two parallel workers hitting the same user account | Give each worker its own test user, or run auth-dependent tests with `workers: 1` |

### Multi-tenant auth (subdomain or path-based)

If your app is multi-tenant, the test user must belong to a specific tenant. Either:

- Pre-seed test tenants and use one auth file per tenant, OR
- Use a single tenant for tests and isolate test data with a `test_` prefix.

Document your choice in the test seeder so future devs (and Claude) know which tenant they're hitting.

## Step 6: Custom Fixtures

Create `tests/fixtures.ts`:

```typescript
import { test as base, expect } from '@playwright/test';

// Extend base test with custom fixtures
export const test = base.extend<{
  // Add custom fixtures here
  authenticatedPage: any;
}>({
  // Example: a page that's already at the app
  authenticatedPage: async ({ page }, use) => {
    await page.goto('/');
    await use(page);
  },
});

export { expect };
```

## Step 7: Seed Test

Create `tests/seed.spec.ts`:

```typescript
import { test, expect } from './fixtures';

test('seed', async ({ page }) => {
  // Navigate to the app
  await page.goto('/');
  
  // Verify the app loaded correctly
  await expect(page).toHaveTitle(/.+/);
  
  // This seed test serves as:
  // 1. Bootstrap for Planner/Generator agents
  // 2. Example of test structure and conventions
  // 3. Environment validation
});
```

## Step 8: Environment Variables

Create `.env` for local testing:

```bash
# .env
BASE_URL=http://localhost:3000
TEST_USER_EMAIL=qa@example.com
TEST_USER_PASSWORD=secure_test_password

# For API testing
API_BASE_URL=http://localhost:3000/api
API_KEY=test_api_key
```

Add to `.gitignore`:
```
playwright/.auth/
test-results/
playwright-report/
.env
```

## Step 9: Verify Setup

```bash
# Run the seed test to verify everything works
npx playwright test seed.spec.ts

# Open the HTML report
npx playwright show-report

# View traces for debugging
npx playwright show-trace test-results/*/trace.zip
```

## Step 10: Specs Directory

```bash
mkdir -p specs
```

This is where the Planner agent will save test plans as Markdown files.

---

## Project Structure After Setup

```
project/
├── .github/                      # Agent definitions (auto-generated)
│   ├── playwright-planner.md
│   ├── playwright-generator.md
│   └── playwright-healer.md
├── specs/                        # Test plans (Planner output)
├── tests/
│   ├── fixtures.ts               # Custom fixtures
│   ├── seed.spec.ts              # Seed test
│   └── auth.setup.ts             # Auth setup
├── playwright/
│   └── .auth/                    # Stored auth state
├── playwright.config.ts
├── .env
└── package.json
```

---

## Useful Commands

```bash
# Run all tests
npx playwright test

# Run specific test file
npx playwright test tests/checkout/place-order.spec.ts

# Run tests matching a pattern
npx playwright test --grep "place order"

# Run in headed mode (see the browser)
npx playwright test --headed

# Run in debug mode
npx playwright test --debug

# Run in UI mode (interactive)
npx playwright test --ui

# Generate test via codegen
npx playwright codegen http://localhost:3000

# Show last report
npx playwright show-report

# View a trace
npx playwright show-trace path/to/trace.zip
```

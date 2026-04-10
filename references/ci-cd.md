# CI/CD Integration — Post-Deploy QA Workflow

## Overview

This reference covers how to integrate Playwright QA agents into your CI/CD pipeline to run automatically after each deploy.

---

## GitHub Actions Workflow

### Basic: Run Tests After Deploy

```yaml
# .github/workflows/qa-post-deploy.yml
name: QA — Post-Deploy E2E Tests

on:
  # Trigger after deploy workflow completes
  workflow_run:
    workflows: ["Deploy"]
    types: [completed]
    branches: [main, staging]
  
  # Or trigger manually
  workflow_dispatch:
    inputs:
      base_url:
        description: 'URL to test against'
        required: false
        default: 'https://staging.tuapp.com'

env:
  BASE_URL: ${{ github.event.inputs.base_url || 'https://staging.tuapp.com' }}

jobs:
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    # Only run if deploy was successful
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps chromium
      
      - name: Run Playwright Tests
        run: npx playwright test
        env:
          BASE_URL: ${{ env.BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
      
      - name: Upload Test Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-${{ github.run_id }}
          path: playwright-report/
          retention-days: 30
      
      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ github.run_id }}
          path: test-results/
          retention-days: 14
```

### Advanced: Planner + Generator + Healer Loop in CI

```yaml
# .github/workflows/qa-agents-loop.yml
name: QA — AI Agent Testing Loop

on:
  workflow_dispatch:
    inputs:
      feature_area:
        description: 'Feature area to test (e.g., checkout, auth, dashboard)'
        required: true
      base_url:
        description: 'URL to test against'
        required: false
        default: 'https://staging.tuapp.com'

jobs:
  agent-qa:
    name: AI Agent QA Loop
    runs-on: ubuntu-latest
    timeout-minutes: 60
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - name: Install dependencies
        run: |
          npm ci
          npx playwright install --with-deps chromium
      
      # Run existing tests first (regression)
      - name: Run Regression Suite
        run: npx playwright test
        env:
          BASE_URL: ${{ github.event.inputs.base_url }}
        continue-on-error: true
      
      # Generate new tests with agents if specs exist
      - name: Check for specs
        id: check-specs
        run: |
          if [ -d "specs" ] && [ "$(ls -A specs/ 2>/dev/null)" ]; then
            echo "has_specs=true" >> $GITHUB_OUTPUT
          else
            echo "has_specs=false" >> $GITHUB_OUTPUT
          fi
      
      # Run full test suite
      - name: Run Full E2E Suite
        run: npx playwright test --reporter=json,html
        env:
          BASE_URL: ${{ github.event.inputs.base_url }}
      
      - name: Upload Reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: qa-agent-report-${{ github.run_id }}
          path: |
            playwright-report/
            test-results/
            specs/
          retention-days: 30
```

---

## Post-Deploy Script (Local / Staging)

Script to run the full QA loop locally or on staging:

```bash
#!/bin/bash
# scripts/run-qa.sh

set -e

BASE_URL=${1:-"http://localhost:3000"}
FEATURE=${2:-"all"}

echo "🧪 Running QA against: $BASE_URL"
echo "📋 Feature scope: $FEATURE"

# 1. Run regression suite
echo "━━━ Phase 1: Regression Suite ━━━"
BASE_URL=$BASE_URL npx playwright test --reporter=list 2>&1 || true

# 2. Check for new/updated specs
echo "━━━ Phase 2: Check Specs ━━━"
if [ -d "specs" ] && [ "$(ls -A specs/)" ]; then
  echo "Found specs, generating tests..."
  # Agents will be invoked by Claude Code
else
  echo "No specs found. Run Planner agent first."
fi

# 3. Run full suite with traces
echo "━━━ Phase 3: Full E2E Suite ━━━"
BASE_URL=$BASE_URL npx playwright test \
  --reporter=html,json \
  --trace=on \
  2>&1 || true

# 4. Generate report
echo "━━━ Phase 4: Report ━━━"
echo "📊 Report: playwright-report/index.html"
echo "🔍 Traces: test-results/"

# 5. Summary
TOTAL=$(cat test-results/results.json 2>/dev/null | jq '.stats.expected + .stats.unexpected + .stats.skipped' || echo "0")
PASSED=$(cat test-results/results.json 2>/dev/null | jq '.stats.expected' || echo "0")
FAILED=$(cat test-results/results.json 2>/dev/null | jq '.stats.unexpected' || echo "0")

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Total: $TOTAL | ✅ $PASSED | ❌ $FAILED"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

Make executable: `chmod +x scripts/run-qa.sh`

Usage:
```bash
# Local
./scripts/run-qa.sh http://localhost:3000

# Staging
./scripts/run-qa.sh https://staging.tuapp.com

# Specific feature
./scripts/run-qa.sh https://staging.tuapp.com checkout
```

---

## Workflow: After New Implementation

When a new feature is implemented, this is the Claude-driven workflow:

### Step 1: Generate Spec (Planner)

Prompt the Planner agent:
```
Use the Playwright Planner agent to explore [feature] at [URL]
and generate a comprehensive test plan covering:
- Happy paths
- Invalid data and validation
- Edge cases and boundary conditions  
- Error states
- Concurrent access
Save the plan to specs/[feature].md
```

### Step 2: Generate Tests (Generator)

Prompt the Generator agent:
```
Use the Playwright Generator agent to create tests from 
specs/[feature].md. Use tests/seed.spec.ts as the seed test.
Save generated tests to tests/[feature]/
```

### Step 3: Run & Heal

```bash
# Run the new tests
npx playwright test tests/[feature]/

# If failures, invoke Healer:
# "Use the Playwright Healer agent to fix failing tests in tests/[feature]/"
```

### Step 4: Add to Regression Suite

Once tests pass, they become part of the permanent regression suite that runs on every deploy.

---

## Docker Configuration (for CI)

```dockerfile
# Dockerfile.qa
FROM mcr.microsoft.com/playwright:v1.52.0-noble

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npx", "playwright", "test"]
```

```yaml
# docker-compose.qa.yml
services:
  qa:
    build:
      context: .
      dockerfile: Dockerfile.qa
    environment:
      - BASE_URL=http://app:3000
      - CI=true
    depends_on:
      - app
    volumes:
      - ./playwright-report:/app/playwright-report
      - ./test-results:/app/test-results
```

---

## Notifications (Optional)

### Slack Notification on Failure

Add to GitHub Actions workflow:

```yaml
- name: Notify Slack on Failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    text: |
      🔴 QA Tests Failed after deploy
      Branch: ${{ github.ref_name }}
      Report: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Environment Management

### Test Data Strategy

```typescript
// tests/helpers/test-data.ts

export function generateTestUser(overrides = {}) {
  const id = Date.now();
  return {
    name: `QA Test User ${id}`,
    email: `qa-test-${id}@test.com`,
    phone: `+1555${String(id).slice(-7)}`,
    ...overrides,
  };
}

export function generateTestOrder(overrides = {}) {
  const id = Date.now();
  return {
    customerEmail: `qa-test-${id}@test.com`,
    items: [{ sku: 'TEST-SKU-001', quantity: 1, price: 19.99 }],
    shippingAddress: '123 Test St, Test City, 00000',
    ...overrides,
  };
}

export function getFutureDate(days: number): string {
  const date = new Date();
  date.setDate(date.getDate() + days);
  return date.toISOString().split('T')[0];
}
```

### Cleanup After Tests

```typescript
// tests/global-teardown.ts
import { request } from '@playwright/test';

export default async function globalTeardown() {
  const api = await request.newContext({
    baseURL: process.env.BASE_URL,
  });
  
  // Clean up test data
  await api.delete('/api/test/cleanup', {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  
  await api.dispose();
}
```

Add to `playwright.config.ts`:
```typescript
export default defineConfig({
  globalTeardown: './tests/global-teardown.ts',
  // ...
});
```

---

## Monitoring Test Health

### Track Flaky Tests

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
    // Track flaky tests over time
    ['list', { printSteps: true }],
  ],
});
```

### Dashboard Integration

Export test results JSON and feed into your monitoring dashboard. Key metrics:
- Pass rate per feature area
- Flaky test rate (passed on retry)
- Test execution time trends
- Coverage gaps (untested features)

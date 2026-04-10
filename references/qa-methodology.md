# QA Methodology Reference — Exhaustive E2E Testing (Generic)

## Testing Philosophy

> **"You are not testing to confirm features work — you are testing to break them."**
> **"Si no lo has validado intentando romperlo, no funciona."**

Actúa como un QA Engineer senior con mentalidad de "romper el sistema". No asumas que nada funciona hasta haberlo verificado. Prioriza bugs críticos: pérdida de datos, estados inconsistentes, acciones duplicadas, y fallos de seguridad.

---

## How This Document Relates to the 8 Adversarial Angles

`SKILL.md` defines **8 adversarial angles** (the *vectors of attack*). This methodology document defines **7 functional test categories** (the *domains to attack*). They are orthogonal — both must be applied:

| | **Functional Categories (this doc)** | **Adversarial Angles (SKILL.md)** |
|---|---|---|
| **Question they answer** | *What domain of behavior do I need to test?* | *How do I attack each domain?* |
| **Examples** | CRUD, State Machines, Automation, Real-Time, Edge Cases, Consistency, Responsive | Empty inputs, invalid data, boundary values, injection, double-click, navigation, regression in nearby features, auth |
| **Granularity** | One per functional area of the app | One per attack vector applied to *each* functional area |
| **Where to start** | Map the app: which categories apply? | For each applicable category, run all 8 angles |

### How to use both together

1. **Map the feature to functional categories.** Is it CRUD? Does it have state transitions? Does it trigger automations? Each Yes opens a category from this document.
2. **For each opened category, run the 8 angles from `SKILL.md`.** A CRUD form must be tested with empty inputs, invalid data, boundary values, injection, double-click, navigation edges, regression in nearby features, and auth edges. Same for a state transition. Same for an automation.
3. **Use this document for the patterns and code samples.** When you need to write a "concurrent access" test or a "scheduled task" test, the canonical patterns are below. The *adversarial framing* of those patterns comes from the angles.

### Quick mapping: which angles apply to which categories

| Functional Category ↓  /  Angle → | 1 Empty | 2 Invalid | 3 Boundary | 4 Injection | 5 Double-click | 6 Navigation | 7 Nearby | 8 Auth |
|---|---|---|---|---|---|---|---|---|
| 1. CRUD                    | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2. State Machine           | ⚠️ | ✅ | — | — | ✅ | ✅ | ✅ | ✅ |
| 3. Automation/Integration  | — | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ |
| 4. Real-Time/Multi-User    | — | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| 5. Edge Cases              | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 6. Consistency             | — | — | — | — | ✅ | ✅ | ✅ | — |
| 7. Responsive/A11y         | — | — | ✅ | — | — | ✅ | — | — |

✅ = mandatory  ⚠️ = situational  — = not usually applicable

> **TL;DR:** the angles tell you *what to break*. The categories tell you *what kinds of things exist that can be broken*. Apply both, every time.

---

## How To Apply This Methodology

This is a **generic framework**. When testing a specific app, Claude must:

1. **Identify the app's domain** — What does it do? (e-commerce, SaaS, booking, CMS, dashboard...)
2. **Map the entities** — What are the core objects? (users, orders, posts, tickets, messages...)
3. **Map the operations** — What can be done with each entity? (CRUD, state transitions, automations...)
4. **Map the integrations** — What external systems are involved? (email, SMS, payments, APIs, webhooks...)
5. **Apply each test category below** to the mapped entities and operations

---

## Test Categories

### 1. FUNCTIONAL TESTING — CRUD & Core Flows

For **every entity** in the application, test all operations:

#### Create — Happy Path
```typescript
test('create [entity] with valid data', async ({ page }) => {
  // Navigate to creation form
  // Fill all required fields with valid data
  // Submit
  // Assert: success message, entity appears in list, correct data displayed
});
```

#### Create — Invalid & Edge Cases
Test each of these scenarios per create form:

| Scenario | What to test | Expected |
|----------|-------------|----------|
| Empty required fields | Submit with nothing filled | Validation errors on each field |
| Invalid format | Wrong email, phone, URL, date formats | Field-specific error messages |
| Boundary values | Min/max lengths, 0, -1, 999999, empty string | Graceful rejection or acceptance |
| Extreme strings | 10K+ chars, only spaces, only special chars | Truncation or validation error |
| Injection | `<script>alert(1)</script>`, `'; DROP TABLE--` | Sanitized, no execution |
| Unicode & emoji | Names with ñ, ü, 日本語, 🎉 | Stored and displayed correctly |
| Duplicate | Create same entity twice | Error or deduplicate, never silent duplicate |
| Past dates | If date field exists, try dates in the past | Reject if not allowed |
| Future extremes | Year 2099, year 1900 | Reject or handle gracefully |

#### Read / List
```typescript
test('list [entities] with pagination', async ({ page }) => {
  // Navigate to list view
  // Assert: items visible, count matches
  // Test pagination: next, previous, jump to page
  // Test sorting: by each sortable column
  // Test filtering: by each available filter
  // Test search: exact match, partial match, no results
});
```

#### Update
| Scenario | Expected |
|----------|----------|
| Valid single field change | Updated, confirmation shown |
| Valid multi-field change | All fields updated atomically |
| Change to conflicting value | Error (e.g., duplicate email, occupied slot) |
| Update deleted/archived entity | Blocked or error |
| Update entity in terminal state | Blocked (e.g., can't edit "completed" order) |
| Concurrent update by 2 users | Last-write-wins or conflict resolution |

#### Delete / Archive
| Scenario | Expected |
|----------|----------|
| Delete active entity | Removed from list, confirmation prompt |
| Delete already deleted entity | Idempotent or 404 |
| Delete entity with dependencies | Cascade or block with explanation |
| Double-click delete button | Only one deletion, not error |
| Soft delete vs hard delete | Verify which applies, check recovery option |

---

### 2. STATE MACHINE TESTING

Most entities have states (pending → confirmed → completed → cancelled, etc). For every state transition:

```
For each [current_state] → [target_state]:
  ✅ Test: valid transition succeeds
  ❌ Test: invalid transition is blocked
  🔄 Test: side effects fire correctly (emails, webhooks, UI updates)
  ⏱️ Test: timestamps update correctly
```

Example pattern:
```typescript
test('transition [entity] from [state_A] to [state_B]', async ({ page }) => {
  // Setup: create entity in state_A
  // Action: trigger transition (button click, API call, etc)
  // Assert: state is now state_B
  // Assert: UI reflects new state
  // Assert: side effects fired (notifications, logs, etc)
  // Assert: cannot transition backwards to state_A (if irreversible)
});
```

**Critical checks:**
- Draw the full state diagram for each entity
- Every arrow is a test case (valid transition)
- Every missing arrow is a test case (invalid transition must be blocked)
- Every state should have at least one exit path (no dead states)

---

### 3. AUTOMATION & INTEGRATION TESTING

For every automated side effect in the system (emails, SMS, webhooks, notifications, scheduled tasks):

| Check | How |
|-------|-----|
| Triggers on correct event | Create the event → verify automation fired |
| Does NOT trigger on wrong event | Create similar-but-different event → verify NO automation |
| Content is correct | Verify the payload/body contains current data, not stale |
| No duplicates | Trigger event once → verify exactly 1 automation, not 2+ |
| Retry on failure | Simulate integration failure → verify retry mechanism |
| Timing is correct | If delayed (e.g., "send 24h before"), verify exact timing |
| Cancellation stops pending automations | Cancel entity → verify scheduled automations are cancelled |
| Update reflects in pending automations | Modify entity → verify pending automations use NEW data |

#### Pattern: Testing Automated Communications
```typescript
test('automation fires on [event] with correct content', async ({ page, request }) => {
  // 1. Perform action that triggers automation (via UI or API)
  // 2. Wait for automation to process
  // 3. Query test endpoint / mailbox / log to verify:
  //    - Exactly 1 message sent (not 0, not 2+)
  //    - Recipient is correct
  //    - Content reflects current entity data
  //    - Template rendered (no raw {{variables}})
});
```

#### Pattern: Testing Scheduled / Timed Tasks
```typescript
test('scheduled task fires at correct time', async ({ page }) => {
  // Use Playwright Clock API to control time
  await page.clock.install({ time: new Date('2026-04-14T10:00:00') });
  
  // Create entity that will trigger timed automation
  // Fast-forward to just before expected trigger
  await page.clock.fastForward('23:59:00');
  // Assert: automation has NOT fired yet
  
  // Fast-forward past trigger point
  await page.clock.fastForward('00:02:00');
  // Assert: automation HAS fired
});
```

---

### 4. REAL-TIME & MULTI-USER TESTING

For apps with real-time features (chat, live updates, collaborative editing, dashboards):

```typescript
test('real-time sync between two users', async ({ browser }) => {
  const ctx1 = await browser.newContext();
  const ctx2 = await browser.newContext();
  const page1 = await ctx1.newPage();
  const page2 = await ctx2.newPage();
  
  // User 1 performs action
  // Assert: User 2 sees the change without refreshing
  
  await ctx1.close();
  await ctx2.close();
});
```

| Check | What to verify |
|-------|---------------|
| Sync | Action by user A is visible to user B in real-time |
| Ordering | Messages/events appear in correct chronological order |
| No data loss | Send N items rapidly → all N arrive |
| Reconnection | Simulate disconnect → reconnect → no data missed |
| Conflict | Both users edit same thing → resolved, not corrupted |

---

### 5. EDGE CASES & BREAK TESTING

Apply these to **every critical flow** in the app:

#### Concurrency
```typescript
test('concurrent access does not corrupt data', async ({ browser }) => {
  const ctx1 = await browser.newContext();
  const ctx2 = await browser.newContext();
  const page1 = await ctx1.newPage();
  const page2 = await ctx2.newPage();
  
  // Both users attempt same action simultaneously
  await Promise.all([
    performCriticalAction(page1),
    performCriticalAction(page2),
  ]);
  
  // Assert: one succeeds, one gets conflict error
  // OR: both succeed if action supports concurrency
  // NEVER: data corruption, duplicate records, or silent failure
  
  await ctx1.close();
  await ctx2.close();
});
```

#### Network Failures
```typescript
test('app handles network failure gracefully', async ({ page }) => {
  // Navigate and fill form with data
  
  // Block API calls
  await page.route('**/api/**', route => route.abort());
  
  // Attempt action
  // Assert: user-friendly error shown (not blank screen or raw error)
  // Assert: form data preserved (user doesn't lose their input)
  // Assert: app is still usable after error
});
```

#### Double Submit
```typescript
test('double submit does not create duplicate', async ({ page }) => {
  // Fill form
  // Double-click submit button
  await page.getByRole('button', { name: /submit|save|create|confirm/i }).dblclick();
  
  // Assert: only 1 entity created, not 2
});
```

#### Session Expiry
```typescript
test('expired session redirects to login', async ({ page }) => {
  // Clear auth cookies/storage
  await page.context().clearCookies();
  
  // Attempt protected action
  // Assert: redirects to login, not 500 error
  // Assert: after re-login, user returns to intended page
});
```

#### Malformed API Input
```typescript
test('API rejects malformed payload', async ({ request }) => {
  const cases = [
    { data: null },
    { data: {} },
    { data: { id: 'not-a-number' } },
    { data: { required_field: null } },
    { data: { number_field: -1 } },
    { data: { string_field: 'x'.repeat(100000) } },
  ];
  
  for (const { data } of cases) {
    const resp = await request.post('/api/endpoint', { data });
    expect(resp.status()).toBeGreaterThanOrEqual(400);
    expect(resp.status()).toBeLessThan(500); // Client error, not server crash
  }
});
```

#### Browser Edge Cases
- **Back button** after form submit → should not resubmit
- **Refresh** during operation → should not corrupt state
- **Multiple tabs** with same session → should stay in sync
- **Slow connection** (throttle network) → loading states, no timeouts
- **Zoom / font scaling** → UI remains usable

---

### 6. CONSISTENCY TESTING

Verify that data stays synchronized across all layers:

| Layer A | Layer B | Check |
|---------|---------|-------|
| Database state | UI display | Entity status matches what's shown |
| Entity state | Communication content | If status = "confirmed", email says "confirmed" |
| Action timestamp | Log timestamp | Events are logged at correct time |
| List count | Actual items | "Showing 15 results" → exactly 15 items rendered |
| Sum/totals | Individual values | Dashboard total = sum of individual entries |
| Cached data | Source data | After update, cached views reflect new data |

---

### 7. RESPONSIVE & ACCESSIBILITY

```typescript
// Test on mobile viewport
test.use({ viewport: { width: 375, height: 667 } });

test('critical flow works on mobile', async ({ page }) => {
  // Navigate through the main flow on mobile
  // Assert: all buttons tappable, forms fillable, no horizontal scroll
});
```

Quick a11y checks:
- Every interactive element reachable via keyboard (Tab navigation)
- Every form field has a label
- Error messages associated with their field
- Sufficient color contrast
- Screen reader can navigate critical flows

---

## Test Execution Strategy

### Priority Order (what to test first)

1. **Critical path** — The main thing users do (signup, purchase, create core entity)
2. **Money flows** — Anything involving payments, billing, credits
3. **Data integrity** — Operations that create/modify/delete permanent data
4. **Automations** — Side effects that can't be easily undone (emails, notifications)
5. **Auth & security** — Login, permissions, session management
6. **Edge cases** — Concurrency, failures, boundaries
7. **UX & polish** — Responsive, loading states, error messages

### When to use Playwright Agents vs Manual Tests

| Use Agents (Planner → Generator) | Write Manual Tests |
|----------------------------------|--------------------|
| New feature exploration | Complex business logic validation |
| Broad coverage sweep | Specific regression for known bugs |
| UI flow testing | API-level testing |
| After major refactors | Performance-sensitive assertions |

---

## Test Output Format

For every test executed, produce:

```markdown
### TEST: [descriptive name]
- **Action:** [what was done]
- **Expected:** [what should happen]
- **Actual:** [what really happened]
- **Status:** ✅ OK / ❌ ERROR
- **Severity:** Low / Medium / High / Critical
- **Details:** [technical info if error — stack trace, screenshot, network log]
- **Probable cause:** [hypothesis]
- **Suggested fix:** [how it could be resolved]
```

## Severity Classification

| Severity | Criteria | Examples |
|----------|----------|---------|
| **Critical** | Data loss, security breach, duplicate actions, money affected | Double charge, data leak, corrupted records |
| **High** | Core flow broken, user blocked | Can't create/edit/delete main entity, auth broken |
| **Medium** | Degraded functionality, confusing UX | Wrong data in notification, non-descriptive error |
| **Low** | Cosmetic, minor impact | Typo, alignment issue, missing hover state |

---

## Session Summary

At the end of each QA session, produce:

```markdown
## 📊 QA Summary

### Tests executed: X
- ✅ Passed: X
- ❌ Failed: X  
- ⏭️ Skipped: X

### 🔴 Top 5 Critical Bugs
1. [Most critical bug with description and steps to reproduce]
2. ...

### 💡 Proposed Improvements
- [Architecture, logic, or UX improvements detected]

### 📋 Coverage
- [Features tested vs pending]
- [Areas with no coverage identified]

### 🔄 New Regression Tests Added
- [List of new test files committed to the suite]
```

---

## Playwright API Quick Reference

### Locators (ALWAYS semantic)
```typescript
page.getByRole('button', { name: 'Submit' })
page.getByLabel('Email')
page.getByPlaceholder('Search...')
page.getByText('Welcome')
page.getByTestId('submit-btn')     // fallback when no semantic option
```

### Assertions (auto-retry built in)
```typescript
await expect(page).toHaveTitle(/Dashboard/);
await expect(page).toHaveURL('/dashboard');
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toHaveText('Expected');
await expect(locator).toHaveValue('input-value');
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();
await expect(locator).toHaveCount(5);
await expect(locator).toHaveAttribute('href', '/path');
await expect(locator).toContainText('partial');
```

### Network Interception
```typescript
// Mock API response
await page.route('**/api/endpoint', route =>
  route.fulfill({ status: 200, body: JSON.stringify({ id: 1 }) })
);

// Block requests (simulate failure)
await page.route('**/api/endpoint', route => route.abort());

// Wait for specific API call
const response = await page.waitForResponse('**/api/endpoint');
expect(response.status()).toBe(200);

// Intercept and modify response
await page.route('**/api/endpoint', async route => {
  const response = await route.fetch();
  const json = await response.json();
  json.modified = true;
  await route.fulfill({ response, json });
});
```

### Clock API (for timing-dependent tests)
```typescript
await page.clock.install({ time: new Date('2026-04-14T20:00:00') });
await page.clock.fastForward('01:00:00');
await page.clock.setFixedTime(new Date('2026-04-15T20:00:00'));
await page.clock.resume(); // back to real time
```

### API Testing (no browser needed)
```typescript
test('API: create entity', async ({ request }) => {
  const response = await request.post('/api/entities', {
    data: { name: 'Test', type: 'example' },
    headers: { Authorization: 'Bearer token' },
  });
  expect(response.ok()).toBeTruthy();
  const data = await response.json();
  expect(data.id).toBeDefined();
});
```

### Multi-context (multi-user simulation)
```typescript
test('two users interact', async ({ browser }) => {
  const ctx1 = await browser.newContext();
  const ctx2 = await browser.newContext();
  const page1 = await ctx1.newPage();
  const page2 = await ctx2.newPage();
  
  // ... test interaction ...
  
  await ctx1.close();
  await ctx2.close();
});
```

---
name: qa-engineer
description: "Adversarial-first E2E QA. Two activation modes — SUGGEST mode (auto, low cost) and EXECUTE mode (on user request, expensive). The core mantra: you are NOT testing to confirm features work — you are testing to BREAK them. Uses Playwright Test Agents (Planner → Generator → Healer) driven by Claude, structured around 8 mandatory adversarial angles (empty inputs, invalid data, boundary values, special chars/injection, double-click, navigation edges, regression in nearby features, auth/permission edges). Angle 7 (regression in nearby features) is non-negotiable: every QA pass re-tests the neighbors of the changed code. SUGGEST mode: load this skill into context whenever Claude makes any behavioral change (new feature, bug fix, refactor, schema/API change, form/flow modification). In suggest mode Claude does NOT run any tests — it appends a single-line suggestion at the end of the implementation message offering an adversarial QA pass with the exact phrase to invoke it. EXECUTE mode: when the user explicitly asks for QA with phrases like 'haz QA', 'testea esto', 'rompe esto', 'valida esto', 'prueba la app', 'lanza tests', 'run QA', 'test this adversarially', or says yes to a SUGGEST line, run the full Planner → Generator → Healer workflow. NEVER run the full workflow without explicit user consent. Covers: first-time Playwright setup, auth setup for protected apps, adversarial test plan generation, test execution, self-healing tests, optional CI/CD integration with video/trace artifacts, and a full QA methodology for exhaustive validation. Works with any web app regardless of stack."
license: MIT
metadata:
  author: AppsurDesarrollo
  version: "2.2.0"
  repository: "https://github.com/AppsurDesarrollo/qa-engineer-skill"
---

# qa-engineer — Adversarial E2E Testing Skill

## Core Mantra

> **You are not testing to confirm the feature works. You are testing to try to break it.**

This single sentence is the soul of the skill. Internalize it before every QA session:

- A test that "checks the page renders" is **worthless**. It only catches the bugs nobody would have shipped anyway.
- A test that submits an empty form, double-clicks the button, navigates back mid-flow, and verifies a nearby feature still works **earns its keep**.
- If you finish a QA session without having tried to break something, **you did not do QA** — you did a smoke test.

The Planner, Generator, and Healer agents are tools. The mindset is yours. Every prompt you send to the Planner, every test the Generator writes, every fix the Healer applies — all of it should be in service of *trying to break the system* on behalf of the user.

> **"Si no lo has validado intentando romperlo, no funciona."**

---

## ACTIVATION RULES — TWO MODES (CRITICAL — read before doing anything else)

This skill has **two activation modes**. The mode determines whether Claude *suggests* QA or *runs* QA. **Confusing them costs tokens and user trust.**

```
                  ┌─────────────────────┐
                  │  Skill loaded into  │
                  │   Claude's context  │
                  └──────────┬──────────┘
                             │
              ┌──────────────┴───────────────┐
              │                              │
              ▼                              ▼
    ┌─────────────────┐            ┌──────────────────┐
    │  SUGGEST mode   │            │  EXECUTE mode    │
    │ (auto, ~50 tok) │            │ (manual, ~200K)  │
    └─────────────────┘            └──────────────────┘
    Triggered by Claude            Triggered by user
    making a behavioral            explicitly asking
    change.                        for QA.
    
    Claude appends ONE             Claude runs the full
    suggestion line at the         Planner → Generator →
    end of its message.            Healer workflow.
    NO tests run.                  Tests run, report produced.
```

### SUGGEST mode — when to use it

**Trigger:** Claude has just made or is about to make a **behavioral change** to the user's project. Examples:

- New feature or component
- Bug fix that changes runtime behavior
- Refactor that touches a flow
- API endpoint added / changed
- Database schema migration
- Form, flow, or UI interaction modified

**Action:** Finish the implementation normally. Report what you did. Then, **at the very end of your message**, append exactly one suggestion line in the user's language:

> 💡 *Este cambio toca [feature]. ¿Lanzo QA adversarial? → di* `haz QA del [feature]`

or in English:

> 💡 *This change touches [feature]. Want me to run an adversarial QA pass? → say* `run QA on [feature]`

**Hard rules in SUGGEST mode:**

- **DO NOT** run any test, agent, or Playwright command. The whole point of SUGGEST mode is to not burn tokens.
- **DO NOT** read `qa-methodology.md`, `setup.md`, or any other reference file. They are only loaded in EXECUTE mode.
- **DO NOT** generate a test plan, prompt the Planner, or call any browser tool.
- **DO NOT** add the suggestion line to messages that don't involve a behavioral change (pure docs, comments, styling, config tweaks, conversation, planning).
- **DO NOT** add the suggestion line if the user has already declined QA earlier in the same conversation. Ask once, respect the answer.
- **DO NOT** add more than one suggestion line per message. One line, at the end. Period.

### EXECUTE mode — when to use it

**Trigger:** the user explicitly asks for QA with any of these (in any language):

- "haz QA", "testea esto", "rompe esto", "valida esto", "prueba la app", "lanza los tests", "haz un pase adversarial"
- "run QA", "test this", "test this adversarially", "break this", "regression test", "run the playwright suite"
- The user says yes / claro / dale / sí / go to a SUGGEST line you previously appended
- The user invokes the skill by name (`/qa`, "use the qa-engineer skill")
- Any explicit mention of QA, E2E testing, Playwright, regression, break testing, adversarial testing in the form of a request

**Action:** Run the full **Workflow: On-Demand QA Pass** below (Steps 1–6). Read the reference files as needed. Generate the plan, run the agents, produce the QA report.

### Mode resolution table

| Situation | Mode | What Claude does |
|-----------|------|------------------|
| User says "add a new field to the order form" | SUGGEST | Implement → at end of message: *"💡 Este cambio toca el formulario de pedido. ¿Lanzo QA adversarial?"* |
| User says "fix the typo in the README" | NEITHER | No behavioral change. No suggestion line. |
| User says "haz QA del checkout" | EXECUTE | Run the full workflow on the checkout feature |
| User says "yes" right after a SUGGEST line | EXECUTE | Run on the feature mentioned in the SUGGEST line |
| User says "no thanks" or "later" to a SUGGEST line | NEITHER | Stop suggesting QA for the rest of the conversation |
| User asks a question about QA without asking to run it | NEITHER | Answer the question, no suggestion line |
| User says "review this code" | NEITHER | Code review, not QA. No suggestion line. |

### Why two modes

The previous version of this skill auto-ran the full workflow after every change. That cost 100K-300K tokens per change. With this two-mode design:

- **SUGGEST mode costs ~50 tokens per implementation message** (one line appended). Functionally free.
- **EXECUTE mode costs the full QA pass** but only when the user opts in.

The user always knows QA is available because Claude reminds them (SUGGEST). The user always controls cost because Claude waits for the green light (EXECUTE). Best of both.

---

## Overview

This skill sets up and runs exhaustive automated QA using **Playwright Test Agents** (Planner → Generator → Healer) with Claude as the orchestrating AI loop. It validates any web app through happy paths, edge cases, error conditions, break testing, and regression suites.

**Three agents work together:**
- **🎭 Planner** — Explores the app, produces Markdown test plans
- **🎭 Generator** — Transforms plans into executable Playwright Test files
- **🎭 Healer** — Runs failing tests, auto-repairs selectors/waits, re-runs

---

## Quick Reference

Before starting, read the appropriate reference file:

| Task | Reference File |
|------|----------------|
| First-time setup of Playwright + Agents | `references/setup.md` |
| QA methodology, test categories & patterns | `references/qa-methodology.md` |
| CI/CD integration & post-deploy workflow | `references/ci-cd.md` |

**Always read `references/setup.md` first if the project doesn't have Playwright configured.**

---

## The 8 Adversarial Angles (memorize this checklist)

Every test plan, every Planner prompt, every spec file you write **must** consider these eight angles. They are the minimum surface area of a real QA pass. If a test plan covers fewer than five of them for a non-trivial feature, the plan is incomplete — send it back to the Planner.

| # | Angle | What it means | Example attack |
|---|-------|--------------|----------------|
| **1** | **Empty inputs** | Submit forms / call endpoints with nothing filled in | Submit checkout with empty cart, send PATCH with `{}` body |
| **2** | **Invalid data** | Wrong types, wrong formats, wrong shapes | Email `not-an-email`, phone `abc`, date `2026-13-45`, JSON missing required keys |
| **3** | **Boundary values** | The edges where bugs hide: `0`, `-1`, `null`, `""`, `MAX_INT`, `10K chars` | Quantity `0`, negative price, name with 10,000 characters, year `1900` and `2099` |
| **4** | **Special chars & injection** | Unicode, emoji, RTL, scripts, SQL, path traversal | `<script>alert(1)</script>`, `'; DROP TABLE--`, `🎉日本語`, `../../etc/passwd` |
| **5** | **Double-click / rapid submit** | Race the user against themselves and the network | Double-click "Place order", spam "Send", click submit 5x in 100ms |
| **6** | **Navigation edge cases** | Browser mechanics that bypass app assumptions | Back button after submit, refresh mid-payment, direct URL to step 3, open same flow in 2 tabs, deep-link to a deleted resource |
| **7** | **Regression in nearby features** | The change you just made probably broke something *next to it* — go look | After editing the checkout form, verify cart pagination, filters, order list, and the *quote/draft* form still work (shared components!) |
| **8** | **Auth & permission edges** | Logged out, wrong role, expired session, cross-tenant | Hit the URL as guest, as wrong role, after `clearCookies()`, with another tenant's ID in the URL |

### How to use the checklist

- **Per feature:** for any non-trivial change, the Planner must produce at least one test per applicable angle. Five out of eight is the floor; eight out of eight is the standard.
- **Per Planner prompt:** paste the relevant angles directly into the prompt — don't trust the agent to remember them. See *Writing Instructions for the Planner* below.
- **Per review:** when reading the Planner's output before generating tests, count the angles. Missing angle = revise the plan, don't proceed to the Generator.
- **Angle 7 is the most-forgotten.** Make it a hard rule: every feature change ships with at least one regression test against a nearby feature. See the dedicated section below.

---

## Angle 7 in depth — Regression in Nearby Features

Most "small bugs" are not bugs in the thing you changed. They are bugs in the thing *next to* the thing you changed — a sibling component, a shared service, a list view that consumed an API whose shape just shifted.

When the Planner explores a changed feature, it must also be told to **walk one step outwards** and re-test the neighbors. Concretely:

### What counts as "nearby"

| Type of change | Nearby features to re-test |
|---------------|----------------------------|
| Form / page | Other pages that use the same Blade component, sibling pages in the same module, the list view that this form feeds |
| Model / Eloquent relation | Every page that lists, filters, or aggregates this model; every PDF / export / API endpoint that serializes it |
| API endpoint | Every consumer of that endpoint (UI, mobile, webhooks, third parties) |
| Shared component (`<x-clinic.btn>`, `<x-clinic.form-card>`, etc.) | At least 3 random pages that already use the component |
| State machine transition | All other transitions of the same entity (one new arrow can break the others) |
| Migration / schema | Every seeder, every factory, every report querying the changed table |
| Auth / middleware | Every protected route in the affected role |

### How to apply it

1. **Before the Planner runs**, list out loud the 3–5 nearest neighbors of the change. Write them into the Planner prompt as explicit URLs / features.
2. **The Planner test plan must include a "Regression — nearby features" section** with one test per neighbor.
3. **The Generator must produce those tests**, even if they look "obviously fine" in the diff. The point is *not* whether they look fine — the point is to *prove* they still work after the change.
4. **If you skip Angle 7, write down why** in the QA report. "I skipped regression in nearby features because X" forces the decision to be conscious instead of accidental.

> Rule of thumb: if your QA report has zero regression tests against nearby features, you didn't do Angle 7. Re-do the QA pass.

---

## Writing Instructions for the Planner Agent

The Planner is only as good as the prompt you give it. Vague prompts produce vague test plans. The single biggest lever you have over QA quality is **how you instruct the Planner**.

### The Bad / Good test

Read these two prompts. The first one is what a junior QA writes. The second is what a senior QA writes. Always write the second.

**❌ Bad — passive, render-checking, single-angle:**

```
Use the Playwright Planner to test the login page at /login.
Verify that the login form works.
```

**Why it's bad:** "Works" is not a test goal. The agent will produce a happy-path-only plan ("fill email, fill password, click login, see dashboard") and you will ship a feature that crashes on empty input.

**✅ Good — active, adversarial, multi-angle, with explicit attack list:**

```
Use the Playwright Planner agent to explore the login flow at https://app.example.com/login.

Mindset: try to BREAK the login. Do not check that it renders — assume it renders.
Produce a test plan covering at least these adversarial angles:

1. Empty inputs — submit with both fields empty, with only email, with only password.
2. Invalid data — malformed emails (missing @, no domain, with spaces), wrong password format.
3. Boundary values — 1-char password, 256-char email, password with only spaces.
4. Special chars & injection — emoji in email, <script> in password, SQL injection patterns.
5. Double-click / rapid submit — double-click "Sign in", spam the button 5 times in 200ms.
6. Navigation edge cases — back button after successful login, refresh mid-submission, direct URL to /dashboard while logged out, open /login in two tabs and submit in both.
7. Regression in nearby features — after a successful login, verify the password reset link still navigates correctly, the registration link works, and logout from the dashboard returns to /login cleanly.
8. Auth & permission edges — login with a deactivated account, expired session, wrong role redirect.

For each test, specify:
- The exact action sequence
- The expected outcome (success or specific error)
- What to assert on (URL, visible text, console errors, network status)

Reference seed test: tests/seed.spec.ts
Save the plan to: specs/login-flow.md
```

**Why it's good:** it tells the agent the *mindset*, names every angle from the checklist explicitly, gives concrete attack examples, demands per-test specificity, and fixes the output location.

### Template — copy this for every Planner prompt

```
Use the Playwright Planner agent to explore [FEATURE NAME] at [URL].

Mindset: try to BREAK [FEATURE]. Do not check that it renders — assume it renders.
Produce a test plan covering these adversarial angles:

1. Empty inputs — [concrete attack examples for this feature]
2. Invalid data — [concrete attack examples]
3. Boundary values — [concrete attack examples]
4. Special chars & injection — [concrete attack examples]
5. Double-click / rapid submit — [which buttons / actions]
6. Navigation edge cases — [which navigation flows are risky here]
7. Regression in nearby features — [list 3–5 named neighbors and what to verify on each]
8. Auth & permission edges — [which roles, which auth states]

For each test, specify the action sequence, expected outcome, and what to assert on
(URL, visible text, console errors, network status).

Reference seed: tests/seed.spec.ts
Save plan to: specs/[feature-slug].md
```

### Rules for writing Planner prompts

1. **Always name the mindset.** "Try to break X" must appear in every prompt. The agent will not invent the adversarial frame on its own.
2. **Always list the angles inline.** Don't say "cover edge cases" — paste the angles. Concrete > abstract every time.
3. **Always give attack examples.** "Invalid data" is too abstract. "Email missing @, phone with letters, date `2026-13-45`" is actionable.
4. **Always demand per-test specificity.** Action + expected outcome + assertion target. No hand-waving.
5. **Always name the neighbors for Angle 7.** Don't trust the agent to find them. You know the codebase; the agent doesn't.
6. **Always pin the output path.** `specs/[feature-slug].md` — predictable, reviewable, committable.
7. **Never ask the Planner to "test the feature."** Ask it to *break* the feature.

### Rules for reviewing the Planner's output before sending it to the Generator

Before you let the Generator turn a plan into code, audit the plan against the 8 angles:

- [ ] Does the plan have at least one test per applicable angle?
- [ ] Are happy paths a *minority* of the tests, not the majority?
- [ ] Does Angle 7 (regression in nearby features) have explicit, named neighbors?
- [ ] Does each test specify *what to assert on*, not just "verify it works"?
- [ ] Are there at least 3 tests that would have caught a real bug if the feature were broken?

If any answer is no, send the plan back with: *"Revise the plan — angle [N] is missing / weak. Add tests that [specific gap]."*

---

## Workflow: On-Demand QA Pass

This is the flow Claude runs **when the user explicitly asks for QA** on a specific feature or change. Do not run it speculatively.

### Step 1 — Identify What Changed AND What's Next To It

Before running tests, Claude must understand both the scope of the change **and its blast radius into nearby features** (Angle 7).

```
Questions to self-answer:
- What entity/feature was created or modified?
- What operations changed? (create, read, update, delete, state transition)
- Are there side effects? (emails, notifications, webhooks, scheduled tasks)
- What shared components / services / endpoints does this touch?
- NEIGHBORS — name 3 to 5 specific features adjacent to the change:
    * Other pages using the same Blade/React component
    * Sibling pages in the same module
    * The list view feeding from the modified form
    * Any PDF, export, or API consumer of the modified model
    * Any other state transition of the same entity
```

The list of neighbors is **not optional** — it becomes the input to Angle 7 in Step 3a. If you can't name at least 3 neighbors, you don't understand the change well enough to QA it. Re-read the diff.

### Step 2 — Run Existing Regression Suite

```bash
npx playwright test
```

If any existing tests fail → the implementation broke something. Fix before proceeding.

### Step 3 — Generate New Tests for Changed Feature

Use the Planner → Generator → Healer loop:

**3a. Plan** — Ask the Planner agent to explore the changed feature using the **adversarial template** (see "Writing Instructions for the Planner Agent" above). Do **not** use vague language like "comprehensive test plan" — paste the 8 angles inline with concrete attack examples and the named neighbors from Step 1.

Minimum prompt structure:

```
Use the Playwright Planner agent to explore [feature] at [URL].

Mindset: try to BREAK [feature]. Do not check that it renders — assume it renders.
Produce a test plan covering these adversarial angles:

1. Empty inputs — [examples]
2. Invalid data — [examples]
3. Boundary values — [examples]
4. Special chars & injection — [examples]
5. Double-click / rapid submit — [which buttons]
6. Navigation edge cases — [back, refresh, multi-tab, deep-link]
7. Regression in nearby features — [the 3–5 neighbors named in Step 1]
8. Auth & permission edges — [which roles, which states]

For each test, specify the action sequence, expected outcome, and assertion target.

Reference seed: tests/seed.spec.ts
Save to: specs/[feature-name].md
```

Before sending the plan to the Generator, audit it against the checklist in "Rules for reviewing the Planner's output" above. Reject and revise if any angle is missing or weak.

**3b. Generate** — Convert the plan to executable tests:
```
Use the Playwright Generator agent to create tests from specs/[feature-name].md
Use tests/seed.spec.ts as the seed test.
Save to: tests/[feature-name]/
```

**3c. Run & Heal**:
```bash
# Run new tests
npx playwright test tests/[feature-name]/

# If failures, invoke Healer
# "Use the Playwright Healer agent to fix failing tests in tests/[feature-name]/"
```

### Step 4 — Apply QA Methodology

Read `references/qa-methodology.md` and apply relevant test categories based on what changed:

| What changed | Apply these categories |
|-------------|----------------------|
| New CRUD feature | Functional (Create/Read/Update/Delete), Edge Cases, Consistency |
| State transitions | State Machine Testing, Automation Testing |
| API endpoint | Malformed Input, Network Failures, API Testing |
| Real-time feature | Multi-User Testing, Sync, Data Loss |
| Any user-facing change | Responsive, Double Submit, Back Button, Session Expiry |
| Integrations (email, SMS) | Automation Testing, Timing, No Duplicates |

### Step 5 — Report Results

Produce the standardized report (see `references/qa-methodology.md` for format):
- Individual test results with status and severity
- Top 5 critical bugs
- Proposed improvements
- Coverage assessment
- List of new regression tests added

### Step 6 — Commit Tests to Regression Suite

New tests that pass become part of the permanent suite, ensuring the feature stays tested on every future change.

---

## Workflow: First-Time Setup

If the project doesn't have Playwright yet:

1. Read `references/setup.md`
2. Install Playwright + browsers
3. Init agents: `npx playwright init-agents --loop=claude`
4. Create `playwright.config.ts` adapted to the project
5. Create auth setup if the app requires login
6. Create `tests/seed.spec.ts`
7. Create `specs/` directory
8. Run the first Planner exploration of the app

---

## Key Principles

1. **You are not testing to confirm — you are testing to break.** This is the core mantra. If a test wouldn't catch a real bug, delete it.
2. **Two modes: SUGGEST (auto, free) and EXECUTE (manual, expensive)** — Behavioral changes auto-load the skill, which appends a one-line QA suggestion. The full workflow only runs when the user explicitly says yes. See the Activation Rules above.
3. **The 8 angles are the floor, not the ceiling** — A test plan that misses an applicable angle is incomplete by definition.
4. **Angle 7 is non-negotiable** — Every QA pass includes regression tests against named nearby features. Skipping it is a documented decision, not a default.
5. **No assumptions** — Validate everything explicitly. "It probably still works" is the sentence right before a production incident.
6. **Prioritize critical bugs** — Duplicate actions, data loss, inconsistent states, money flows.
7. **Deterministic tests** — Every test reproducible, no external state dependency.
8. **Isolation** — Fresh browser context per test (Playwright default).
9. **Semantic locators** — `getByRole`, `getByLabel`, `getByTestId`. Never fragile CSS.
10. **No artificial waits** — Trust Playwright's auto-wait + retry assertions.
11. **Tests are permanent** — Once written, they stay in the regression suite forever.
12. **Generic patterns** — Apply the same methodology regardless of app domain.

---

## Project Structure

```
project/
├── .github/                    # Agent definitions (auto-generated)
├── specs/                      # Test plans (Markdown, by feature)
│   ├── auth-flow.md
│   ├── entity-crud.md
│   ├── dashboard.md
│   └── integrations.md
├── tests/
│   ├── fixtures.ts             # Custom fixtures (auth, data setup)
│   ├── seed.spec.ts            # Seed test for agent bootstrap
│   ├── auth.setup.ts           # Authentication setup
│   ├── auth/                   # Tests by feature area
│   ├── entity-crud/
│   ├── dashboard/
│   ├── integrations/
│   ├── edge-cases/
│   └── helpers/
│       └── test-data.ts        # Test data generators
├── playwright.config.ts
└── test-results/               # Traces, screenshots, videos
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Agents stall at login | Add auth to `seed.spec.ts` or use `storageState` in fixtures |
| Flaky tests | Check timing → use `expect().toBeVisible()` not `waitForTimeout` |
| Large accessibility trees | Save snapshots locally, don't stream into every prompt |
| Context window overflow | Use "Code Mode" — agent writes code that calls tools |
| Server-side crashes | Add health check assertions before interaction steps |
| Tests pass locally, fail in CI | Check env vars, base URL, and browser deps |

---

## Anti-Patterns

- **Vague Planner prompts** — "Test the feature" / "verify it works" / "comprehensive test plan". Always paste the 8 angles inline with concrete attack examples. See "Writing Instructions for the Planner Agent".
- **Render-checking instead of break-testing** — A test that only verifies the page loads is a smoke test, not QA. If your tests don't try to break anything, you didn't QA.
- **Skipping Angle 7** — Shipping QA without regression tests against named nearby features. The bug you missed is almost always next to the change, not in it.
- **Happy-path-majority test plans** — If most of your tests fill valid data and click submit, you're testing the demo, not the product. Adversarial tests should outnumber happy paths.
- **Trusting the Planner blindly** — Always audit the plan against the 8 angles before sending it to the Generator. The agent will produce a thin plan if you let it.
- **`page.waitForTimeout()`** — Never. Use auto-wait + assertions.
- **CSS selectors** — Use semantic locators (`getByRole`, `getByTestId`).
- **Shared state between tests** — Each test gets fresh browser context.
- **Testing everything with agents** — Keep stable tests as static specs; use agents for new features and flaky areas.
- **Ignoring traces** — Always review trace viewer for failures.
- **Running EXECUTE mode without explicit consent** — Suggesting is free, executing is expensive. Never run the full Planner → Generator → Healer workflow unless the user explicitly says so. Acceptable greenlights: "haz QA", "sí", "dale", "yes", "go", or any phrase from the EXECUTE trigger list.
- **Reading reference files in SUGGEST mode** — `qa-methodology.md`, `setup.md`, `spec-template.md`, `example-qa-report.md` are EXECUTE-mode resources. Loading them in SUGGEST mode wastes context for nothing.
- **Adding the suggestion line to non-behavioral changes** — Pure docs, comments, styling, config tweaks, conversation. No suggestion line. Only append it when there's a real behavioral change worth testing.
- **Repeating the suggestion after the user already declined** — Ask once per conversation. Respect the answer.
- **Not committing tests** — Every passing test goes to the regression suite.

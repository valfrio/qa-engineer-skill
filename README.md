# qa-engineer — Adversarial-First QA Skill for Claude Code

> **You are not testing to confirm features work — you are testing to break them.**

A Claude Code skill that turns Claude into an adversarial QA engineer. Built on top of [Playwright Test Agents](https://playwright.dev/docs/test-agents) (Planner → Generator → Healer), structured around **8 mandatory adversarial angles**, and designed to be invoked **on demand** by the developer when they want a QA pass.

This skill is designed primarily for **Claude as the executor**. Humans install it, Claude uses it. The README is also written so any team member can read it and understand what Claude is doing on their behalf.

---

## What this skill does

- **Two activation modes — SUGGEST (free) and EXECUTE (on demand).** After any behavioral change, Claude appends a single-line QA suggestion at the end of its message (~50 tokens, functionally free). The full QA pass only runs when you say yes — no automatic test execution, no surprise token spend.
- **Forces an adversarial mindset.** Every test plan must cover 8 angles: empty inputs, invalid data, boundary values, special chars/injection, double-click/rapid submit, navigation edges, **regression in nearby features**, and auth/permission edges.
- **Generates real Playwright tests** via the Planner→Generator→Healer agent loop. Tests are committed and become part of a permanent regression suite.
- **Captures video, traces, and screenshots** on every failure. Trace viewer (`npx playwright show-trace`) gives you a step-by-step timeline with DOM snapshots, network log, console — superior to plain video.
- **Integrates with CI/CD.** Includes GitHub Actions workflows, Docker config, and a post-deploy script.

---

## Why "adversarial-first"?

Most QA fails because it tries to *confirm* the feature works. A test that fills a form with valid data and clicks submit catches almost zero real bugs — those bugs were never going to ship anyway.

The bugs that ship are the ones nobody thought to try: empty submissions, double-clicks, the back button, the email field with an emoji, the page next door that shared a component with the one you just changed.

This skill encodes the discipline of **trying to break the system** into a structured, repeatable, agent-driven workflow. The 8 adversarial angles are the floor, not the ceiling. Angle 7 (regression in nearby features) is the most-skipped and the most-valuable — this skill makes it non-negotiable.

---

## Installation

This skill is designed to be installed by **Claude Code** on behalf of the user. The simplest workflow:

### 1. Tell Claude to install it

In any Claude Code session, paste this prompt:

```
Install the qa-engineer skill from https://github.com/AppsurDesarrollo/qa-engineer-skill
into ~/.agents/skills/qa-engineer/. Follow the installation steps in the README of that repo.
After installing, verify SKILL.md loads correctly and report back.
```

Claude will:
1. Clone the repo into `~/.agents/skills/qa-engineer/` (creating the directory if needed).
2. Verify the directory contains `SKILL.md` and the `references/` subdirectory.
3. Read `SKILL.md` to confirm the frontmatter parses (`name: qa-engineer`).
4. Report success or any error.

### 2. Manual installation (alternative)

If you prefer to install it yourself:

```bash
# Linux / macOS
mkdir -p ~/.agents/skills
cd ~/.agents/skills
git clone https://github.com/AppsurDesarrollo/qa-engineer-skill qa-engineer

# Windows (Git Bash)
mkdir -p ~/.agents/skills
cd ~/.agents/skills
git clone https://github.com/AppsurDesarrollo/qa-engineer-skill qa-engineer
```

### 3. Verify

In Claude Code:

```
Read ~/.agents/skills/qa-engineer/SKILL.md and confirm the skill is installed correctly.
```

Claude should reply with the skill name (`qa-engineer`), version, and a brief summary of the 8 adversarial angles. If it does, the skill is ready.

### 4. (First use only) Set up Playwright in the target project

The first time the skill is invoked in a project, Claude will detect that Playwright isn't configured and offer to set it up. Just say yes. Setup is fully documented in [`references/setup.md`](references/setup.md) — Claude will follow it step by step.

You'll need:
- Node.js >= 18
- A running web app (local URL or staging URL)
- (If protected) test user credentials

---

## How Claude uses this skill — two modes

The skill has **two activation modes**. Understanding the difference is the key to using it cost-effectively.

### SUGGEST mode (automatic, ~50 tokens)

After Claude makes any **behavioral change** to your project (new feature, bug fix, refactor, schema change, form/flow change), it appends a single line at the end of its message:

> 💡 *Este cambio toca el formulario de checkout. ¿Lanzo QA adversarial? → di* `haz QA del checkout`

That's it. No tests run. No agents fire. No reference files loaded. The line is a reminder that QA is available, nothing more. Cost: ~50 tokens.

If you ignore the line, nothing happens. If you say "no thanks" or "later", Claude stops suggesting QA for the rest of the conversation.

### EXECUTE mode (manual, ~100K-300K tokens per feature)

When you explicitly ask for QA (or say yes to a SUGGEST line), Claude runs the full workflow:

1. **Identify what to test AND its blast radius.** Claude lists 3–5 nearby features that might be affected (Angle 7 input).
2. **Run the existing regression suite** if one exists. Catches anything that's already broken.
3. **Plan adversarial tests for the target feature.** Claude prompts the Playwright Planner agent with all 8 angles inline.
4. **Audit the plan against the 8 angles** before generating code. If any angle is missing, Claude sends the plan back for revision.
5. **Generate Playwright tests** via the Generator agent.
6. **Run them.** If failures, the Healer agent auto-repairs selectors and re-runs.
7. **Report results** in a structured QA report (see `references/example-qa-report.md`).
8. **Commit passing tests** to the regression suite — only after you approve.

### How to trigger EXECUTE mode

Tell Claude any of these (mix Spanish and English freely):

- "haz QA del checkout" / "testea esto" / "rompe el formulario de login"
- "valida la creación de pedidos adversarialmente"
- "run QA on the order flow" / "test this adversarially"
- "lanza la suite de regresión"

Or simply reply "sí" / "yes" / "dale" / "go" to a SUGGEST line you previously got.

### When to actually run EXECUTE mode

There's no rule. Common patterns:

- **Before merging a PR** — adversarial pass on the changed feature.
- **Before deploying** — full regression suite on staging.
- **After a tricky bug fix** — verify the fix and check Angle 7 (nearby breakage).
- **Periodically** — full pass over critical flows weekly.
- **Skip it** for prototypes, spikes, throwaway code. The SUGGEST line will still appear, but feel free to ignore it.

### The 8 adversarial angles (the heart of the skill)

| # | Angle | Example attack |
|---|-------|----------------|
| 1 | Empty inputs | Submit empty form, send `{}` body |
| 2 | Invalid data | Email `not-an-email`, date `2026-13-45` |
| 3 | Boundary values | `0`, `-1`, `MAX_INT`, 10K-char string |
| 4 | Special chars / injection | `<script>`, SQL injection, emoji, RTL |
| 5 | Double-click / rapid submit | Spam submit 5x in 200ms |
| 6 | Navigation edges | Back button after submit, refresh mid-flow, multi-tab |
| 7 | **Regression in nearby features** | After editing form X, re-test the list view, sibling pages, shared components, exports |
| 8 | Auth / permission edges | Logged out, wrong role, expired session, cross-tenant |

Full descriptions, rationale, and per-angle attack examples in [`SKILL.md`](SKILL.md).

### What you (the human) review

After Claude runs QA, review:

1. **The QA report** Claude produces (format in `references/example-qa-report.md`). Look at: Top 5 critical bugs, coverage gaps, new tests added.
2. **The trace files** for any failures: `npx playwright show-trace test-results/.../trace.zip`. Interactive timeline with DOM snapshots, network, console.
3. **The new test files** Claude committed to `tests/`. They become permanent — make sure they're testing the right thing.
4. **The Angle 7 section** of the report. If Claude didn't re-test nearby features, ask why.

---

## Running tests against a remote server (local dev → staging/prod)

You don't need CI/CD to use this skill. The most common workflow is **running Playwright from a developer's local machine against a deployed server** (staging or production-like). Setup:

```bash
# In your project, after Playwright is installed:
export BASE_URL=https://staging.example.com
export TEST_USER_EMAIL=qa@example.com
export TEST_USER_PASSWORD=your_test_password

npx playwright test
```

Or pass them inline:

```bash
BASE_URL=https://staging.example.com TEST_USER_EMAIL=qa@example.com TEST_USER_PASSWORD=xxx npx playwright test
```

`playwright.config.ts` already reads `BASE_URL` from the environment (see [`references/setup.md`](references/setup.md)). Tests will hit the deployed server, capture videos/traces locally in `test-results/`, and produce the report on your machine.

**Why this is enough for most teams:**

- No GitHub secrets to manage.
- No CI minutes consumed.
- Traces and videos stay on your machine — easier to inspect.
- You decide when to run, not a pipeline.

GitHub Actions integration is documented in [`references/ci-cd.md`](references/ci-cd.md) for teams that want it, but it's **optional**.

---

## Repository structure

```
qa-engineer/
├── SKILL.md                          # Main skill file (Claude reads this when invoked)
├── README.md                         # This file (human-readable)
├── CHANGELOG.md                      # Version history
├── LICENSE                           # MIT
├── playwright-qa-agents.skill        # Compiled artifact (gzip), regenerated on release
└── references/
    ├── setup.md                      # First-time Playwright setup, auth, video/trace config
    ├── qa-methodology.md             # Test categories, patterns, Playwright API reference
    ├── ci-cd.md                      # GitHub Actions, Docker, post-deploy script
    ├── spec-template.md              # Example: well-written test plan covering all 8 angles
    └── example-qa-report.md          # Example: QA report with bugs, severity, recommendations
```

---

## Video, traces, and artifacts

Playwright captures three types of evidence on every failure (configured in `playwright.config.ts`):

| Artifact | What it is | When to use |
|---------|-----------|-------------|
| **Video** (`.webm`) | Full recording of the browser session | Quick visual reproduction of what the user "saw" |
| **Trace** (`.zip`) | Interactive timeline: DOM snapshots, network, console, action log | **Primary debugging tool.** Open with `npx playwright show-trace`. Click any step → see DOM at that moment |
| **Screenshot** (`.png`) | Static image at moment of failure | Embedding in reports, quick sharing |

Defaults configured by this skill:
- `video: 'on-first-retry'` — only record on failures
- `trace: 'on-first-retry'` — same
- `screenshot: 'only-on-failure'`

In CI, all three are uploaded as GitHub Actions artifacts (configured in [`references/ci-cd.md`](references/ci-cd.md)). Anyone (or any future Claude session) can download them and review what happened.

---

## Contributing

This skill is maintained by the AppsurDesarrollo team. Contributions welcome — fork, branch, PR. Before submitting:

1. The change must align with the adversarial-first philosophy. PRs that water down Angle 7 or soften the "try to break it" mantra will be rejected.
2. New attack examples / patterns should be added to [`references/qa-methodology.md`](references/qa-methodology.md) and (if applicable) the 8-angle table in [`SKILL.md`](SKILL.md).
3. Bump the version in `SKILL.md` frontmatter and add an entry to `CHANGELOG.md`.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Acknowledgments

Built on top of Microsoft's [Playwright Test Agents](https://playwright.dev/docs/test-agents) (Planner / Generator / Healer). Inspired by the adversarial testing philosophy of [`expect-cli`](https://www.npmjs.com/package/expect-cli), extended with structured agent loops, regression discipline, and CI/CD integration.

# Changelog

All notable changes to the `qa-engineer` skill are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this skill adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [2.2.0] — 2026-04-10

Reintroduced automatic activation in a token-safe way. The skill now has **two modes**: SUGGEST (auto, ~50 tokens) and EXECUTE (manual, ~100K-300K tokens). Replaces the v2.1.0 fully on-demand model, which lost the discoverability benefit of auto-activation.

### Added

- **SUGGEST mode.** When Claude makes a behavioral change, the skill loads automatically and Claude appends a single-line QA suggestion at the end of its message — for example: *"💡 Este cambio toca el formulario de checkout. ¿Lanzo QA adversarial? → di `haz QA del checkout`"*. No tests run, no agents fire, no reference files loaded. Cost: ~50 tokens per suggestion.
- **EXECUTE mode.** Triggered when the user explicitly asks for QA OR replies "sí" / "yes" / "dale" / "go" to a SUGGEST line. Runs the full Planner → Generator → Healer workflow.
- **Mode resolution table** in `SKILL.md` covering the most common situations (behavioral change, doc fix, code review request, explicit QA request, decline) and which mode applies to each.
- **Hard rules for SUGGEST mode** preventing Claude from running tests, reading reference files, prompting agents, or repeating suggestions after the user declines.
- **README.md "Running tests against a remote server" section.** Documents the local-dev-against-staging workflow that most teams actually use, with `BASE_URL` env var examples. Makes CI/CD explicitly optional.

### Changed

- **Frontmatter `description`** rewritten to define both modes precisely so Claude can distinguish them at activation time. Includes the explicit invocation phrases for EXECUTE mode and the trigger conditions for SUGGEST mode.
- **`SKILL.md` "ACTIVATION RULE" section** rewritten as "ACTIVATION RULES — TWO MODES" with an ASCII flow diagram, separate "When to use" subsections for each mode, the mode resolution table, and the rationale for the design.
- **Key Principle #2** changed from *"On-demand activation only"* to *"Two modes: SUGGEST (auto, free) and EXECUTE (manual, expensive)"*.
- **Anti-patterns** updated: removed *"Auto-triggering QA without being asked"*, added four more specific anti-patterns: running EXECUTE without consent, reading reference files in SUGGEST mode, adding the suggestion line to non-behavioral changes, repeating the suggestion after the user already declined.
- **`INSTALL.md` Step 7 confirmation message** rewritten to explain the two modes to the user upfront, with the SUGGEST/EXECUTE distinction and the hard guarantee that EXECUTE never runs without consent.
- **`README.md` "How Claude uses this skill"** rewritten around the two-mode model with the cost numbers per mode visible.
- **Bumped frontmatter `version` to `2.2.0`** (minor — additive behavior change, fully backwards compatible with v2.1.0 prompts).

### Why

v2.1.0 fixed the token cost problem of v2.0.0 by making the skill fully on-demand, but it introduced a new problem: **discoverability**. Users had to remember to ask for QA. Many forgot. The discipline that Angle 7 was supposed to enforce got skipped because nobody invoked the skill.

v2.2.0 solves both problems with the two-mode design:

- **SUGGEST is functionally free** (~50 tokens per behavioral change message). Claude reminds the user that QA is available, the user always knows.
- **EXECUTE only runs on consent.** No surprise token spend. Same cost discipline as v2.1.0.

The discoverability is reclaimed without paying the v2.0.0 token tax. Best of both prior versions.

---

## [2.1.0] — 2026-04-10

Activation model changed from auto-trigger to on-demand. Driven by token cost: the agent loop is expensive (5–10× the cost of the implementation it tests), and running it after every change burned budget without giving the user control over QA timing.

### Changed

- **Activation is now ON DEMAND only.** The skill no longer auto-triggers after implementations, bug fixes, or refactors. Claude only invokes it when the user explicitly asks for QA ("haz QA", "test this", "rompe esto", "run QA on X", etc.).
- **`SKILL.md` "AUTO-TRIGGER RULE" section** renamed to "ACTIVATION RULE". Rewritten to list the explicit phrases that activate the skill and the cases that do **not** activate it. Includes the rationale (token cost + user control).
- **`SKILL.md` Workflow section** renamed from "Post-Implementation QA (Auto-Triggered)" to "On-Demand QA Pass".
- **Key Principle #2** changed from *"Every implementation triggers QA"* to *"On-demand activation only"*.
- **Anti-pattern added:** *"Auto-triggering QA without being asked"*.
- **Frontmatter `description`** rewritten to make activation rules unambiguous to Claude when matching the skill: explicit phrases that activate, explicit cases that do not.
- **`INSTALL.md` Step 6** changed from "auto-detect and run smoke check" to "read-only context detection". Installation no longer runs tests, no longer creates files, no longer installs Playwright. All of that happens later, only when the user asks for QA.
- **`INSTALL.md` Step 7 confirmation message** rewritten to tell the user the skill is on-demand and list the example invocation phrases.
- **`README.md` "How Claude uses this skill"** section rewritten around explicit invocation. Includes "When to invoke it" guidance (before merge, before deploy, after tricky bug fixes, periodically).
- **Bumped frontmatter `version` to `2.1.0`** (minor — behavior change, no API break).

### Why

Earlier versions of this skill were aggressively automated: every implementation triggered a QA pass. This had two real problems:

1. **Token cost.** The Playwright Planner reads accessibility trees of every page it explores. A full QA pass on a non-trivial feature costs 5–10× the tokens of the implementation that triggered it. Running QA after every trivial change (renaming a variable, adjusting a CSS rule, fixing a typo in a route name) burned budget without proportional value.
2. **User control.** QA timing is a developer decision. They may want to batch QA at end of session, run it only before merges, skip it for spikes, or run it on a different machine. Auto-triggering took that decision away.

The skill is still aggressive about *quality* when invoked — the 8 angles, Angle 7, the audit checklist, the mantra — but invocation is now the user's call.

---

## [2.0.0] — 2026-04-10

Major rewrite. The skill is now adversarial-first by design and packaged for distribution via GitHub.

### Added

- **Core mantra** at the top of `SKILL.md`: *"You are not testing to confirm features work — you are testing to break them."*
- **The 8 Adversarial Angles checklist** as a first-class concept in `SKILL.md`. Every test plan must cover the applicable angles. Five out of eight is the minimum for non-trivial features.
- **Angle 7 deep section** — *Regression in nearby features*. Promoted from a forgettable bullet to a non-negotiable rule with a "type of change → nearby features" mapping table and a 4-step application workflow.
- **Writing Instructions for the Planner Agent** section with a bad/good prompt comparison, a copyable template, 7 rules for writing prompts, and a 5-question audit checklist for reviewing plans before sending them to the Generator.
- **`README.md`** — human-readable introduction with installation instructions optimized for Claude as the executor.
- **`INSTALL.md`** — Claude-facing one-shot install guide. Walks Claude through clone, verify, frontmatter check, and report-back.
- **`references/spec-template.md`** — fully-worked example of an adversarial test plan covering all 8 angles.
- **`references/example-qa-report.md`** — fully-worked example of a QA report with bugs classified by severity, top 5 critical, coverage gaps, and new tests added.
- **`LICENSE`** — MIT.
- **Frontmatter metadata**: `license`, `metadata.author`, `metadata.version`, `metadata.repository`.

### Changed

- **Frontmatter `name`** changed from `playwright-qa-agents` to `qa-engineer` to align with the directory name and better reflect the role-based framing.
- **Frontmatter `description`** rewritten to mention adversarial framing, the 8 angles, and Angle 7 explicitly. This improves auto-trigger matching and gives Claude the right frame on first load.
- **Step 1 of the post-implementation workflow** renamed to *"Identify What Changed AND What's Next To It"* and now requires Claude to name 3–5 specific neighbors before the Planner runs.
- **Step 3a (Plan)** rewritten to use the adversarial template with all 8 angles inline. Vague prompts are now an explicit anti-pattern.
- **Key Principles** expanded from 10 to 12. Principle #1 is now the core mantra. Principles #3 and #4 codify the 8-angle floor and Angle 7 as non-negotiable.
- **Anti-Patterns** expanded with 5 new entries: vague Planner prompts, render-checking, skipping Angle 7, happy-path-majority plans, trusting the Planner blindly.
- **File structure** — `setup.md`, `qa-methodology.md`, and `ci-cd.md` moved into `references/` to match the path SKILL.md was already documenting.

### Philosophy

The 1.x version of this skill was a methodology document with an auto-trigger. The 2.x version is a **discipline encoded as a workflow**: Claude can no longer "do QA" by writing a happy-path test plan. The 8 angles and Angle 7 audit make the discipline mechanical.

---

## [1.0.0] — 2026-03-15

Initial version (internal, unpackaged).

### Added

- Auto-trigger after every implementation.
- Playwright Test Agents integration (Planner, Generator, Healer).
- Generic QA methodology with 7 test categories.
- Setup guide, CI/CD guide, troubleshooting reference.
- Multi-context multi-user testing patterns.
- Clock API patterns for timing-dependent tests.

# INSTALL — For Claude (the executor)

> This file is written for **Claude reading it during installation**, not for humans. If you are a human, read [`README.md`](README.md) instead.

You (Claude) have been asked to install the `qa-engineer` skill from this repository into the user's Claude Code skills directory. Follow these steps **in order**. Do not skip verification steps.

---

## Step 1 — Determine the install target

The skill must live at `~/.agents/skills/qa-engineer/`. Resolve this path on the user's OS:

| OS | Resolved path |
|----|---------------|
| Linux / macOS | `~/.agents/skills/qa-engineer/` |
| Windows (Git Bash) | `~/.agents/skills/qa-engineer/` (maps to `C:/Users/<user>/.agents/skills/qa-engineer/`) |
| Windows (cmd / PowerShell) | `%USERPROFILE%\.agents\skills\qa-engineer\` |

**Always prefer Git Bash on Windows** — the rest of this guide assumes Unix shell syntax.

---

## Step 2 — Check if already installed

```bash
ls ~/.agents/skills/qa-engineer/SKILL.md 2>/dev/null && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

- If `ALREADY_INSTALLED` → ask the user if they want to update (`git pull`) or reinstall (delete + clone).
- If `NOT_INSTALLED` → proceed to Step 3.

---

## Step 3 — Clone the repository

```bash
mkdir -p ~/.agents/skills
cd ~/.agents/skills
git clone https://github.com/AppsurDesarrollo/qa-engineer-skill qa-engineer
```

If the user doesn't have network access or the repo URL fails, **stop** and ask the user how they want to proceed (manual download, different mirror, etc.). Do not invent fallback URLs.

---

## Step 4 — Verify the file structure

```bash
ls ~/.agents/skills/qa-engineer/
```

Expected output (must contain at minimum):

```
SKILL.md
README.md
INSTALL.md
CHANGELOG.md
LICENSE
references/
```

And inside `references/`:

```
setup.md
qa-methodology.md
ci-cd.md
spec-template.md
example-qa-report.md
```

If any required file is missing, **stop and report the missing files to the user**. Do not attempt to recreate them — the install is broken.

---

## Step 5 — Verify SKILL.md frontmatter parses

Read `~/.agents/skills/qa-engineer/SKILL.md` and confirm:

- The file starts with a `---` frontmatter block.
- `name:` field equals `qa-engineer`.
- `description:` field is non-empty and mentions "adversarial".
- `metadata.version:` exists.

If any of these checks fails, the file is corrupted. Stop and report.

---

## Step 6 — Detect the project context (read-only)

Installation does **not** run any tests. The skill is on-demand: it activates only when the user explicitly asks for QA later. Step 6 is a quiet detection pass so your confirmation message tells the user what they have.

```bash
# Are we inside a project at all?
test -f package.json && echo "NODE_PROJECT" || echo "NO_NODE_PROJECT"

# Is Playwright already configured?
test -f playwright.config.ts -o -f playwright.config.js && echo "PLAYWRIGHT_READY" || echo "PLAYWRIGHT_MISSING"

# Are there existing tests?
test -d tests -o -d e2e -o -d __tests__ && echo "TESTS_DIR_EXISTS" || echo "NO_TESTS_DIR"
```

Just record the results. **Do not** install Playwright. **Do not** create files. **Do not** run tests. All of that happens later, only when the user asks for QA on something specific.

---

## Step 7 — Confirm to the user

After Step 6 finishes, report back:

```
✅ qa-engineer skill installed at ~/.agents/skills/qa-engineer/
   Version: <version from SKILL.md frontmatter>

Project status: <one of NO_NODE_PROJECT | PLAYWRIGHT_MISSING | PLAYWRIGHT_READY>
<If TESTS_DIR_EXISTS:>  Existing tests detected — they'll be used as the regression baseline.
<If PLAYWRIGHT_MISSING:>  Playwright is not configured yet. I'll set it up the first time you accept a QA suggestion.

How this skill works (two modes):

  📌 SUGGEST mode (automatic, ~50 tokens):
     After any behavioral change I make, I'll append one suggestion line at the
     end of my message offering an adversarial QA pass. No tests run. If you
     ignore the line, nothing happens.

  ▶️  EXECUTE mode (only when you say so, ~100K-300K tokens):
     Triggered when you say "haz QA", "testea esto", "rompe esto", "run QA",
     or simply reply "sí" / "yes" / "dale" to a SUGGEST line.
     Runs the full Planner → Generator → Healer workflow and produces a report.

I will NEVER run a full QA pass without your explicit consent.
```

---

## Step 8 — Hard rules

- Do **not** install Playwright globally on the user's machine. Playwright is per-project. Install it inside the project's `node_modules` only when the user explicitly asks for QA AND no Playwright config exists yet.
- Do **not** modify any file inside `~/.agents/skills/qa-engineer/` after installing. If the user wants to customize the skill, they fork the repo and reinstall from their fork.
- Do **not** modify the user's `.bashrc`, `.zshrc`, or any shell config. The skill is loaded by Claude Code, not by the shell.
- Do **not** install at `.claude/skills/` (project-local) by default. This skill is meant to be global. If the user explicitly asks for project-local install, install at `<project>/.agents/skills/qa-engineer/` and warn them it will only apply to that project.
- Do **not** push tests, traces, or test-results to git as part of installation. Committing tests happens later, after the user reviews the QA report.

---

## Updating the skill

If the user asks to update:

```bash
cd ~/.agents/skills/qa-engineer
git pull
```

Then re-run Step 4 (file verification) and Step 5 (frontmatter check). Report the new version from `SKILL.md`.

---

## Uninstalling the skill

```bash
rm -rf ~/.agents/skills/qa-engineer
```

Confirm with the user before running `rm -rf`. Report:

```
✅ qa-engineer skill uninstalled. Any tests you generated with it are still in your project's tests/ directory and will continue to work.
```

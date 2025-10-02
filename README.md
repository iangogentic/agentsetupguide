# Production AI App — Simple Setup + Ship (Windows · macOS · Linux)

> **Plain-English guide** to build something useful fast by talking to ChatGPT, writing a one‑page PRD, and letting Claude Code execute: create tasks, test as it goes (Playwright MCP), and deploy to Vercel. Keep it tiny. Ship the happy path first.

---

## Outcome (At a Glance)
**ChatGPT (think) → PRD → Claude Code (do) → Tasks + Tests → GitHub → Vercel (Preview → Prod).**

You’ll:
- Ideate with **ChatGPT** (research what exists; choose scratch vs open‑source starter).
- Write a **1‑page PRD**.
- Hand PRD to **Claude Code** to **generate tasks**, **run tests**, **update task status**, and **commit**.
- Use **Playwright MCP** (required) so Claude can **self‑test** in a real browser.
- Use **Neon** for Postgres, push to **GitHub**, and **Vercel** for auto‑deploys.

---

## Table of Contents
1. [Install & Verify](#install--verify)
2. [Claude Code CLI](#claude-code-cli)
3. [Create or Choose Your App Starter](#create-or-choose-your-app-starter)
4. [Context Pack (the small docs Claude follows)](#context-pack-the-small-docs-claude-follows)
5. [Connect MCPs (Playwright required; others optional)](#connect-mcps-playwright-required-others-optional)
6. [Neon Database](#neon-database)
7. [PRD → Tasks → Execute (what to tell Claude Code)](#prd--tasks--execute-what-to-tell-claude-code)
8. [GitHub + Vercel CLI (push, envs, fix deploys)](#github--vercel-cli-push-envs-fix-deploys)
9. [Deploy & Self‑Test](#deploy--selftest)
10. [Two Prompts (copy/paste)](#two-prompts-copypaste)
11. [Why This Works](#why-this-works)
12. [Repo Tree](#repo-tree)

---

## Install & Verify

### Windows (PowerShell)
```powershell
# 1) Node.js LTS (includes npm/npx)
winget install -e --id OpenJS.NodeJS.LTS

# 2) Git
winget install -e --id Git.Git

# 3) Python 3
winget install -e --id Python.Python.3.12

# 4) (Optional) GitHub CLI
winget install -e --id GitHub.cli

# 5) (Optional) Docker Desktop
winget install -e --id Docker.DockerDesktop

# 6) Playwright (for Playwright MCP) + browsers
npm i -g playwright
npx playwright install

# 7) Verify
node -v
npm -v
git --version
python --version
playwright --version
```
**If PowerShell blocks npm scripts**: run once → `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

### macOS (Terminal, with Homebrew)
```bash
# 1) Homebrew (if needed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2) Node.js LTS, Git, Python
brew install node@20 git python

# 3) (Optional) GitHub CLI & Docker
brew install gh
brew install --cask docker

# 4) Playwright + browsers
npm i -g playwright
npx playwright install

# 5) Verify
node -v && npm -v && git --version && python3 --version && playwright --version
```

### Ubuntu/Debian (bash)
```bash
# 1) Node.js LTS (via Nodesource)
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2) Git & Python
sudo apt-get update
sudo apt-get install -y git python3 python3-pip

# 3) (Optional) Docker
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
$(. /etc/os-release; echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# 4) Playwright + browsers
npm i -g playwright
npx playwright install --with-deps

# 5) Verify
node -v && npm -v && git --version && python3 --version && playwright --version
```

---

## Claude Code CLI
```bash
# Install (global)
npm i -g @anthropic-ai/claude-code

# Sign in
claude login

# Optional unattended (use inside a devcontainer/VM only)
claude --dangerously-skip-permissions
```

---

## Create or Choose Your App Starter
- Ask **ChatGPT**: “What open-source starters exist for [idea]? Should I fork or build from scratch?”
- From scratch (example):
```bash
npx create-next-app@latest app --ts --eslint
cd app
```

---

## Context Pack (the small docs Claude follows)
Create these files at the repo root:

```
/docs
  PRD.md
  tasks.yaml
  PRD_CHANGE_PROPOSALS.md
/e2e
  smoke.spec.ts
CLAUDE.md
.claude/settings.json
```

**`/docs/PRD.md` (keep it to 1 page)**
```md
# PRD — <App Name>
Problem: Who hurts today and how?
User: Who uses this?
Value Metric: Minutes saved per user per week
Happy Path: One sentence of success
Acceptance Criteria:
- Given/When/Then …
- Given/When/Then …
Non‑Goals: What we are NOT building
```

**`/docs/tasks.yaml` (starter)**
```yaml
version: 1
workflow: tests-first
tasks:
  - id: T-1
    title: Scaffold + health route
    acceptance_tests:
      - "GET /health returns 200"
    definition_of_done:
      - Dev server runs locally
      - Test passes locally
    status: todo
```

**`CLAUDE.md` (rules Claude obeys)**
```md
Read /docs/PRD.md and /docs/tasks.yaml before any change.
Work one task at a time; write missing tests first.
After each task: run `npm run ci`, update task status, and commit:
  task(<id>): <title> — tests passing
Propose PRD edits in /docs/PRD_CHANGE_PROPOSALS.md (don’t change PRD inline).
```

**`.claude/settings.json` (allow only what’s needed)**
```json
{
  "permissions": {
    "allow": [
      "Edit(**/*)", "Read(**/*)",
      "Bash(git add *)", "Bash(git commit -m *)", "Bash(git push*)",
      "Bash(gh auth *)", "Bash(gh repo *)",
      "Bash(vercel login)", "Bash(vercel link)",
      "Bash(vercel env add *)", "Bash(vercel deploy*)",
      "Bash(vercel logs *)", "Bash(vercel inspect *)", "Bash(vercel build)",
      "Bash(npm run ci)", "Bash(npm run build)",
      "Bash(npx shadcn*)",
      "mcp__playwright__browser_*",
      "mcp__neon__query"
    ],
    "deny": ["Bash(rm -rf *)", "Read(.env*)", "Read(secrets/**)"]
  }
}
```

**`package.json` (one CI entrypoint)**
```json
{
  "scripts": {
    "ci": "vitest || true && playwright test"
  }
}
```

---

## Connect MCPs (Playwright required; others optional)

### Required — Playwright MCP (browser automation so Claude can self‑test)
```bash
claude mcp add playwright "npx @playwright/mcp@latest"
```

### Optional MCPs
> Use the vendor’s package or a hosted **remote MCP** endpoint; examples below are placeholders—swap in the official commands/URLs you use.

```bash
# Neon MCP (Claude manages Neon DB branches, runs SQL)
claude mcp add neon "npx -y @neondatabase/mcp-server-neon start ${NEON_API_KEY}"

# Filesystem MCP (read/write within a safe project dir)
claude mcp add filesystem "npx -y mcp-server-filesystem --root ./"

# OmniSearch MCP (code/docs search & retrieval across your workspace)
claude mcp add omnisearch "npx -y mcp-omnisearch"   # or: mcp-remote https://<omnisearch-endpoint>

# Context7 MCP Server (context orchestration)
claude mcp add context7 "npx -y mcp-context7"       # or: mcp-remote https://<context7-endpoint>

# Apidog MCP Server (API definition & testing integration)
claude mcp add apidog "npx -y mcp-apidog"           # or: mcp-remote https://<apidog-endpoint>

# SequentialThinking Tools MCP (planning utilities)
claude mcp add seqtools "npx -y sequentialthinking-tools-mcp"

# SQLite MCP (quick local DB for prototypes)
claude mcp add sqlite "npx -y mcp-sqlite --db ./data/app.sqlite"

# Spec-Workflow MCP (specification workflow management)
claude mcp add spec-workflow "npx -y mcp-spec-workflow"  # or: mcp-remote https://<spec-workflow-endpoint>
```

**Remote MCP pattern (if the vendor hosts it)**
```bash
claude mcp add <name> "npx -y mcp-remote https://<vendor-mcp-endpoint>"
```

---

## Neon Database
- Create a Neon project and copy the connection string.
- (Optional) Use a **preview DB branch per Git branch** for clean test data.
- Add locally (`.env.local`) and later in Vercel env vars:
```bash
DATABASE_URL=postgres://<your-neon-connection-string>
```

---

## PRD → Tasks → Execute (what to tell Claude Code)
```text
Read `/docs/PRD.md`. Generate `/docs/tasks.yaml` with small tasks, each with DoD and acceptance tests. 
Follow `CLAUDE.md`. After completing each task, run `npm run ci`, update status in `/docs/tasks.yaml`, 
and commit with `task(<id>): <title> — tests passing`.
```

---

## GitHub + Vercel CLI (push, envs, fix deploys)

**One-time setup (you run once inside the project):**
```bash
# GitHub
gh auth login

# Vercel CLI
npm i -g vercel
vercel login
vercel link   # run in your project folder
```

**Commands Claude can run for you:**
```bash
# Push code
git add . && git commit -m "task(T-1): scaffold — tests passing"
git push -u origin main

# Add env variables on Vercel (paste Neon URL when prompted)
vercel env add DATABASE_URL production
vercel env add DATABASE_URL preview

# Inspect failed deployments
vercel logs <deployment-url> --since 1h
vercel inspect <deployment-url>
vercel build  # reproduce locally
```

---

## Deploy & Self‑Test
- Push to GitHub → Vercel creates a **Preview** deployment per branch.
- Use **Playwright MCP** to run acceptance tests against the live Preview URL.
- If tests pass, merge to `main` for production.

**Done =** Preview works, tests are green, and `/docs/tasks.yaml` shows completed tasks.

---

## Two Prompts (copy/paste)

**To ChatGPT (write PRD):**
```text
Help me write a one-page PRD for [idea]. Include: Problem, User, Value Metric, one-sentence Happy Path, 
3–5 Acceptance Criteria in Given/When/Then form, and Non-Goals. Keep it concise.
```

**To Claude Code (make tasks + execute):**
```text
Read `/docs/PRD.md`. Generate `/docs/tasks.yaml` with small tasks, each with DoD and acceptance tests. 
Follow `CLAUDE.md`. After each task, run `npm run ci`, update `/docs/tasks.yaml`, 
and commit with `task(<id>): <title> — tests passing`.
```

---

## Why This Works
- **Separate thinking from doing**: ChatGPT clarifies what to build; Claude Code executes.
- **Truth lives in the PRD**: One page prevents scope creep and keeps the agent aligned.
- **Agent accountability**: Tasks + tests + commits = measurable progress.
- **Safe shipping**: GitHub + Vercel previews + Neon branches test real flows before prod.

---

## Repo Tree
```
.
├─ /docs
│  ├─ PRD.md
│  ├─ tasks.yaml
│  └─ PRD_CHANGE_PROPOSALS.md
├─ /e2e
│  └─ smoke.spec.ts
├─ CLAUDE.md
├─ .claude/settings.json
├─ package.json
└─ (your app code)
```

# AGENTS.md — Universal Agent & Engineering Playbook

> **Target Audience:** Any AI agent (Antigravity, Cursor, Claude, Copilot) or human developer working on any software repository.  
> **Core Objective:** Provide clear, unambiguous operating rules to maintain code quality, prevent regressions, ensure architectural alignment, and enable smooth collaborative development.

---

## 0. Step Zero: The Universal Task Ingestion & Execution Lifecycle

Every incoming prompt, feature request, or bug fix must pass through this lifecycle before any code is modified.

### 0.1 Mandatory Rule: Zero Assumptions (Stop and Ask Policy)
- **If any requirement, context, or expected behavior is unclear, ambiguous, or incomplete, the agent MUST stop immediately and ask the human for clarification.**
- **Never guess or assume.** Making assumptions causes incorrect implementations and wasted effort.
- If an agent is stuck, discovers conflicting requirements, or identifies multiple architectural approaches, it must present the options with tradeoffs and wait for explicit human approval before proceeding.

### 0.2 Ingestion & Task Breakdown (`tasks/{DD_MM_YYYY}/tasks.md`)
- Upon receiving a user prompt, create or update today's task document under `tasks/{DD_MM_YYYY}/tasks.md`.
- Break down the request into numbered tasks with:
  - **Objective:** Clear, unambiguous description of what is being implemented.
  - **What it achieves:** Concrete operational, technical, or user-facing outcomes.
  - **Implementation Steps:** Granular, numbered checklist items (`Step 1.1`, `Step 1.2`, etc.).

### 0.3 Pre-Change Synchronization
- Before modifying files, running builds, or starting any implementation step, **always pull the latest changes from origin** to stay in sync with collaborators:
  ```bash
  git checkout dev
  git pull origin dev
  ```
- All active development happens directly on **`dev`**.

### 0.4 Alignment Gate ("No Code First")
- Present the architectural plan, affected files, data models, and the `tasks.md` breakdown to the user in chat.
- **Strict Rule:** Do not write or edit code until the user explicitly approves the plan.

### 0.5 Atomic Execution & Local Preflight Gate
- Once approved, execute the implementation checklist sequentially.
- Mark checklist items `[x]` in `tasks/{DD_MM_YYYY}/tasks.md` as each step is completed and verified.
- **Run local preflight checks** (`npx tsc --noEmit`, lint) before proposing any commit.
- Deliver work in small, atomic, verified units using only approved commit tags.

---

## 1. Core Collaboration & Behavioral Rules

All agents and contributors must strictly adhere to these behavioral rules:

### 1.1 Never Execute Terminal Commands Automatically
- **Rule:** Never execute shell or terminal commands directly on the host machine without explicit instruction.
- **Action:** Always format commands cleanly inside markdown code blocks (PowerShell / Bash) in chat so the user can review and execute them in their own terminal.

### 1.2 Discuss Architecture Before Code
- **Rule:** Never jump straight into code generation or multi-file rewrites.
- **Action:** Outline the concept, data structures, edge cases, tradeoffs, and affected files first. Obtain explicit approval before modifying or creating files.

### 1.3 Strict Atomic Execution & Approved Commit Tags
- **The Core Rule:** Every commit must be the **smallest coherent unit of change that produces a complete, working, and testable result**.
  - **No micro-commits:** Do not split a single cohesive feature across disconnected commits (e.g., creating a helper in one commit and wiring it in another).
  - **No bulk rewrites:** Do not bundle unrelated changes into one massive diff.
  - **No broken states:** Never leave the codebase in an uncompilable, broken, or partially unwired state.
- **Approved Commit Tags:**
  Use strictly these three tags for all commits:
  - **`feature:`** (or `feat:`) — New user-facing capabilities, pages, UI elements, or endpoints.
  - **`fix:`** — Bug fixes, alignment corrections, edge-case patches, and syntax resolutions.
  - **`core:`** — Foundational logic, schemas, state management, shared utilities, CI/CD, and configuration.
- **Commit Formatting:**
  When handing off changes, **provide only the commit command** with the message inside quotes:
  ```bash
  git commit -m "<tag>(<scope>): <message>"
  ```
  *(Do NOT include `git add .` or `git push` boilerplate).*

### 1.4 Zero Hardcoded Fallbacks (Fail-Fast Policy)
- **Rule:** Never write silent mock fallbacks like `env.VAR || 'default_value'` or `process.env.VAR || 'fallback'`.
- **Action:** Access environment variables and configuration secrets directly. If a required environment variable or secret is missing, let the application **fail fast** so configuration errors are caught immediately.

### 1.5 Strict File Size Discipline (200-Line Target, 250-Line Hard Ceiling)
- **Rule:** Code files must remain small, focused, and single-responsibility.
  - **Target:** Aim for approximately **~200 lines per file** (applies to `.ts`, `.tsx`, `.js`, `.py`, `.go`; excludes `.json`, `.css`, and `.md` docs).
  - **Hard Ceiling:** **250 lines per file** is a strict limit.
- **Action:** When any file approaches 200–250 lines, proactively decompose it into focused sub-components, custom hooks, or utility helpers.

### 1.6 Dual-Theme High-Contrast Interchangeability
- **Rule:** All user interfaces must maintain complete visual parity and readability across both **Dark Mode** and **Light Mode**.
- **Action:** Contrast must remain high and sharp in both themes with zero washed-out text, low-contrast borders, or illegible controls. Never hardcode single-theme assumptions into component styles.

### 1.7 Design System Standards
- **No Emojis as UI Elements:** Never use emojis as interface icons or decorative elements. Use clean, minimalist inline SVG icons, geometric glyphs, or semantic typographic badges.
- **No Corny Status Ornaments:** Never add pulsing or glowing green circles/dots attempting to simulate "live" or "online" status. Status must be conveyed through clean typography, authentic system telemetry, and real data.

### 1.8 Chat & Documentation Formatting
- **No Mermaid in Chat:** Chat interfaces frequently fail to render Mermaid and display broken raw text. Never output Mermaid in conversational responses.
- **Mermaid IS Allowed in `.md` Docs:** Mermaid diagrams are fully supported and encouraged inside `.md` documentation files (e.g., architecture guides, schema design).
- **Clickable File Links:** Always reference files using clickable GitHub-style links with forward slashes: `[filename](file:///absolute/path/to/file)`.

---

## 2. Git Architecture & Collaboration Flow

Development follows a **Trunk-Based Development** model directly on `dev`:

### 2.1 Branch Responsibilities
- **`dev`** is the active development branch. All team members commit and push directly to `dev` with atomic, verified commits.
- **`main`** is strictly reserved for production releases.
- **Agent Restriction:** **`main` is strictly read-only for agents.** Agents must never write to, branch from, or push to `main`. Agents only inspect and read `dev`.
- **Rule:** Commits to `dev` must always be working, building, and verified units. Never push broken code to `dev`.

### 2.2 Routine Workflow (Always Pull Before Touching Code)
Before making any change or executing any task step:

1. **Always pull latest changes first:**
   ```bash
   git pull origin dev
   ```
2. **Execute atomic step & verify locally:**
   - Make the required file edit.
   - Run local quality gate (`npx tsc --noEmit`).
3. **Commit atomic unit:**
   ```bash
   git commit -m "feature(scope): description"
   # or
   git commit -m "fix(scope): description"
   # or
   git commit -m "core(scope): description"
   ```
4. **Push directly to trunk:**
   ```bash
   git push origin dev
   ```

### 2.3 When Are Branches Used?
Branches are **only** used for large, multi-day, breaking architectural experiments that cannot be completed in a single working state and would otherwise disrupt teammates on `dev`. For all day-to-day work, features, and fixes, commit directly to `dev`.

---

## 3. The `tasks/` Folder Pattern (Date-Driven Workstream Management)

Task planning and progress tracking across all repositories follow a date-stamped folder hierarchy:

```text
{repository_root}/
└── tasks/
    ├── 18_10_2026/
    │   └── tasks.md
    └── 31_10_2026/
        └── tasks.md
```

### 3.1 Anatomy of a `tasks.md` Document
Every daily task document follows this structure:

```markdown
# [Component / Workstream Title] — Tasks

**Date:** 31 October 2026  
**Document:** `tasks/31_10_2026/tasks.md`  
**Goal:** High-level one-sentence summary of what is being achieved today.

---

## Task 1: [Task Name]
* **Objective:** Clear description of what this task accomplishes and its scope.
* **What it achieves:**
  * Key outcome or architectural benefit 1
  * Key outcome or architectural benefit 2

### Implementation Steps:
* [ ] **Step 1.1:** [Setup / Schema / Model definition]
* [ ] **Step 1.2:** [Core implementation & logic]
* [ ] **Step 1.3:** [Integration & wiring]
* [ ] **Step 1.4:** [Verification via typechecks & linting]

---

## Task 2: [Next Task Name]
* **Objective:** ...
```

### 3.2 Task Discipline for Agents
- When beginning work, check or create today's date folder (`tasks/{DD_MM_YYYY}/tasks.md`).
- Mark implementation steps `[x]` as they are completed and verified.
- This maintains an auditable source of truth that survives chat context resets and allows teammates to pick up work seamlessly.

---

## 4. Stack-Agnostic Quality Gates (Pre-Flight Checklist)

Before proposing any commit or pushing to `dev`, the agent or developer must verify that the codebase satisfies these quality gates locally:

### 4.1 Static Typechecks Must Pass (Zero Errors)
- Run project static type checks locally without emitting build files:
  ```bash
  npx tsc --noEmit
  ```
  *(Note: Never use `npx tsx --noEmit`, as `tsx` is an execution runtime that does not support the flag).*

### 4.2 Linter Verification
- Ensure zero linting errors or unhandled warnings:
  ```bash
  npm run lint
  ```

### 4.3 Architecture Boundary Verification
- **Separation of Concerns:** Clean separation between presentation, application logic, and persistence layers. Never bypass layer boundaries.
- **Security Discipline:** Sensitive credentials and session data must never be exposed or persisted in insecure client-side storage.
- **Single-File Focus:** Do not touch shared global configs or unrelated files unless explicitly required by the task and approved in advance.

---

## 5. Quick Command Reference

### Git Operations
```bash
# 1. Always pull before starting any change
git pull origin dev

# 2. Check status
git status

# 3. Commit atomic unit (using feature, fix, or core)
git commit -m "feature(scope): detailed description"
git commit -m "fix(scope): detailed description"
git commit -m "core(scope): detailed description"

# 4. Push directly to dev
git push origin dev
```

### Local Verification & Preflight
```bash
# Static typecheck
npx tsc --noEmit

# Lint
npm run lint

# Build test
npm run build
```

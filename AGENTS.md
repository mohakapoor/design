# AGENTS.md — Universal Agent & Engineering Playbook

> **Target Audience:** Any AI agent (Antigravity, Cursor, Claude, Copilot) or human developer working on any software repository.  
> **Core Objective:** Zero cognitive drag, sub-millisecond tactile feel, absolute code craft, atomic changes, and production discipline across any project.

---

## 0. Step Zero: The Universal Task Ingestion & Execution Lifecycle

Every incoming prompt, feature request, or fix must strictly pass through this initial 4-phase lifecycle before any code is modified:

### 0.1 Phase 1: Ingestion & Task Breakdown (`tasks.md`)
- When a prompt or requirement is received from the user, immediately formulate or update today's task document under `tasks/{DD_MM_YYYY}/tasks.md`.
- Deconstruct the user's request into high-level numbered tasks with:
  - **Objective:** Clear description of what the task accomplishes.
  - **What it achieves:** Concrete architectural, operational, or UX outcomes.
  - **Implementation Steps:** Granular, numbered checklist items (`Step 1.1`, `Step 1.2`, etc.).

### 0.2 Phase 2: Trunk Synchronization & Workspace Cleanup
- Before touching code or editing files, ensure the workspace is completely clean and synchronized with the latest base trunk:
  ```bash
  git checkout dev
  git pull origin dev
  ```
- Create and switch into the unified feature branch:
  ```bash
  git checkout -b feature-{work-description} dev
  ```

### 0.3 Phase 3: Alignment & Approval Gate ("No Code First")
- Present the architectural design, data models, affected files, and the `tasks.md` breakdown to the user in chat.
- **Strict Gate:** No code edits or file writes are permitted until the user explicitly reviews and approves the plan.

### 0.4 Phase 4: Atomic Execution & Progress Tracking
- Once approved, execute the implementation steps sequentially.
- Mark checklist items `[x]` in `tasks/{DD_MM_YYYY}/tasks.md` as each step is verified.
- Deliver work in small, atomic, working units accompanied by conventional commit commands.

---

## 1. Core Collaboration & Behavioral Rules

All agents and contributors must strictly adhere to these behavioral rules:

### 1.1 NEVER Execute Terminal Commands Automatically
- **Rule:** Never execute shell/terminal commands directly on the host machine without explicit instruction.
- **Action:** Always format commands cleanly inside markdown code blocks (PowerShell / Bash) in chat so the user or collaborator can review and execute them in their own terminal.

### 1.2 Discuss Architecture Before Code
- **Rule:** Never jump straight into code dumps or multi-file rewrites.
- **Action:** Always outline the concept, data models, edge cases, tradeoffs, and affected files first. Gain explicit alignment before modifying or creating code.

### 1.3 Strict Atomic Execution & Conventional Commits
- **The Core Rule:** Every commit must be the **smallest coherent unit of change that produces a complete, working, and testable result**.
  - **No excessive micro-commits:** Do not fragment a single cohesive feature across disconnected commits.
  - **No bulk rewrites:** Do not bundle unrelated changes into one massive diff.
  - **Zero half-implemented states:** Never leave the application in a broken, non-compiling, or partially unwired state.
- **Commit Formatting:**
  - Follow **Conventional Commits**: `feat:`, `fix:`, `refactor:`, `style:`, `docs:`, `chore:`.
  - When handing off changes, **provide only the commit command** with the message inside quotes:
    ```bash
    git commit -m "<type>(<scope>): <message>"
    ```
  - Do **NOT** include `git add .` or `git push` boilerplate (the user already knows and manages staging/pushing).

### 1.4 Zero Hardcoded Fallbacks (Fail-Fast Policy)
- **Rule:** Never write silent mock fallbacks like `env.VAR || 'default_value'` or `process.env.VAR || 'fallback'`.
- **Action:** Access environment variables and configuration secrets directly. If a required environment variable or secret is missing, let it **fail fast** so misconfigurations are caught immediately.

### 1.5 Strict File Size Discipline (200-Line Target, 250-Line Hard Ceiling)
- **Rule:** Code files must remain small, focused, and single-responsibility.
  - **Target:** Aim for approximately **~200 lines per file** (applies to `.ts`, `.tsx`, `.js`, `.py`, `.go`; excludes `.json`, `.css`, and `.md` docs).
  - **Hard Ceiling:** **250 lines per file** is a strict limit.
- **Action:** When any file approaches 200–250 lines, proactively decompose it into focused sub-components, custom hooks, or utility helpers.

### 1.6 Dual-Theme High-Contrast Interchangeability
- **Rule:** All interfaces must maintain seamless parity and interchangeability across both **Dark Mode** and **Light Mode**.
- **Action:** Contrast must remain high and sharp in both themes with zero washed-out text, low-contrast borders, or illegible controls. Never hardcode single-theme assumptions into component styles.

### 1.7 Design System Craft Discipline
- **No Emojis as UI Elements:** Never use emojis as interface icons or decorative elements. Use clean, minimalist inline SVG icons, subtle geometric glyphs, or semantic typographic badges.
- **No Corny Status Ornaments:** Never add pulsing or glowing green circles/dots attempting to simulate "live" or "online" status. Status must be conveyed through clean typography, authentic system telemetry, and real data.

### 1.8 Chat & Documentation Formatting
- **No Mermaid in Chat:** Chat interfaces frequently fail to render Mermaid and display broken raw text. Never output Mermaid in conversational responses.
- **Mermaid IS Allowed in `.md` Docs:** Mermaid diagrams are fully supported and encouraged inside `.md` documentation files (e.g. architecture guides, schema design).
- **Clickable File Links:** Always reference files using clickable GitHub-style links with forward slashes: `[filename](file:///absolute/path/to/file)`.

---

## 2. Git Branching, Repository Base & Collaboration Flow

To support seamless multi-contributor collaboration and automated PR workflows, adhere strictly to this branching strategy:

### 2.1 Trunk Architecture
- **`dev`** is the primary base trunk for all active development. All work branches branch off `dev` and pull into `dev`.
- **`main`** is strictly reserved for production releases.
- **Agent Restriction:** **`main` is strictly read-only for agents.** Agents must never branch off, write to, or push to `main`. Agents only inspect/read `dev` and active work branches.
- **Rule:** **NEVER commit directly to `dev` or `main`.**

### 2.2 Unified Branch Naming Convention
Instead of maintaining fragmented branch prefixes, all work uses one simple, unified pattern:

$$\text{Branch Syntax: } \mathbf{feature\text{-}\{work\text{-}description\}}$$

- **Examples:** `feature-calendar-sync`, `feature-task-filters`, `feature-auth-fix`, `feature-api-docs`
- Every branch represents a self-contained unit of work that will be opened as a Pull Request directly into `dev`.

### 2.3 Contributor & Automated PR Workflow
When starting any unit of work:

1. **Update base trunk:**
   ```bash
   git checkout dev
   git pull origin dev
   ```
2. **Create unified feature branch:**
   ```bash
   git checkout -b feature-{work-description} dev
   ```
3. **Execute work in atomic commits:**
   - Commit complete, working steps with conventional messages.
4. **Push feature branch:**
   ```bash
   git push -u origin feature-{work-description}
   ```
5. **Open Pull Request into `dev`:**
   - Base branch: `dev` $\leftarrow$ Compare branch: `feature-{work-description}`
   - Title: Follow conventional commit style (`feat: add task filtering by status`).
   - PR Body links to today's task document under `tasks/{DD_MM_YYYY}/tasks.md`.
   - Automated CI gates (typecheck, lint, build) must pass before merge into `dev`.

---

## 3. The `tasks/` Folder Pattern (Date-Driven Workstream Management)

Task planning and progress tracking across all repositories follow a simple, date-stamped folder hierarchy:

```text
{repository_root}/
└── tasks/
    ├── 18_10_2026/
    │   └── tasks.md
    └── 31_10_2026/
        └── tasks.md
```

### 3.1 Anatomy of a `tasks.md` Document
Every daily task document follows this clean, high-level structure:

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
- This maintains an auditable, high-level source of truth that survives chat context resets and allows teammates to pick up work seamlessly.

---

## 4. Stack-Agnostic Quality Gates (Pre-Flight Checklist)

Before proposing any commit or opening a PR, the agent or developer must verify that the codebase satisfies these quality gates:

### 4.1 Static Typechecks Must Pass (Zero Errors)
- Run project static type checks without emitting build files:
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
# 1. Update base trunk & clean workspace
git checkout dev
git pull origin dev

# 2. Branch out into unified feature branch
git checkout -b feature-{work-description} dev

# 3. Check status
git status

# 4. Commit atomic unit (conventional commits only)
git commit -m "feat(scope): detailed description"

# 5. Push feature branch
git push -u origin feature-{work-description}
```

### Verification & Testing
```bash
# Static typecheck
npx tsc --noEmit

# Lint
npm run lint

# Build test
npm run build
```

---
name: af-project
description: Apply af-project conventions when creating or evolving Alfin's personal software projects. Trigger when asked to follow this project guideline, scaffold a new repository, implement Rust/Actix APIs or SvelteKit frontends, review architecture, conduct code reviews with HTML reports, or enforce full-stack project standards.
license: Viewing-Only
metadata:
  author: Alfin Syahruddin
  version: "1.0.0"
---

# af-project

Reusable architecture, workflow, and code guidelines for Alfin's personal projects across backend, web, and native desktop platforms.

> [!NOTE]
> - **Backend & Web**: Rust/Actix API and SvelteKit CSR Single-Page Application represent the reference stack when building web services.
> - **Native Desktop**: Rust with GPUI (via GPUI Kit) in an acyclic 4-crate workspace (`core`, `app`, `storage`, `desktop`) represents the reference standard for native desktop applications.
> - **Other platforms** (e.g. mobile, CLI tools, ML pipelines): Follow the shared rules (Setup, Documentation, Workflow) and the target platform's idiomatic design patterns.

---

## Inviolable Golden Rules

When creating or modifying code under this guideline, agents and contributors must strictly enforce these invariants:

1. **Strict Layer Segregation**:
   - *Backend*: Routes only extract HTTP inputs and invoke services. Repositories only execute parameterized SQL. Services orchestrate business rules and external I/O. All domain DTOs strictly reside in `entities/`.
   - *Desktop*: Strict acyclic 4-crate workspace boundary (`desktop` -> `app` -> `core`, with `storage` implementing the `app` repository port). `core` never imports UI, storage, clocks, random generators, or filesystem APIs. `app` never imports GPUI or concrete OS storage. `storage` never calls back into views.
2. **Unified API Envelope**: Every HTTP endpoint returns `AppResponse<T>` (`Result<Json<BaseResponse<T>>, AppError>`) via `.json()` or `.json_data()`. Never return ad-hoc, raw JSON structures.
3. **Zero Unwraps in Production**: Never use `.unwrap()` or `.expect()` in production Rust paths; propagate errors via `Result<T, AppError>` / `DomainError` / `StorageError` and `?`.
4. **Single State Owner & UI Thread Responsiveness (Desktop)**: The application session controller is owned on the main UI executor. All mutations are ordered sequentially on the UI thread. Never perform filesystem or network I/O on the UI thread—disk persistence runs on a dedicated background worker with debouncing. Views read cached projections behind `Arc` slices.
5. **No Floating-Point Financial Math**: Checked integer arithmetic only for prices, lots, money, and IDs. Floating-point numbers are strictly forbidden in domain accounting and are only permitted for layout coordinates and percentage display.
6. **Pinned Toolchain & Explicit Dependencies**: Never modify pinned toolchains (`rust-toolchain.toml`) or dependencies (`Cargo.lock`, `package.json`) without authorization. Use `--locked` for all cargo builds. Desktop UI crates strictly pin `gpui-kit = "=0.8.0"`.
7. **Svelte 5 Runes Only (Web)**: All frontend components must use modern Svelte 5 Runes (`$props`, `$state`, `$derived`, `$effect`). Never generate legacy Svelte 3/4 syntax (`export let`, `$:`, `on:click`).
8. **Pure CSR Mode (Web)**: The web frontend is strictly a single-page application (`export const ssr = false;` in root `+layout.ts`). Never create server routes (`+page.server.ts` or `+server.ts`).
9. **Zero-Egress E2E Tests (Web)**: Playwright browser tests run with only the frontend dev server. Every API request must be intercepted with mock fixtures; tests must fail fast on unmocked requests.
10. **Natural Language Commits**: When a commit is authorized, write a descriptive message. Do NOT use Conventional Commit prefixes (`feat:`, `fix:`). Multiline commit messages with dashed bullet points are explicitly intended for multi-part changes. Message style does not authorize creating a commit.
11. **Append-Only Migrations**: Never modify or reorder migrations that have already been applied. Always create a new sequential file under `backend/migrations/` (e.g. `YYYYMMDDHHMM_description.sql`).
12. **No `@layer components` for Feature Styling**: Never use Tailwind `@layer components` or global `@apply` abstractions for feature- or page-specific styling. Colocate styles directly in Svelte components using utility classes or scoped `<style>` blocks.

---

## Agent Decision Matrix: What to Read

Find your current task and read only the relevant reference files:

| If your task is...                                            | Read this reference first                                                                                         |
| :------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------- |
| **Starting a new project from scratch**                       | [Setup](references/setup-project.md)                                                                             |
| **Deciding where to place new files or folders**              | [Project Structure](references/project-structure.md)                                                              |
| **Selecting libraries or adding dependencies**                | [Tech Stack](references/tech-stack.md)                                                                            |
| **Writing or updating README.md, AGENTS.md, or docs/**        | [Documentation](references/documentation.md)                                                                      |
| **Running tests, linting, Docker, or handoff**                | [Workflow & Verification](references/workflow.md)                                                                 |
| **Managing environment variables, secrets, or Compose**       | [Environment and Compose](references/environment-and-docker.md)                                                   |
| **Building a new backend endpoint, migration, or service**    | [Backend Code Patterns](references/backend/code-patterns.md) → [Vertical Slice](references/backend/feature.md)     |
| **Modifying core backend startup, config, or error handling** | [Backend Foundations](references/backend/foundations.md)                                                          |
| **Implementing auth, passwords, or Redis session tracking**   | [Backend Optional Components](references/backend/optional-components.md)                                          |
| **Building a new frontend page, component, or state module**  | [Frontend Code Patterns](references/frontend/code-patterns.md) → [Vertical Slice](references/frontend/feature.md)  |
| **Modifying frontend setup, CSS tokens, or API client**       | [Frontend Foundations](references/frontend/foundations.md)                                                         |
| **Implementing auth state, toasts, or theme toggling**        | [Frontend Optional Components](references/frontend/optional-components.md)                                         |
| **Building or modifying a desktop view, model, or controller**| [Desktop Code Patterns](references/desktop/code-patterns.md) → [Vertical Slice](references/desktop/feature.md)     |
| **Designing desktop UI, layouts, themes, or shortcuts**       | [Desktop UI Guide](references/desktop/ui.md)                                                                      |
| **Setting up desktop workspace, crates, or persistence**      | [Desktop Foundations](references/desktop/foundations.md)                                                          |
| **Packaging desktop app (macOS DMG, Windows EXE, Linux DEB)** | [Desktop Packaging](references/desktop/packaging.md)                                                              |
| **Conducting a code review or generating an HTML review report** | [Code Review](references/code-review.md)                                                                         |

---

## AI Workflow & Code Review Rules

These rules govern how you operate when assisting in an AF-Project:

- **Ask the user about any unclear decision** before proceeding. Do not assume ambiguous requirements, architecture choices, or scope. A clarifying question is always cheaper than a wrong implementation.
- **Always ask report language for code reviews**: When conducting a code review, ALWAYS ask whether the user wants English or Indonesia for the HTML report before generating it ([references/code-review.md](references/code-review.md)).
- **Always clarify review scope if ambiguous**: When asked to review code without an explicit diff target, immediately prompt whether to review the latest local commit (`HEAD^!`), a specific commit SHA, a branch comparison (e.g. `main...feature`), or uncommitted working changes.
- **Strict diff-only boundary**: Only review modified and added lines in the diff. Never nitpick untouched legacy code.
- **Generate self-contained HTML artifact**: Save reports to `CODE-REVIEW/code-review-<slug>.html` and ensure `CODE-REVIEW/` is in `.gitignore`.

---

## Standard Agent Execution Procedure

1. **Inspect Context**: Check repository manifests (`Cargo.toml`, `package.json`), existing tests, and contributor documentation.
2. **Follow Boundaries**: Keep domain DTOs in `entities/`, queries in `repositories/`, logic in `services/`, and routes thin.
3. **Execute Verification**: Select and report checks appropriate to the changed area, following [Workflow & Verification](references/workflow.md). Distinguish failures caused by the change from unrelated or baseline failures.
4. **Handoff Clearly**: Summarize implemented changes, tests run, and operational limits.

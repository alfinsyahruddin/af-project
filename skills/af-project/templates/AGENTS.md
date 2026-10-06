# Agent & Contributor Guide

Use `af-project` skill.

Welcome to <Project Name>. This document establishes the core architectural principles, golden rules, and verification procedures.

---

## 1. Documentation Index

| Guide | Content |
| :--- | :--- |
| `docs/backend.md` | Layer segregation, error mapping, SQLx patterns |
| `docs/frontend.md` | Svelte 5 runes, state helpers, routing guards |
| `docs/environment.md` | Host vs container ports, default local credentials |
| `docs/testing.md` | Unit tests, HTTP contract tests, Playwright E2E |
| `docs/workflow.md` | Contributor workflows, verification checklists, git guidelines |

---

## 2. Inviolable Golden Rules

1. **Strict Layering**: Routes only extract HTTP inputs and invoke services. Repositories only execute SQL. All DTOs reside in `backend/src/entities/`.
2. **Unified API Envelope**: Every endpoint returns `AppResponse<T>` via `.json()` or `.json_data()`. Never return raw ad-hoc JSON.
3. **No Unwraps in Production**: Use `Result<T, AppError>` and `?` error propagation exclusively.
4. **Svelte 5 Runes Only**: Strictly use `$props`, `$state`, `$derived`, and `$effect`. No legacy Svelte 3/4 syntax.
5. **CSR Only**: Set `export const ssr = false;` in root `+layout.ts`. Never create server routes.
6. **Zero-Egress E2E**: Playwright tests must mock all API endpoints and run against the frontend dev server only.
7. **Commit Message Style**: When a commit is authorized, use concise natural language. Do NOT use Conventional Commit prefixes (`feat:`, `fix:`). Multiline commit messages with dashed list details are explicitly intended for multi-part changes. This style guidance does not authorize creating a commit.
8. **Append-Only Migrations**: Never modify or reorder migrations that have already run. Always create a new sequential file under `backend/migrations/`.
9. **No `@layer components` for Views**: Never use Tailwind `@layer components` or global `@apply` abstractions for feature- or page-specific styling. Colocate styles directly in Svelte components with utility classes or scoped `<style>` blocks.

---

## 3. Verification Checklist

Choose checks based on the files and behavior changed. Run focused checks for the affected area; run the full suite when the change spans areas or focused checks do not give enough coverage. Report the commands run and their results, and distinguish failures caused by the change from unrelated or baseline failures. An uncommitted working tree is a valid handoff when the changes are reviewable and unrelated user changes are preserved.
Commit message style does not authorize creating a commit; commit only when the user has authorized it.

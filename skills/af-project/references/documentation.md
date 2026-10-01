# Documentation Standards: README, AGENTS.md, & Docs

Clear, accurate, and actionable documentation is mandatory. `README.md` introduces the project to human developers; `AGENTS.md` enforces architecture invariants for AI coding agents and contributors; `docs/` houses modular deep dives.

---

## 1. `README.md` Template Skeleton

Every project must maintain a root `README.md` conforming to this layout:

```markdown
# <Project Name>

<One clear sentence explaining what this project does and the problem it solves.>

---

## Features
- **Feature A**: Summary of capability.
- **Feature B**: Summary of capability.

## Tech Stack
- **Backend**: Rust (Actix-web 4, SQLx, PostgreSQL, Redis)
- **Frontend**: SvelteKit 3, Svelte 5, Bun, Tailwind CSS v4
- **Infrastructure**: Docker Compose, Multi-stage Alpine images

## Quick Start

### 1. Prerequisites
- Docker & Docker Compose
- Bun (latest)
- Rust (stable toolchain)

### 2. Environment Setup
\`\`\`sh
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
\`\`\`

### 3. Start Backing Services
\`\`\`sh
docker compose up postgres redis -d
\`\`\`

### 4. Run Development Servers
\`\`\`sh
# Backend (from backend/)
cargo run

# Frontend (from frontend/)
bun run dev
\`\`\`

## Verification & Testing
\`\`\`sh
# Backend checks
cargo fmt --check && cargo test && cargo clippy --all-targets --all-features --locked -- -D warnings

# Frontend checks
bun run check && bun run lint && bun run test:unit && bun run format:check && bun run test:e2e
\`\`\`

## License

Copyright © 2026 Alfin Syahruddin. All rights reserved.

This project is source-available for viewing and evaluation purposes only. Unauthorized copying, modification, forking, redistribution, or hosting is strictly prohibited. See [`LICENSE`](./LICENSE) for full terms.
```

---

## 2. `AGENTS.md` Template Skeleton

`AGENTS.md` is the primary instruction file for AI agents working in the repository. Keep it compact, prescriptive, and focused on non-negotiable rules:

```markdown
# Agent & Contributor Guide

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

1. **Strict Layering**: Routes only extract HTTP inputs and invoke services. Repositories only execute SQL. All DTOs reside in \`backend/src/entities/\`.
2. **Unified API Envelope**: Every endpoint returns \`AppResponse<T>\` via \`.json()\` or \`.json_data()\`. Never return raw ad-hoc JSON.
3. **No Unwraps in Production**: Use \`Result<T, AppError>\` and \`?\` error propagation exclusively.
4. **Svelte 5 Runes Only**: Strictly use \`$props\`, \`$state\`, \`$derived\`, and \`$effect\`. No legacy Svelte 3/4 syntax.
5. **CSR Only**: Set \`export const ssr = false;\` in root \`+layout.ts\`. Never create server routes.
6. **Zero-Egress E2E**: Playwright tests must mock all API endpoints and run against the frontend dev server only.
7. **Commit Message Style**: Use concise natural language. Do NOT use Conventional Commit prefixes (`feat:`, `fix:`). Multiline commit messages with dashed list details are explicitly intended for multi-part changes.
8. **Append-Only Migrations**: Never modify or reorder migrations that have already run. Always create a new sequential file under `backend/migrations/`.
9. **No `@layer components` for Views**: Never use Tailwind `@layer components` or global `@apply` abstractions for feature- or page-specific styling. Colocate styles directly in Svelte components with utility classes or scoped `<style>` blocks.

---

## 3. Verification Checklist

Before completing any task, run:
\`\`\`sh
# Backend
cd backend && cargo fmt --check && cargo test && cargo clippy --all-targets --all-features --locked -- -D warnings

# Frontend
cd frontend && bun run check && bun run lint && bun run test:unit && bun run format:check
\`\`\`
```

---

## 3. Modular `docs/` Directory Guidelines

> [!TIP]
> When a project grows, avoid bloating `AGENTS.md`. Split deep domain rules into focused markdown files in `docs/`:

- `docs/backend.md`: Deep dive on entities, transactions, external client retry backoff, and SQL queries.
- `docs/frontend.md`: Component catalog, theme tokens, client session helpers, and modal patterns.
- `docs/environment.md`: Full reference table of all environment variables, public vs secret attributes, and default ports.
- `docs/testing.md`: Testing guidelines, mock fixture setups, and contract validation.
- `docs/workflow.md`: Contributor git workflows, migration verification steps, and release tagging.

Always update documentation in the same commit whenever an invariant, endpoint, or environment variable changes.

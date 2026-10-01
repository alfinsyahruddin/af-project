# Workflow & Verification Standards

Follow these verification procedures, commands, and contributor workflows to ensure code quality, test reliability, and smooth task handoff.

---

## 1. Automated Verification Commands

Run the full verification suite before committing or completing any task:

### Backend Quality Gates
Execute from `backend/`:

```sh
# 1. Format check
cargo fmt --check

# 2. Unit and HTTP contract tests
cargo test

# 3. Strict Clippy linting (warnings treated as errors)
cargo clippy --all-targets --all-features --locked -- -D warnings
```

### Frontend Quality Gates
Execute from `frontend/`:

```sh
# 1. Svelte & TypeScript type check
bun run check

# 2. ESLint code quality check
bun run lint

# 3. Unit and component tests (always run via bun run test:unit, never plain bun test)
bun run test:unit

# 4. Code formatting check
bun run format:check

# 5. Playwright E2E browser tests (with dev server only)
bun run test:e2e
```

---

## 2. Infrastructure & Docker Operations

Run these commands from the repository root:

```sh
# Start backing services (PostgreSQL & Redis) in background for host development
docker compose up postgres redis -d

# Start the full containerized stack (Postgres, Redis, Backend, Frontend)
docker compose up -d --build

# View real-time container logs
docker compose logs -f

# Stop and tear down all project containers and networks
docker compose down
```

---

## 3. Database Migration Verification

When modifying database schemas:
1. **Append-Only Rule**: Never edit or reorder an existing migration that has already been applied. Always create a new sequential file under `backend/migrations/` (e.g., `YYYYMMDDHHMM_description.sql`).
2. **Clean-Slate Verification**: Verify migrations apply cleanly to a completely fresh database instance without errors:
   ```sh
   sqlx migrate run
   ```
3. **Compile-Time Embedding**: Remember that `sqlx::migrate!()` runs at compile-time in Rust. Ensure new SQL files are committed and present during `cargo check`, `cargo build`, and container image creation.

---

## 4. Pre-Handoff Quality Gate Checklist

Before reporting a task complete or submitting a pull request, verify each gate:

| Gate | Check | Expected Outcome |
| :--- | :--- | :--- |
| **Backend Formatting** | `cargo fmt --check` | Clean exit (0) |
| **Backend Tests** | `cargo test` | All unit and contract tests pass |
| **Backend Lints** | `cargo clippy ... -D warnings` | Zero warnings and zero errors |
| **Frontend Types** | `bun run check` | 0 errors and 0 warnings |
| **Frontend Lints** | `bun run lint` | Clean exit (0) |
| **Frontend Unit Tests** | `bun run test:unit` | All Vitest suites pass |
| **Frontend E2E Tests** | `bun run test:e2e` | All mocked journeys pass; no unmocked egress |
| **Git Working Tree** | `git status` | Clean; no leftover scratch or temp files |

---

## 5. Contributor Handoff Standards

> [!NOTE]
> **Commit Message Guidelines**:
> - Use clear, descriptive natural language summaries in title or sentence case (e.g., `Add user profile settings endpoint and validation`).
> - **Do NOT use Conventional Commit prefixes** (never use `feat:`, `fix:`, `chore:`).
> - **Multiline list style is intended**: When a commit spans multiple distinct parts or changes, a multiline format (a concise summary title followed by a blank line and dashed bullet details) is explicitly intended and recommended.

#### Commit Message Format Examples:
**Single-line format:**
```text
Add user profile settings endpoint and validation
```

**Multiline list format (intended for multi-part changes):**
```text
Add user profile settings endpoint and validation

- Implement settings route, service, and DTO validation in backend
- Add settings repository query with parameterized SQL
- Add Svelte 5 settings component in frontend and wire with api.ts
- Add Playwright E2E journey test covering settings update
```

When finishing an implementation task:
1. **Summary of Changes**: Detail what was implemented, updated, or refactored.
2. **Verification Statement**: List exactly which commands ran and succeeded.
3. **Operational Limits**: Note any known edge cases, pending credentials, or remaining out-of-scope work.
4. **Preserve User Changes**: Never discard unrelated files or modifications present in the working tree.

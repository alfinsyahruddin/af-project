# Project Structure & File Placement

Maintain a clean, predictable repository directory structure where code, tests, configuration, and documentation reside in dedicated, discoverable locations.

---

## 1. Full-Stack Repository Tree

The standard repository layout for a Rust/Actix API and SvelteKit web application:

```text
<project>/
├── backend/                      # Rust API service (Tokio, Actix-web, SQLx, Redis)
│   ├── migrations/               # Sequential, append-only SQLx migration scripts
│   ├── src/
│   │   ├── clients/              # External API adapters and third-party HTTP clients
│   │   ├── constants/            # Global constants (e.g., Redis key prefixes, limits)
│   │   ├── entities/             # Domain structs, DTOs, errors, response envelopes, config
│   │   ├── enums/                # System-wide domain enums and database-mapped types
│   │   ├── guards/               # Actix extractors (authentication, role-based authorization)
│   │   ├── helpers/              # Pure utility functions (hashing, math, parsing)
│   │   ├── repositories/         # Direct PostgreSQL query execution (SQLx)
│   │   ├── routes/               # Actix HTTP route handlers and parameter extraction
│   │   ├── services/             # Business logic orchestration, validation, domain rules
│   │   ├── setup/                # Infrastructure initialization (database pools, Redis, HTTP)
│   │   ├── di.rs                 # Dependency injection container and app data wiring
│   │   ├── http.rs               # Middleware, CORS configuration, centralized error handling
│   │   ├── lib.rs                # Library entrypoint exporting modules for tests
│   │   └── main.rs               # Server runtime bootstrap entrypoint
│   ├── tests/                    # Integration and HTTP contract test suites
│   ├── Cargo.toml                # Rust package manifest
│   ├── Dockerfile                # Multi-stage production container build (Alpine)
│   ├── .dockerignore             # Docker build context exclusion list
│   ├── .env.example              # Host development environment template
│   └── .env.docker.example       # Docker container environment template
│
├── frontend/                     # SvelteKit 3 single-page web client (Svelte 5, Bun, Tailwind v4)
│   ├── src/
│   │   ├── app.css               # Tailwind CSS v4 design tokens and CSS variables
│   │   ├── app.html              # Shell HTML template
│   │   ├── lib/
│   │   │   ├── api.ts            # Typed API client with unified response handling
│   │   │   ├── constants.ts      # Shared UI constants and configuration keys
│   │   │   ├── types.ts          # Frontend domain interfaces and type definitions
│   │   │   ├── components/       # Svelte 5 UI components (shared, layout, and feature-specific)
│   │   │   └── helpers/          # Client utilities, reactive state modules (.svelte.ts)
│   │   └── routes/               # CSR client routes (+layout.svelte, +layout.ts, +page.svelte)
│   ├── tests/
│   │   ├── unit/                 # Vitest unit and component tests
│   │   └── e2e/                  # Playwright browser end-to-end user journey tests
│   ├── package.json              # Bun dependencies and scripts
│   ├── vitest.config.ts          # Kit 3 component tests with JSDOM
│   ├── playwright.config.ts      # Playwright test configuration
│   ├── tsconfig.json             # TypeScript configuration extending $app/tsconfig
│   ├── vite.config.ts            # Kit 3 adapter, preprocessing, Tailwind, and environment config
│   ├── nginx.conf                # Nginx Alpine SPA routing fallback and cache rules
│   ├── Dockerfile                # Multi-stage container build (Bun -> Nginx Alpine SPA)
│   ├── .dockerignore             # Docker build context exclusion list
│   ├── .env.example              # Host development environment template
│   └── .env.docker.example       # Container build environment template
│
├── docs/                         # Detailed architecture, environment, and contributor guides
├── docker-compose.yml            # Local development backing services and full-stack orchestration
├── AGENTS.md                     # Contributor entrypoint and critical architecture invariants
├── LICENSE                       # Terms selected for this project's distribution
└── README.md                     # Product overview, prerequisites, and quick-start instructions
```

---

### 1.2 Desktop Application Workspace Tree

The standard repository layout for a native Rust + GPUI desktop application using the 4-crate architecture:

```text
<desktop-project>/
├── Cargo.toml                 # Workspace manifest & pinned dependency declarations
├── Cargo.lock                 # Locked dependency graph
├── rust-toolchain.toml        # Pinned compiler toolchain (Rust 1.97.1)
├── AGENTS.md                  # Contributor entrypoint and critical invariants
├── README.md                  # Project overview, prerequisites, and build steps
│
├── crates/
│   ├── <app>-core/            # Pure domain engine & deterministic state machines
│   │   ├── src/
│   │   │   ├── entities/      # Pure domain models (enums, immutable structs)
│   │   │   ├── rules/         # Invariant rules, checked math, validation
│   │   │   ├── engine/        # Core state machines and deterministic transition logic
│   │   │   └── lib.rs         # Root library exporting domain rules and types
│   │   └── tests/             # Invariant contract tests & domain boundary scenarios
│   │
│   ├── <app>-app/             # Application controller & state orchestration
│   │   ├── src/
│   │   │   ├── controller/    # SessionController, command dispatch, state ownership
│   │   │   ├── projection/    # Read-only projection derivations (Arc<Projection>)
│   │   │   ├── entities/      # Application-layer state, snapshots, leaderboard/accounts
│   │   │   ├── types/         # Commands, outcomes, AppError, SessionRepository port
│   │   │   └── lib.rs         # Module exports
│   │   └── tests/             # Command workflow & state progression tests (headless)
│   │
│   ├── <app>-storage/         # Persistence implementation & save coordination
│   │   ├── src/
│   │   │   ├── repository/    # FileSessionRepository implementing app's port
│   │   │   ├── coordinator/   # Background save coordinator (channels, 250ms debouncing)
│   │   │   ├── lock/          # Profile file locking (.lock) preventing multi-instance write
│   │   │   ├── schema/        # Versioned snapshot serialization & recovery backup
│   │   │   └── lib.rs         # Storage adapter exports
│   │   └── tests/             # Round-trip persistence, corruption, and recovery tests
│   │
│   └── <app>-desktop/         # GPUI presentation & native OS entrypoint
│       ├── src/
│       │   ├── views/         # GPUI views (workspace, panels, status bars, order tables)
│       │   ├── modals/        # Modal dialogs (settings, confirmations, inspections)
│       │   ├── components/    # Reusable GPUI elements (segmented controls, charts, badges)
│       │   ├── actions/       # GPUI actions and scoped keyboard shortcuts
│       │   ├── theme/         # Color palettes, font definitions, and dark/light modes
│       │   ├── assets/        # Asset loaders (fonts, icons, SVG vectors)
│       │   ├── main.rs        # Runtime bootstrap, window creation, lock acquisition
│       │   └── lib.rs         # View exports for in-process interaction testing
│       ├── build.rs           # Windows resource compilation (app.rc, embedded icons)
│       └── tests/             # In-process GPUI interaction & modal lifecycle tests
│
├── assets/                    # Static runtime assets (fonts, icons, licenses)
├── packaging/                 # Native OS packaging scripts and metadata
│   ├── macos/                 # Info.plist, entitlements, archive.sh, package.sh (DMG)
│   ├── windows/               # installer.iss (Inno Setup), app.rc, package.ps1
│   └── linux/                 # app.desktop, Debian control file, package.sh (.deb)
└── docs/                      # Architectural specs, UI guide, verification procedures
```

---

## 2. File Placement Quick Lookup

When adding new files or features, use this lookup table to determine the exact location:

| If you are creating... | Place it in... | Notes / Constraints |
| :--- | :--- | :--- |
| **Request / Response DTO** | `backend/src/entities/` | Never define DTOs inside route handlers or repositories. |
| **Database Row Model** | `backend/src/entities/` | Derived with `sqlx::FromRow` where applicable. |
| **Domain Error Variants** | `backend/src/entities/app_error.rs` | Managed in centralized `AppError` enum. |
| **Raw SQL Queries** | `backend/src/repositories/` | Parameterized SQL queries only (`$1, $2`). |
| **Business Logic / Workflow** | `backend/src/services/` | Coordinates repositories, validation, and caching. |
| **HTTP Route Handler** | `backend/src/routes/` | Thin extractors returning `AppResponse<T>`. |
| **Auth / Role Check Extractor** | `backend/src/guards/` | Implements Actix `FromRequest`. |
| **Third-Party API Adapter** | `backend/src/clients/` | Define an abstract trait for unit test mockability. |
| **Database Migration Script** | `backend/migrations/` | Sequential prefix (`YYYYMMDDHHMM_name.sql`). Append-only. |
| **HTTP Contract Test** | `backend/tests/` | Black-box HTTP tests using `actix_web::test`. |
| **Reusable UI Component** | `frontend/src/lib/components/` | Svelte 5 runes only (`$props`, `$state`, `$derived`). |
| **Client Reactive Store** | `frontend/src/lib/helpers/` | Encapsulate in `.svelte.ts` modules. |
| **Client HTTP Transport** | `frontend/src/lib/api.ts` | Centralized typed wrapper around `fetch()` with auto-refresh deduplication. |
| **Client Navigation Route** | `frontend/src/routes/` | Page layout and view rendering. |
| **Browser E2E Test** | `frontend/tests/e2e/` | Playwright journey test with 100% mocked API calls. |
| **SPA Fallback Server Conf** | `frontend/nginx.conf` | Alpine Nginx config routing client paths to `index.html`. |
| **Static Adapter Config** | `frontend/vite.config.ts` | Passes `@sveltejs/adapter-static` with fallback `index.html` to `sveltekit()`. |
| **Tailwind & Vite Setup** | `frontend/vite.config.ts` | Configures `@tailwindcss/vite` and `envPrefix: ['VITE_', 'PUBLIC_']`. |
| **Desktop Domain Rule / Math** | `crates/<app>-core/src/rules/` | Pure function; zero I/O or external dependencies. |
| **Desktop Core State Machine** | `crates/<app>-core/src/engine/` | Deterministic state transitions and invariant checks. |
| **Desktop Contract Test** | `crates/<app>-core/tests/` | Invariant checks, stress scenarios, boundary conditions. |
| **Desktop State Controller** | `crates/<app>-app/src/controller/` | Owns session state; executes on UI executor only. |
| **Desktop State Projection** | `crates/<app>-app/src/projection/` | Read-only DTO shared behind `Arc<Projection>`. |
| **Desktop Command Workflow Test** | `crates/<app>-app/tests/` | Multi-step command execution without window creation. |
| **Desktop Storage Repository** | `crates/<app>-storage/src/` | Implements `SessionRepository` port via versioned JSON. |
| **Desktop Atomic Save Worker** | `crates/<app>-storage/src/` | Background thread with 250ms debouncing and `.recovery` backup. |
| **Desktop GPUI View / Panel** | `crates/<app>-desktop/src/views/` | Implements `gpui::Render`; reads immutable projections. |
| **Desktop Modal Dialog** | `crates/<app>-desktop/src/modals/` | Self-contained GPUI view with scoped keyboard dismissal. |
| **Desktop Keyboard Shortcut** | `crates/<app>-desktop/src/actions/` | Declarative action bound via `cx.bind_keys`. |
| **Desktop Packaging Script** | `packaging/<platform>/` | macOS `archive.sh`, Windows `installer.iss`, Linux `package.sh`. |

---

## 3. Placement Guardrails

> [!WARNING]
> - **No Server Routes in Frontend**: The frontend is strictly a Single-Page Application (`ssr = false`). Never create `+page.server.ts` or `+server.ts` files in `frontend/src/routes/`.
> - **No SQL in Services**: Services must never execute raw SQL or know about database drivers. Database calls must always go through `repositories/`.
> - **No Business Rules in Routes**: Handlers must only deserialize parameters, call a service method, and map the result to `AppResponse<T>`.
> - **No UI or Storage in Core**: `<app>-core` must never import GPUI, storage, clocks, random generators, or filesystem APIs.
> - **No GPUI in Application Controller**: `<app>-app` must never import GPUI or concrete OS storage. It defines repository traits and runs headless.
> - **No Disk I/O in Views**: `<app>-desktop` views must never read or write filesystem files synchronously. Persistence is delegated through the background save coordinator.

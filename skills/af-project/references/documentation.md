# Documentation Standards: README, AGENTS.md, & Docs

Clear, accurate, and actionable documentation is mandatory. `README.md` introduces the project to human developers; `AGENTS.md` enforces architecture invariants for AI coding agents and contributors; `docs/` houses modular deep dives.

Use the starter templates in [`templates/`](../templates/) as the authoritative source of truth for the root `README.md` and `AGENTS.md` skeletons.

---

## 1. Root `README.md`

Every project must maintain a root `README.md` based on the starter template in [`templates/README.md`](../templates/README.md).

### Headings & Layout Structure

The root `README.md` begins with a centered branding and preview block followed by detailed overview, features, tech stack, and setup instructions:

1. **Centered Header Block (`<div align="center">`)**:
   - **Theme-aware Logo**: `<picture>` block specifying dark mode (`./frontend/static/logo-dark.svg`) and light mode (`./frontend/static/logo-light.svg`) with `height="36"`.
   - **App Name**: App / Project Name
   - **Tagline**: Single bold sentence defining core purpose and problem solved.
   - **Badges**: Centered badges in `<p align="center">` highlighting key tech stack (Rust, Svelte 5) and primary data or third-party service providers.
2. **Overview (`## Overview`)**:
   - Bold project name followed by a comprehensive summary of functionality, target audience, and key value proposition.
   - Quick resource links: 🌐 Live Preview, 🎬 1-min Teaser, 🎥 3-min Demo.
3. **Features (`## Features`)**: Bulleted list summarizing core features.
4. **Tech Stack (`## Tech Stack`)**: Breakdown across backend, frontend, and infrastructure.
5. **Quick Start (`## Quick Start`)**: Step-by-step setup (Prerequisites, Environment Setup, Backing Services, Development Servers).
6. **Verification & Testing (`## Verification & Testing`)**: Focused checks vs full-suite execution guidance.
7. **License (`## License`)**: Project licensing terms.

### Example Header Format

```markdown
<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="./frontend/static/logo-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="./frontend/static/logo-light.svg">
  <img src="./frontend/static/logo-light.svg" alt="Trading Lab Logo" height="36" />
</picture>

---

**AI-powered backtesting platform for the Indonesia Stock Exchange (IDX).**

<p align="center">
    <a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/Rust-1.80+-orange?logo=rust" alt="Rust"/></a>
    <a href="https://svelte.dev/"><img src="https://img.shields.io/badge/Svelte-5_Runes-FF3E00?logo=svelte" alt="Svelte 5"/></a>
    <a href="https://sectors.app"><img src="https://img.shields.io/badge/Data-Sectors.app-ff0000" alt="Sectors.app"/></a>
</p>

<br />

<img src="./frontend/static/backtest.webp" alt="Trading Lab Quantitative Backtesting Platform" width="100%" />

</div>

---

## Overview

**Trading Lab** is a backtesting platform built for the Indonesia Stock Exchange (IDX). It enables traders to easily build custom multi-condition trading strategies (AI-Powered), test them against historical market data, analyze risk and performance metrics, and share winning strategies with the community.

- 🌐 **Live Preview**: [https://trading-lab.xyz](https://trading-lab.xyz)
- 🎬 **1-min Teaser**: [https://youtu.be/p1dY_RPsLf4](https://youtu.be/p1dY_RPsLf4)
- 🎥 **3-min Demo**: [https://youtu.be/pqEO3HtPNvs](https://youtu.be/pqEO3HtPNvs)

## Features
```

---

## 2. Contributor Guide (`AGENTS.md`)

`AGENTS.md` is the primary instruction file for AI agents working in the repository. Keep it compact, prescriptive, and focused on non-negotiable rules. It explicitly instructs coding agents to use the `af-project` skill (`Use \`af-project\` skill.`).

See the authoritative starter template in [`templates/AGENTS.md`](../templates/AGENTS.md).

It contains three mandatory sections:
1. **Documentation Index**: Quick lookup table mapping modular guides under `docs/` (`docs/backend.md`, `docs/frontend.md`, etc.).
2. **Inviolable Golden Rules**: Explicit architectural constraints (strict layering, unified API envelope, zero unwraps, Svelte 5 runes, CSR only, zero-egress E2E, natural language commits, append-only migrations, no `@layer components` for view styling).
3. **Verification Checklist**: Context-aware testing and reporting instructions.

---

## 3. Modular `docs/` Directory Guidelines

> [!TIP]
> When a project grows, avoid bloating `AGENTS.md`. Split deep domain rules into focused markdown files in `docs/`:

- `docs/backend.md`: Deep dive on entities, transactions, external client retry backoff, and SQL queries.
- `docs/frontend.md`: Component catalog, theme tokens, client session helpers, and modal patterns.
- `docs/environment.md`: Full reference table of all environment variables, public vs secret attributes, and default ports.
- `docs/testing.md`: Testing guidelines, mock fixture setups, and contract validation.
- `docs/workflow.md`: Contributor git workflows, migration verification steps, and release tagging.

When committing a change to an invariant, endpoint, or environment variable, update its documentation in that commit.

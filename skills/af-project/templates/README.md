<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="./frontend/static/logo-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="./frontend/static/logo-light.svg">
  <img src="./frontend/static/logo-light.svg" alt="<Project Name> Logo" height="36" />
</picture>

# <Project Name>

**One clear sentence explaining what this project does and the problem it solves.**

<p align="center">
    <a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/Rust-1.80+-orange?logo=rust" alt="Rust"/></a>
    <a href="https://svelte.dev/"><img src="https://img.shields.io/badge/Svelte-5_Runes-FF3E00?logo=svelte" alt="Svelte 5"/></a>
</p>

</div>

---

## Overview

**<Project Name>** is <detailed description of the platform, the problem it solves, and core capabilities>.

- 🌐 **Live Preview**: [https://example.com](https://example.com)
- 🎬 **1-min Teaser**: [https://youtu.be/...](https://youtu.be/...)
- 🎥 **3-min Demo**: [https://youtu.be/...](https://youtu.be/...)

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

```sh
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

### 3. Start Backing Services

```sh
docker compose up postgres redis -d
```

### 4. Run Development Servers

```sh
# Backend (from backend/)
cargo run

# Frontend (from frontend/)
bun run dev
```

## Verification & Testing

Choose checks based on the files and behavior changed. Run focused checks for the affected area; run the full suite when a change spans areas or focused checks do not give enough coverage. See the project workflow guide for available commands.

## License

This is a private project, not an open source project. The skill source is available for viewing only. Unauthorized copying, modification, forking, redistribution, or hosting is prohibited. See the [`LICENSE`](./LICENSE) for full terms.

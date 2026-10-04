<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="./resources/af-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="./resources/af-light.svg">
  <img src="./resources/af-light.svg" alt="af-project Logo" width="88" height="88" />
</picture>

# af-project

**A reusable project guideline and agent skill for Alfin's personal software projects across platforms.**

<p align="center">
    <a href="./skills/af-project/LICENSE"><img src="https://img.shields.io/badge/License-Viewing--Only-blue" alt="License"/></a>
    <a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/Rust-1.80+-orange?logo=rust" alt="Rust"/></a>
    <a href="https://svelte.dev/"><img src="https://img.shields.io/badge/Svelte-5_Runes-FF3E00?logo=svelte" alt="Svelte 5"/></a>
</p>

<br />

</div>

It provides strict architectural invariants, progressive setup workflows, and concrete reference implementations.

---

## What's Included?

- 🦀 **Backend Architecture**: Layered Rust API (Actix).
- ⚡ **Frontend Architecture**: SvelteKit 3 (Svelte 5, Bun, Tailwind CSS).
- 🔒 **Security**: Argon2id password hashing, revocable Redis multi-device session tracking, and JWT auth.
- 🧪 **Testing**: Rust HTTP contract tests, Vitest unit suites, and zero-egress Playwright E2E browser journeys.
- 🐳 **Containerized**: Production-ready multi-stage Dockerfiles and healthchecked Docker Compose services.

---

## Commands

### 1. Installation

Make sure Node.js are installed, then run:

```sh
npx skills add alfinsyahruddin/af-project
```

### 2. Update

```sh
npx skills update af-project
```
### 3. Remove

```
npx skills remove af-project
```

---

## Usage in Coding Agents

Instruct your agent to follow this guideline by adding this line to your project's `AGENTS.md` or `CLAUDE.md`:

```markdown
Use `af-project` skill.
```

---

## License

This is a private project. The skill source is available for viewing only under [`skills/af-project/LICENSE`](./skills/af-project/LICENSE).

# af-project

A reusable project guideline and agent skill for Alfin's personal software projects across platforms. It provides strict architectural invariants, progressive setup workflows, and concrete reference implementations.

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

This is a private project and is not open source. The skill source is available for viewing only under [`skills/af-project/LICENSE`](./skills/af-project/LICENSE).

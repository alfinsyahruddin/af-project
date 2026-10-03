# Environment Configuration & Container Orchestration

Maintain a strict separation between host development configuration, container orchestration settings, and production secrets.

---

## 1. Dual Environment Template Architecture

> [!CAUTION]
> Never commit `.env` or `.env.docker` files containing active secrets to version control. Only commit `.env.example` and `.env.docker.example` templates with placeholder values.

Every project maintains dedicated templates for host development and container execution:

| Committed Template | Local Target File | Target Runtime Environment |
| :--- | :--- | :--- |
| `backend/.env.example` | `backend/.env` | Backend API running directly on the host machine |
| `backend/.env.docker.example` | `backend/.env.docker` | Backend API running inside Docker Compose |
| `frontend/.env.example` | `frontend/.env` | SvelteKit Vite dev server running on the host |
| `frontend/.env.docker.example` | `frontend/.env.docker` | Frontend image build arguments and browser client values |

### Root `.gitignore` Protection
Enforce strict exclusion of environment files while explicitly preserving template examples:

```gitignore
# Exclude local environment files containing credentials
.env
.env.*
!.env.example
!.env.docker.example
```

---

## 2. Host vs. Container Configuration Matrix

| Variable | Host Development (`backend/.env`) | Container Execution (`backend/.env.docker`) | Rationale |
| :--- | :--- | :--- | :--- |
| `BIND_ADDRESS` | `127.0.0.1` | `0.0.0.0` | Containers require `0.0.0.0` to accept forwarded Docker traffic |
| `PORT` | `8000` | `8000` | Standard HTTP listening port |
| `DATABASE_URL` | `postgres://...127.0.0.1:5432/app` | `postgres://...postgres:5432/app` | Container resolves service name `postgres` via Docker internal DNS |
| `REDIS_URL` | `redis://127.0.0.1:6379` | `redis://...redis:6379` | Container resolves service name `redis` via Docker internal DNS |
| `PUBLIC_API_BASE_URL` | `http://127.0.0.1:8000` | `http://127.0.0.1:8000` | **Browser must reach host machine**, not internal Docker network |

### Frontend Public Environment Variables
> [!IMPORTANT]
> Variables prefixed with `PUBLIC_` are bundled directly into client JavaScript code. Never place secret API keys, private database passwords, or JWT secrets in `PUBLIC_` variables.

Configure Vite in `frontend/vite.config.ts` to allow the `PUBLIC_` prefix:

```ts
import adapter from '@sveltejs/adapter-static';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
import { sveltekit } from '@sveltejs/kit/vite';
import tailwindcss from '@tailwindcss/vite';
import { defineConfig } from 'vite';

export default defineConfig({
	envPrefix: ['VITE_', 'PUBLIC_'],
	plugins: [
		tailwindcss(),
		sveltekit({
			preprocess: vitePreprocess(),
			adapter: adapter({ fallback: 'index.html' })
		})
	],
	server: {
		port: 3000
	}
});
```

---

## 3. Docker Compose Orchestration (`docker-compose.yml`)

Use Compose to orchestrate stateful backing services and containerized application images (ready-to-use template at [`templates/docker-compose.yml`](../templates/docker-compose.yml)):

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: app_postgres
    restart: unless-stopped
    env_file:
      - backend/.env.docker
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    container_name: app_redis
    restart: unless-stopped
    env_file:
      - backend/.env.docker
    command: ["/bin/sh", "-c", "redis-server --appendonly yes --requirepass \"$$REDIS_PASSWORD\""]
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a \"$$REDIS_PASSWORD\" ping"]
      interval: 5s
      timeout: 5s
      retries: 10

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: app_backend
    restart: unless-stopped
    env_file:
      - backend/.env.docker
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: app_frontend
    restart: unless-stopped
    env_file:
      - frontend/.env.docker
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  postgres-data:
  redis-data:
```

*(Note: The doubled dollar sign `$$` ensures variable expansion occurs inside the container shell, not during Compose YAML parsing).*

---

## 4. Multi-Stage Dockerfile Patterns

### Backend Dockerfile (`backend/Dockerfile`)
Compile migrations directly into the binary via `sqlx::migrate!()` (template at [`templates/backend/Dockerfile`](../templates/backend/Dockerfile)):

```dockerfile
FROM rust:1.80-alpine AS builder
RUN apk add --no-cache musl-dev
WORKDIR /app

COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations

RUN cargo build --release

FROM alpine:3.20 AS runner
RUN apk add --no-cache ca-certificates
WORKDIR /app

COPY --from=builder /app/target/release/backend /app/backend
EXPOSE 8000
CMD ["/app/backend"]
```

### Frontend Dockerfile (`frontend/Dockerfile`)
Build static SPA assets with Bun and serve via an Nginx Alpine container (template at [`templates/frontend/Dockerfile`](../templates/frontend/Dockerfile)):

```dockerfile
FROM oven/bun:1-alpine AS builder
WORKDIR /app

COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

COPY . .
RUN if [ -f .env.docker ]; then cp .env.docker .env; else cp .env.docker.example .env; fi
RUN bun run build

FROM nginx:alpine AS runner
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

### Frontend Nginx Configuration (`frontend/nginx.conf`)
Configure SPA client routing and caching for static assets:

```nginx
server {
    listen 3000;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml;

    # SPA routing fallback: send all client navigation paths to index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache immutable static assets
    location ~* \.(?:css|js|woff2?|svg|png|jpg|jpeg|gif|ico|webp)$ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
        try_files $uri =404;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

### SvelteKit 3 Static Adapter (`frontend/vite.config.ts`)

Configure `@sveltejs/adapter-static` through the `sveltekit()` plugin in `vite.config.ts` to emit the SPA into `build/` with an `index.html` fallback. Use the complete Vite configuration above; SvelteKit 3 does not read `svelte.config.js`.

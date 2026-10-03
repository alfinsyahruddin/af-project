# Frontend Foundations

These core foundation modules establish the centralized API transport, Tailwind CSS v4 styling tokens, pure CSR shell, and SvelteKit 3 static adapter configuration for web clients.

> [!TIP]
> Ready-to-use boilerplate templates for these foundation primitives are located in [`templates/frontend/`](../../templates/frontend/).

---

## 1. Centralized Typed API Transport (`src/lib/api.ts`)

All HTTP interactions must flow through a centralized, typed client with `ApiError`, automatic 401 token refresh deduplication, and envelope unwrapping:

```ts
const BASE_URL = import.meta.env.PUBLIC_API_BASE_URL || 'http://127.0.0.1:8000';

export class ApiError extends Error {
	constructor(
		public override message: string,
		public status: number
	) {
		super(message);
		this.name = 'ApiError';
	}
}

interface ApiEnvelope<T> {
	data: T | null;
	status: number;
	message: string | null;
	timestamp: string;
}

let refreshPromise: Promise<string | null> | null = null;

export function handleAuthFailure(): void {
	if (typeof window !== 'undefined') {
		localStorage.removeItem('auth_token');
		localStorage.removeItem('refresh_token');
		if (!window.location.pathname.startsWith('/login')) {
			window.location.href = '/login';
		}
	}
}

export async function attemptTokenRefresh(): Promise<string | null> {
	const refreshToken = typeof window !== 'undefined' ? localStorage.getItem('refresh_token') : null;
	if (!refreshToken) {
		handleAuthFailure();
		return null;
	}

	if (refreshPromise) return refreshPromise;

	refreshPromise = (async () => {
		try {
			const res = await fetch(`${BASE_URL}/api/users/refresh`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ refresh_token: refreshToken })
			});

			const envelope: ApiEnvelope<{ access_token: string; refresh_token: string }> = await res.json();
			if (!res.ok || envelope.status >= 400 || !envelope.data) {
				handleAuthFailure();
				return null;
			}

			localStorage.setItem('auth_token', envelope.data.access_token);
			localStorage.setItem('refresh_token', envelope.data.refresh_token);
			return envelope.data.access_token;
		} catch {
			handleAuthFailure();
			return null;
		} finally {
			refreshPromise = null;
		}
	})();

	return refreshPromise;
}

export async function request<T>(
	path: string,
	init?: RequestInit,
	token?: string,
	retryOnAuth = true
): Promise<T> {
	const authToken = token ?? (typeof window !== 'undefined' ? localStorage.getItem('auth_token') : undefined);
	const headers: Record<string, string> = {
		'Content-Type': 'application/json',
		Accept: 'application/json',
		...(authToken ? { Authorization: `Bearer ${authToken}` } : {})
	};

	const res = await fetch(`${BASE_URL}${path}`, {
		...init,
		headers: { ...headers, ...(init?.headers ?? {}) }
	});

	let envelope: ApiEnvelope<T>;
	try {
		envelope = await res.json();
	} catch {
		if (res.status === 401 && retryOnAuth && path !== '/api/users/login' && path !== '/api/users/refresh') {
			const newToken = await attemptTokenRefresh();
			if (newToken) return request<T>(path, init, newToken, false);
		}
		throw new ApiError('An unexpected server response occurred', res.status);
	}

	if (!res.ok || envelope.status >= 400) {
		if ((res.status === 401 || envelope.status === 401) && retryOnAuth && path !== '/api/users/login' && path !== '/api/users/refresh') {
			const newToken = await attemptTokenRefresh();
			if (newToken) return request<T>(path, init, newToken, false);
		}
		throw new ApiError(envelope.message || 'An unexpected error occurred', envelope.status || res.status);
	}

	return envelope.data as T;
}
```

---

## 2. Tailwind CSS v4 Styling & Tokens (`src/app.css`)

Configure Tailwind CSS v4 with cascade layers, `@custom-variant dark`, and semantic CSS variables:

```css
@import 'tailwindcss';

@custom-variant dark (&:where(.dark, .dark *));

@theme {
	--font-family-sans: system-ui, -apple-system, sans-serif;
	--color-accent: #30b4c9;
	--color-accent-hover: #2a9fb2;
}

:root {
	--bg: #f1f2f4;
	--bg-card: #ffffff;
	--bg-card-hover: #f4f6fa;
	--bg-input: #ffffff;
	--bg-input-disabled: #f1f2f4;
	--fg: #1e293b;
	--fg-muted: #64748b;
	--border: #e2e8f0;
	--border-strong: #cbd5e1;
	--accent: #30b4c9;
	--danger: #ef4444;
}

:root.dark {
	--bg: #0f172a;
	--bg-card: #1e293b;
	--bg-card-hover: #27354f;
	--bg-input: #1e293b;
	--bg-input-disabled: #334155;
	--fg: #f8fafc;
	--fg-muted: #94a3b8;
	--border: #334155;
	--border-strong: #475569;
	--accent: #30b4c9;
	--danger: #ef4444;
}
```

> [!CAUTION]
> **No `@layer components` for Feature Styling**:
> - Never use `@layer components` or global `@apply` abstractions for feature- or page-specific styling.
> - Feature and page styling must be written directly in the Svelte component using Tailwind utility classes or scoped component `<style>` blocks.
> - `src/app.css` is strictly reserved for `@theme` tokens, `@custom-variant`, `:root` / `:root.dark` design variables, and global base HTML/body resets.

---

## 3. Pure CSR Mode & Shell Configuration

The web client operates strictly as a Single-Page Application (CSR only):

### `src/routes/+layout.ts`: Disable SSR Globally
```ts
export const ssr = false;
export const prerender = false;
```

### `src/routes/+layout.svelte`: Root Application Shell
```svelte
<script lang="ts">
	import '../app.css';
	import type { Snippet } from 'svelte';
	import ToastViewport from '#lib/components/ToastViewport.svelte';

	interface Props {
		children: Snippet;
	}

	let { children }: Props = $props();
</script>

<div class="min-h-screen bg-(--bg) text-(--fg) font-sans antialiased">
	{@render children()}
	<ToastViewport />
</div>
```

### `src/app.html`: Shell HTML Template
```html
<!doctype html>
<html lang="en">
	<head>
		<meta charset="utf-8" />
		<link rel="icon" href="%sveltekit.assets%/favicon.png" />
		<meta name="viewport" content="width=device-width, initial-scale=1" />
		%sveltekit.head%
	</head>
	<body data-sveltekit-preload-data="hover">
		<div style="display: contents">%sveltekit.body%</div>
	</body>
</html>
```

---

## 4. SvelteKit 3 Build & Adapter Configuration

### `package.json`: Library Subpath Imports

Declare the library directory with Node subpath imports:

```json
{
	"imports": {
		"#lib/*": "./src/lib/*"
	}
}
```

Import TypeScript modules using their `.js` output extension (TypeScript and Vite resolve the corresponding `.ts` source), including rune modules such as `#lib/helpers/toast.svelte.js`. Keep `.svelte` for component imports, such as `#lib/components/ToastViewport.svelte`. SvelteKit 3 no longer generates the `$lib` alias.

### `tsconfig.json`: SvelteKit TypeScript Configuration

Extend `$app/tsconfig` and explicitly include source, tests, and tool configuration files:

```json
{
	"extends": "$app/tsconfig",
	"compilerOptions": {
		"sourceMap": true,
		"strict": true
	},
	"include": ["src", "tests", "*.ts", "*.js"],
	"exclude": ["src/service-worker"]
}
```

SvelteKit 3 reads its project configuration from the `sveltekit()` Vite plugin. A separate `svelte.config.js` is no longer supported. Pass the static adapter and Svelte preprocessing through the plugin options:

### `vite.config.ts`: Static SPA Adapter, Tailwind v4 & Environment Prefix
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

### `vitest.config.ts`: Kit 3 Component Tests

Reuse the application Vite configuration so tests share its Kit 3 compiler and subpath import setup. The Svelte Testing Library plugin selects browser exports and cleans up mounted components between tests:

```ts
import { svelteTesting } from '@testing-library/svelte/vite';
import { defineConfig, mergeConfig } from 'vitest/config';
import viteConfig from './vite.config.js';

export default mergeConfig(
	viteConfig,
	defineConfig({
		plugins: [svelteTesting()],
		test: {
			environment: 'jsdom',
			include: ['tests/unit/**/*.{test,spec}.ts']
		}
	})
);
```

---

## 5. Nginx Production SPA Routing (`frontend/nginx.conf`)

Route all SPA client navigation paths to `index.html` and cache immutable static assets:

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

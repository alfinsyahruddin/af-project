# Frontend Optional Components: Auth, Themes, & Toasts

These optional components provide client-side authentication handling, reactive theme toggling, and global toast notifications. Add them only when project requirements demand stateful user sessions and enhanced interactivity.

---

## 1. Authentication & Session State (`src/lib/helpers/auth.svelte.ts`)

Manage authentication tokens, current user session state, and login/logout lifecycle:

```ts
import { request, handleAuthFailure } from '#lib/api.js';

export interface UserSession {
	id: string;
	email: string;
	role: 'ADMIN' | 'MEMBER';
}

class AuthState {
	user = $state<UserSession | null>(null);
	isAuthenticated = $derived(this.user !== null);
	isLoading = $state(true);

	init() {
		const token = typeof window !== 'undefined' ? localStorage.getItem('auth_token') : null;
		if (token) {
			this.fetchCurrentUser();
		} else {
			this.isLoading = false;
		}
	}

	async fetchCurrentUser() {
		this.isLoading = true;
		try {
			this.user = await request<UserSession>('/api/users/me');
		} catch {
			this.user = null;
			handleAuthFailure();
		} finally {
			this.isLoading = false;
		}
	}

	setSession(token: string, refreshToken: string, user: UserSession) {
		localStorage.setItem('auth_token', token);
		localStorage.setItem('refresh_token', refreshToken);
		this.user = user;
	}

	logout() {
		this.user = null;
		handleAuthFailure();
	}
}

export const authState = new AuthState();
```

### Client-Side Protected Route Guard (`src/routes/dashboard/+layout.ts`)
Prevent unauthenticated users from seeing protected views:

```ts
import { redirect } from '@sveltejs/kit';

export const load = () => {
	const token = typeof window !== 'undefined' ? localStorage.getItem('auth_token') : null;
	if (!token) {
		throw redirect(302, '/login');
	}
};
```

> [!NOTE]
> Client navigation guards improve UX by avoiding blank screen flashes; the backend must independently authenticate and authorize every API request.

---

## 2. Theme Management (`src/lib/helpers/theme.ts` & `ThemeToggle.svelte`)

### `src/lib/helpers/theme.ts`
Detect system color scheme, persist user preference in `localStorage`, and toggle the `.dark` class:

```ts
export type Theme = 'dark' | 'light';

export function getCurrentTheme(): Theme {
	if (typeof window === 'undefined') return 'light';

	const stored = localStorage.getItem('theme') as Theme | null;
	if (stored === 'dark' || stored === 'light') return stored;

	return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
}

export function applyTheme(theme: Theme): void {
	if (typeof document === 'undefined') return;

	const root = document.documentElement;
	if (theme === 'dark') {
		root.classList.add('dark');
	} else {
		root.classList.remove('dark');
	}
	localStorage.setItem('theme', theme);
}

export function toggleTheme(): Theme {
	const current = getCurrentTheme();
	const next: Theme = current === 'dark' ? 'light' : 'dark';
	applyTheme(next);
	return next;
}
```

### `src/lib/components/ThemeToggle.svelte`
Interactive theme switcher using Svelte 5 runes and `@iconify/svelte`:

```svelte
<script lang="ts">
	import Icon from '@iconify/svelte';
	import { toggleTheme, getCurrentTheme } from '#lib/helpers/theme.js';

	interface Props {
		ariaLabel?: string;
	}

	const { ariaLabel = 'Toggle theme' }: Props = $props();

	let theme = $state<'dark' | 'light'>(getCurrentTheme());
	const isDark = $derived(theme === 'dark');

	function handleToggle() {
		theme = toggleTheme();
	}
</script>

<button
	type="button"
	onclick={handleToggle}
	class="btn-interactive flex size-9 items-center justify-center rounded-lg transition-colors duration-150 hover:bg-(--bg-card-hover)"
	style="color: var(--fg-muted);"
	aria-label={ariaLabel}
	title={isDark ? 'Switch to light mode' : 'Switch to dark mode'}
>
	{#if isDark}
		<Icon icon="lucide:sun" width="18" height="18" />
	{:else}
		<Icon icon="lucide:moon" width="18" height="18" />
	{/if}
</button>
```

---

## 3. Global Toast Notifications

### `src/lib/helpers/toast.svelte.ts`
Manage notification queue with auto-dismiss timers and convenient shorthand methods:

```ts
export type ToastType = 'success' | 'error' | 'info';

export interface ToastItem {
	id: number;
	type: ToastType;
	message: string;
}

let nextId = 0;

export const toasts: ToastItem[] = $state([]);

export function showToast(type: ToastType, message: string, duration = 4000): void {
	const id = nextId++;
	toasts.push({ id, type, message });
	setTimeout(() => {
		const idx = toasts.findIndex((t) => t.id === id);
		if (idx !== -1) toasts.splice(idx, 1);
	}, duration);
}

export function dismissToast(id: number): void {
	const idx = toasts.findIndex((t) => t.id === id);
	if (idx !== -1) toasts.splice(idx, 1);
}

export const toast = {
	success: (msg: string, duration?: number) => showToast('success', msg, duration),
	error: (msg: string, duration?: number) => showToast('error', msg, duration),
	info: (msg: string, duration?: number) => showToast('info', msg, duration),
	dismiss: (id: number) => dismissToast(id)
};
```

### `src/lib/components/ToastViewport.svelte`
Smooth floating toast viewport with enter/exit transitions and Lucide icons:

```svelte
<script lang="ts">
	import Icon from '@iconify/svelte';
	import { toasts, dismissToast } from '#lib/helpers/toast.svelte.js';
	import { fly } from 'svelte/transition';
	import { backOut, backIn } from 'svelte/easing';
</script>

<div
	class="pointer-events-none fixed inset-x-4 bottom-4 z-100 flex flex-col gap-2.5 sm:left-auto sm:w-88"
>
	{#each toasts as toast (toast.id)}
		<div
			in:fly={{ x: 150, duration: 400, easing: backOut }}
			out:fly={{ x: 150, duration: 300, easing: backIn }}
			class="pointer-events-auto flex items-center gap-3 rounded-2xl border px-4 py-3.5 shadow-xl backdrop-blur-xl"
			style="
				background-color: {toast.type === 'success'
				? 'rgba(34, 197, 94, 0.14)'
				: toast.type === 'error'
					? 'rgba(239, 68, 68, 0.14)'
					: 'rgba(48, 180, 201, 0.14)'};
				border-color: {toast.type === 'success'
				? 'rgba(34, 197, 94, 0.35)'
				: toast.type === 'error'
					? 'rgba(239, 68, 68, 0.35)'
					: 'rgba(48, 180, 201, 0.35)'};
				box-shadow: 0 12px 30px -4px rgba(0, 0, 0, 0.15), 0 4px 12px -2px rgba(0, 0, 0, 0.08);
			"
		>
			<!-- Icon -->
			<div
				class="shrink-0"
				style="color: {toast.type === 'success'
					? 'var(--success, #22c55e)'
					: toast.type === 'error'
						? 'var(--danger, #ef4444)'
						: 'var(--accent, #30b4c9)'}"
			>
				{#if toast.type === 'success'}
					<Icon icon="lucide:check-circle" width="18" height="18" />
				{:else if toast.type === 'error'}
					<Icon icon="lucide:alert-circle" width="18" height="18" />
				{:else}
					<Icon icon="lucide:info" width="18" height="18" />
				{/if}
			</div>

			<!-- Message -->
			<p class="font-600 flex-1 text-sm leading-snug" style="color: var(--fg)">
				{toast.message}
			</p>

			<!-- Dismiss button -->
			<button
				onclick={() => dismissToast(toast.id)}
				class="btn-interactive -mr-1 rounded-lg p-1 transition-colors duration-150 hover:bg-black/5 dark:hover:bg-white/10 cursor-pointer"
				style="color: var(--fg-muted);"
				aria-label="Dismiss notification"
			>
				<Icon icon="lucide:x" width="14" height="14" />
			</button>
		</div>
	{/each}
</div>
```

---

## 4. Complete `package.json` Reference

SvelteKit 3 requires Node.js 22.17 or newer when running on Node, TypeScript 6, Svelte 5.57.1 or newer, Vite 8.0.12 or newer, and `@sveltejs/vite-plugin-svelte` 7 or newer. The versions below meet those minimums and match the frontend template.

```json
{
	"name": "frontend",
	"version": "0.1.0",
	"private": true,
	"type": "module",
	"scripts": {
		"dev": "bun --bun vite dev",
		"build": "bun --bun vite build",
		"preview": "bun --bun vite preview",
		"check": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json",
		"check:watch": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json --watch",
		"lint": "prettier --check . && eslint .",
		"format": "prettier --write .",
		"format:check": "prettier --check .",
		"test:unit": "bun --bun vitest run",
		"test:e2e": "playwright test"
	},
	"dependencies": {
		"@iconify/svelte": "^5.2.2"
	},
	"devDependencies": {
		"@playwright/test": "^1.49.0",
		"@sveltejs/adapter-static": "^4.0.0",
		"@sveltejs/kit": "^3.0.0",
		"@sveltejs/vite-plugin-svelte": "^7.2.0",
		"@tailwindcss/vite": "^4.3.3",
		"@testing-library/svelte": "^5.4.2",
		"@types/node": "^22.10.0",
		"eslint": "^9.16.0",
		"jsdom": "^29.1.1",
		"prettier": "^3.9.5",
		"prettier-plugin-svelte": "^4.1.1",
		"prettier-plugin-tailwindcss": "^0.8.1",
		"svelte": "^5.57.1",
		"svelte-check": "^4.7.5",
		"tailwindcss": "^4.3.3",
		"typescript": "^6.0.3",
		"vite": "^8.1.5",
		"vitest": "^4.1.10"
	},
	"packageManager": "bun@1.4.0",
	"imports": {
		"#lib/*": "./src/lib/*"
	}
}
```

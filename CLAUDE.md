# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Commands

```bash
npm run dev      # Start dev server (Turbopack, outputs to .next/dev)
npm run build    # Production build (Turbopack by default)
npm run start    # Start production server
npm run lint     # Run ESLint (next build no longer lints automatically)
```

## Architecture

Next.js 16 App Router project with React 19.2, TypeScript, and Tailwind CSS v4.

- `app/` — App Router. All layouts and pages are Server Components by default. Add `'use client'` only for components needing state, event handlers, or browser APIs.
- `app/layout.tsx` — Root layout (required; wraps all routes with `<html>` and `<body>`)
- `app/page.tsx` — Home route (`/`)
- `public/` — Static assets served from `/`
- `@/*` — Path alias resolving to the project root (configured in `tsconfig.json`)

## Next.js 16 breaking changes

**Turbopack is the default bundler.** `next dev` and `next build` use Turbopack. To use Webpack: `next build --webpack`. Custom webpack configs will cause build failures unless you opt out.

**All Request-time APIs are async only** — no synchronous fallback:
- `cookies()`, `headers()`, `draftMode()` must be awaited
- `params` and `searchParams` in pages/layouts are now `Promise<...>` — always `await` them
- Run `npx next typegen` to generate `PageProps`/`LayoutProps`/`RouteContext` helper types

**`middleware` renamed to `proxy`.** Rename `middleware.ts` → `proxy.ts` and the exported function to `proxy`. The `edge` runtime is not supported in `proxy` — keep using `middleware.ts` for edge runtime needs.

**Parallel route slots require `default.js`.** Builds fail without explicit `default.js` in every `@slot` directory.

**`next lint` command removed.** Use `npm run lint` (ESLint CLI directly).

**`revalidateTag` requires a second `cacheLife` argument** (e.g. `revalidateTag('posts', 'max')`). Use `updateTag` in Server Actions for immediate cache refresh.

**`cacheComponents` replaces `experimental.ppr` and `experimental.dynamicIO`** for Partial Prerendering.

**`middleware` config flags renamed:** `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`.

**Removed:** AMP support, `serverRuntimeConfig`/`publicRuntimeConfig` (use env vars), `next/legacy/image`, `images.domains` (use `images.remotePatterns`).

**`next/image` defaults changed:** `minimumCacheTTL` is now 4 hours (was 60s), `qualities` defaults to `[75]` only, local images with query strings require `images.localPatterns.search` config, max 3 redirects by default.

**`dev` and `build` use separate output dirs.** Dev outputs to `.next/dev`; they can run concurrently.

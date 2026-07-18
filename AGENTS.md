# AGENTS.md

Guidance for coding agents working in this repository. It complements `README.md`
with the setup, commands, and conventions needed to make correct changes.

## Project Overview

This is an LLM chat application template that runs on **Cloudflare Workers** and
uses **Workers AI** to stream chat responses over Server-Sent Events (SSE). The
Worker serves a static frontend and exposes a single JSON API endpoint.

- Runtime: Cloudflare Workers (edge, not Node.js). Only `nodejs_compat` APIs are
  available; do not assume a full Node.js runtime at request time.
- Language: TypeScript (backend) and plain JavaScript (frontend).

## Project Structure & Entry Points

```
/
├── public/                     # Static assets served via the ASSETS binding
│   ├── index.html              # Chat UI markup and inline CSS
│   └── chat.js                 # Frontend logic: sends requests, parses SSE stream
├── src/
│   ├── index.ts                # Worker entry point (default export) + /api/chat handler
│   └── types.ts                # Env and ChatMessage type definitions
├── worker-configuration.d.ts   # Generated Worker/binding types (do not edit by hand)
├── wrangler.jsonc              # Worker config: name, bindings, compatibility settings
├── tsconfig.json               # TypeScript configuration
└── package.json                # Dependencies and scripts
```

- **Backend entry point:** `src/index.ts` — the default export's `fetch` handler
  routes requests. Anything under `/api/` is treated as an API route; the only
  implemented route is `POST /api/chat`. All other paths are forwarded to the
  static `ASSETS` binding.
- **Frontend entry point:** `public/index.html`, which loads `public/chat.js`.
- **Model & prompt:** `MODEL_ID` and `SYSTEM_PROMPT` are constants at the top of
  `src/index.ts`.

## Setup

Requires Node.js v18+ and a Cloudflare account with Workers AI access.

```bash
npm install        # install dependencies
npm run cf-typegen # generate worker-configuration.d.ts from wrangler.jsonc bindings
```

Run `npm run cf-typegen` again whenever you change bindings in `wrangler.jsonc`.

## Build, Test, Lint & Typecheck Commands

Only the scripts below exist (see `package.json`); do not invent others.

- **Type-check + config validation:** `npm run check`
  (runs `tsc --noEmit && wrangler deploy --dry-run`).
- **Type-check only:** `npx tsc --noEmit`.
- **Tests:** `npm test` (Vitest via `@cloudflare/vitest-pool-workers`). Note:
  there are currently no test files in the repo, so this reports "No test files
  found". Place new tests in `**/*.{test,spec}.ts`; `tsconfig.json` excludes the
  `test` directory from the main type-check.
- **Local dev server:** `npm run dev` (or `npm start`) → http://localhost:8787.
  Even local dev calls Workers AI and incurs Cloudflare usage charges.
- **Deploy:** `npm run deploy`.
- **Live logs:** `npx wrangler tail`.

There is **no dedicated linter or formatter** configured (no ESLint/Prettier
config, no lint script). Use `npm run check` as the primary correctness gate
before committing. Match the existing code style rather than reformatting files.

Always run `npm run check` before opening a PR.

## Coding Conventions

- **TypeScript is strict** (`strict: true` in `tsconfig.json`). Keep the code
  type-safe; avoid `any` and unsafe casts. Use the `Env` and `ChatMessage`
  interfaces in `src/types.ts` and extend them there when adding bindings.
- The Worker export uses `satisfies ExportedHandler<Env>`; preserve that pattern.
- Indentation is 2 spaces; strings use double quotes; statements end with
  semicolons. Existing files use JSDoc-style block comments for modules and
  functions — follow the surrounding style and keep comments minimal.
- Add new Cloudflare bindings in `wrangler.jsonc`, mirror them in the `Env`
  interface, and regenerate types with `npm run cf-typegen`.
- The frontend (`public/chat.js`) is dependency-free vanilla JS/DOM. Keep it that
  way unless a change requires otherwise.

## Notes & Gotchas

- **Streaming contract:** the backend returns the raw Workers AI response
  (`returnRawResponse: true`), and `public/chat.js` parses newline-delimited JSON
  chunks (reading `jsonData.response`). Changing the model options or response
  format on either side can break streaming — keep the two in sync.
- **AI Gateway** integration is present but commented out in `src/index.ts`;
  enable it by uncommenting the `gateway` block and setting a real gateway ID.
- **`worker-configuration.d.ts` is generated.** Do not edit it manually; run
  `npm run cf-typegen` instead. It is referenced by `tsconfig.json` `types`.
- No pre-commit hooks or CI workflows are configured in this repo.
- Scope changes narrowly and preserve the compatibility settings in
  `wrangler.jsonc` (`compatibility_date`, `nodejs_compat`,
  `global_fetch_strictly_public`) unless a change specifically requires updating
  them.

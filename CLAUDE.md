# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies + generate Prisma client + run migrations
npm run setup

# Development server (Windows-compatible)
npm run dev

# Run tests
npm test

# Run a single test file
npx vitest run src/path/to/file.test.ts

# Prisma: generate client after schema changes
npx prisma generate

# Prisma: create and apply a new migration
npx prisma migrate dev --name <migration-name>

# Reset the database
npm run db:reset

# Build for production
npm run build
```

## Architecture Overview

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates them with live preview and code editing in the browser.

### Key Architectural Concepts

**Virtual File System** (`src/lib/file-system.ts`)  
All files are stored in-memory as a `VirtualFileSystem` class — nothing is written to disk. It serializes to/from JSON for DB persistence. The client holds file state and sends it with each chat request; the server mutates it via AI tool calls and returns the updated state.

**AI Generation Pipeline** (`src/app/api/chat/route.ts`)  
- Uses Vercel AI SDK `streamText` with Anthropic provider
- Claude is given two tools: `str_replace_editor` (edit files) and `file_manager` (create/delete/rename)
- `maxSteps: 40` allows multi-turn tool use in a single request
- Falls back to a mock provider (generates a static demo component) when `ANTHROPIC_API_KEY` is not set
- On `onFinish`, saves messages + serialized file system to the DB if a `projectId` is present

**Live Preview** (`src/components/preview/`)  
- Renders an iframe with `srcdoc` — fully sandboxed, no server involvement
- `jsx-transformer.ts` transpiles JSX to browser-executable JS using Babel standalone
- An import map resolves `@/` aliases and React/ReactDOM to CDN URLs
- Auto-detects entry point: looks for `App.jsx`, `index.jsx`, etc.

**Authentication** (`src/lib/auth.ts`, `src/middleware.ts`)  
- JWT-based sessions stored in httpOnly cookies, 7-day expiration (Jose + bcrypt)
- Middleware guards `/api/projects` and `/api/filesystem` routes
- Anonymous users can generate components; their work is tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts` so it can be saved after sign-up/login

**Data Model** (`prisma/schema.prisma`)  
The database schema is defined in `prisma/schema.prisma` — reference it whenever you need to understand the structure of data stored in the database. SQLite with two models: `User` and `Project`. Projects store `messages` and `data` (the serialized VirtualFileSystem) as JSON strings. Prisma client is generated into `src/generated/prisma/`.

**UI Layout** (`src/components/`)  
Three-panel resizable layout: Chat (35%) | Preview+Code editor (65%). The code view splits into a file tree and Monaco editor. Radix UI primitives + Tailwind CSS v4.

### Important Notes

- `node-compat.cjs` is required at startup (`NODE_OPTIONS=--require`) to patch Node.js compatibility issues with certain dependencies — do not remove it from the dev/build scripts.
- The Prisma client is output to `src/generated/prisma/` (not the default location) — import from there.
- When the `ANTHROPIC_API_KEY` env var is missing, the app silently uses the mock provider. Set it in `.env` to use real Claude.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates code into a virtual (in-memory) file system, which is then compiled and rendered in an iframe preview.

## Commands

```bash
npm run setup         # First-time setup: install deps + prisma generate + migrate
npm run dev           # Start dev server (uses Turbopack)
npm run build         # Production build
npm run lint          # ESLint
npm test              # Run all tests with Vitest
npm test -- --watch   # Watch mode
npm run db:reset      # Reset SQLite database
```

> All Next.js commands internally use `NODE_OPTIONS='--require ./node-compat.cjs'` for Node.js compatibility — this is handled by the npm scripts.

## Environment

Copy `.env.example` to `.env`. Set `ANTHROPIC_API_KEY` to use Claude; if omitted, the app falls back to a `MockLanguageModel` that generates static placeholder components.

## Architecture

### Data Flow

1. User sends a chat message → `POST /api/chat`
2. The chat route serializes the current `VirtualFileSystem` state and streams a `streamText` response using the Vercel AI SDK
3. Claude calls two tools during generation: `str_replace_editor` (edit file content) and `file_manager` (rename/delete files)
4. Tool results are applied to the `VirtualFileSystem` in the client via `FileSystemContext`
5. On stream completion, the updated file system + messages are persisted to the database via a server action
6. The preview iframe re-compiles and renders the virtual files using `@babel/standalone`

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` is a Map-based in-memory file system. Files never touch disk. It serializes to/from JSON for persistence in the `Project.data` database column and for passing state to the AI chat route.

### AI Integration

- **`src/app/api/chat/route.ts`** — the only AI endpoint; uses `claude-haiku` (10k max tokens) with the Vercel AI SDK's `streamText`
- **`src/lib/provider.ts`** — selects between the Anthropic provider and `MockLanguageModel`
- **`src/lib/tools/`** — Zod schemas defining the two AI tools (`str_replace_editor`, `file_manager`)
- **`src/lib/prompts/generation.tsx`** — system prompt instructing Claude to produce Tailwind-styled React components using the `@/` import alias

### State Management

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — holds the `VirtualFileSystem` instance and selected file; consumed by editor, file tree, and chat
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, handles tool call results and anonymous session state

### Auth & Database

- JWT sessions via `jose` stored in httpOnly cookies (7-day expiry); helpers in `src/lib/auth.ts`
- Prisma with SQLite (`prisma/dev.db`); schema has `User` → `Project` (one-to-many, cascade delete)
- Server actions in `src/actions/` handle auth (sign up/in/out) and project CRUD
- `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes

### Path Aliases

`@/` resolves to `src/`. Generated components should always use this alias for imports.

## Testing

Tests use Vitest + jsdom + Testing Library. Test files live next to source in `__tests__/` subdirectories.

```bash
npm test -- src/lib/__tests__/file-system.test.ts   # Run a single test file
```

## Key Conventions

- Shadcn UI components live in `src/components/ui/` (new-york style, neutral base, CSS variables)
- Tailwind CSS v4 with oklch color tokens in `src/app/globals.css`
- No ESLint config beyond `next/core-web-vitals` — run `npm run lint` before committing

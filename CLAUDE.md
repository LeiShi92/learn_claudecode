# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server
npm run dev

# Run tests
npx vitest

# Run a single test file
npx vitest src/components/chat/__tests__/ChatInterface.test.tsx

# Run tests in watch mode
npx vitest --watch

# Database migrations (after schema changes)
npx prisma migrate dev

# Regenerate Prisma client
npx prisma generate
```

## Architecture

This is **UIGen** — an AI-powered React component generator. Users describe components in a chat, Claude generates code, and it renders live in an iframe preview.

### AI Integration

- `src/app/api/chat/route.ts` — The chat API endpoint. Uses Vercel AI SDK's `streamText` with two tools: `str_replace_editor` (for creating/editing files) and `file_manager` (for rename/delete). Tool calls are handled client-side via `onToolCall`.
- `src/lib/provider.ts` — Returns the language model. If `ANTHROPIC_API_KEY` is not set, falls back to `MockLanguageModel` (a static demo that simulates tool calls). Real model is `claude-haiku-4-5`.
- `src/lib/prompts/generation.tsx` — System prompt for component generation.

### Virtual File System

All generated code lives in memory — nothing is written to disk. The `VirtualFileSystem` class (`src/lib/file-system.ts`) manages a tree of `FileNode` objects. It supports CRUD, rename, serialization/deserialization (for API transport and DB storage), and text editor commands (`viewFile`, `replaceInFile`, `insertInFile`).

The VFS is serialized as `Record<string, FileNode>` for JSON transport. The server reconstructs it from the client-sent `files` payload on each request, runs tool calls against it, then saves the result to the DB on finish.

### Context Architecture

Two React contexts wrap the main UI:

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — Owns the `VirtualFileSystem` instance. Exposes CRUD operations and `handleToolCall`, which applies incoming AI tool calls (from `str_replace_editor` and `file_manager`) to the VFS.
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — Wraps Vercel AI SDK's `useChat`. Serializes the current VFS state into each chat request body and dispatches tool calls to `handleToolCall`.

### Preview System

`src/lib/transform/jsx-transformer.ts` transforms JSX/TSX files using Babel Standalone, creates blob URLs for each file, builds an `importMap` for the browser, and assembles a full HTML document. Third-party packages are resolved via `esm.sh`. Missing local imports get placeholder stub modules. The preview runs in an `<iframe>` via `src/components/preview/PreviewFrame.tsx`.

### Auth & Persistence

- Custom JWT auth using `jose` — no third-party auth library. Sessions are stored in httpOnly cookies (`auth-token`). See `src/lib/auth.ts`.
- `src/middleware.ts` — Protects `/api/*` routes, redirects unauthenticated requests for project-specific pages.
- Database: Prisma + SQLite (`prisma/dev.db`). Two models: `User` (email/password) and `Project` (stores `messages` and `data` as JSON strings, `userId` is optional to allow anonymous projects).
- Anonymous work is tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts` so it can be saved after sign-up.

### Server Actions

`src/actions/` contains Next.js server actions: `create-project`, `get-project`, `get-projects`. These interact with Prisma directly.

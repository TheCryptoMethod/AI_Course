# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** is an AI-powered React component generator with live preview. Users describe components in natural language and Claude generates them in real-time using a virtual file system, Monaco editor, and sandboxed iframe preview.

## Commands

```bash
# First-time setup
npm run setup          # install deps + prisma generate + run migrations

# Development
npm run dev            # start dev server with Turbopack (localhost:3000)
npm run dev:daemon     # same, but backgrounded with logs to logs.txt

# Production
npm run build
npm start

# Database
npm run db:reset       # destructive — force reset SQLite database

# Quality
npm run lint
npm run test           # Vitest in watch mode
npx vitest run         # single run (CI-friendly)

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

Requires `ANTHROPIC_API_KEY` in `.env`. Without it, the app falls back to a `MockLanguageModel` that returns static component code.

All npm scripts use `NODE_OPTIONS='--require ./node-compat.cjs'` to patch Node 25+ SSR compatibility (removes `globalThis.localStorage`/`sessionStorage` on server to prevent broken Web Storage API detection).

## Architecture

### Component Generation Flow

1. User sends a prompt via the chat interface
2. `/api/chat` streams a response from Claude using the Vercel AI SDK
3. Claude is given a system prompt ([src/lib/prompts/generation.tsx](src/lib/prompts/generation.tsx)) and two tools:
   - `str_replace_editor` — create/view/edit files in the virtual FS
   - `file_manager` — rename or delete files
4. Tool calls are intercepted by `FileSystemContext` and applied to the in-memory virtual file system
5. File changes propagate in real-time to the Monaco editor and preview iframe

### Virtual File System

`VirtualFileSystem` ([src/lib/file-system.ts](src/lib/file-system.ts)) is a pure in-memory tree structure with no disk I/O. It serializes to/from JSON for persistence (stored in SQLite via Prisma). The root is `/` and the AI always writes `/App.jsx` as the component entry point.

### Preview System

`PreviewFrame` ([src/components/preview/PreviewFrame.tsx](src/components/preview/PreviewFrame.tsx)) renders components inside a sandboxed `<iframe>`. Babel Standalone transpiles JSX in-browser, Tailwind CSS is injected, and an import map resolves `@/` aliases to other virtual files.

### State Management

- **`FileSystemContext`** ([src/lib/contexts/file-system-context.tsx](src/lib/contexts/file-system-context.tsx)) — owns virtual FS state and applies AI tool call results
- **`ChatContext`** ([src/lib/contexts/chat-context.tsx](src/lib/contexts/chat-context.tsx)) — wraps Vercel AI SDK's `useChat` hook, bridges tool results back to the file system

### Routes

- `/` — landing page / main content
- `/[projectId]` — loads a saved project
- `/api/chat` — streaming AI chat endpoint

### Authentication & Persistence

JWT tokens in httpOnly cookies (`src/lib/auth.ts`). Authenticated users get their projects saved to SQLite (Prisma schema in `prisma/schema.prisma`: `User` → `Project`). Projects store the serialized virtual FS and chat message history. Server actions in `src/actions/` handle project CRUD. The Prisma client is generated into `src/generated/prisma/`.

### Main UI Layout

`main-content.tsx` splits the screen: 35% chat panel (left) / 65% editor+preview panel (right). The right panel has tabs — **Preview** (iframe) and **Code** (30% file tree + 70% Monaco editor).

### Testing

Tests live in `__tests__/` directories next to the code they test. Vitest + jsdom + React Testing Library. Path aliases (`@/`) work in tests via `vite-tsconfig-paths`.

## Key Constraints for AI-Generated Components

The system prompt instructs Claude to:
- Always create `/App.jsx` as the root entry point
- Use Tailwind CSS for all styling (no hardcoded styles)
- Use `@/` import aliases for cross-file imports within the virtual FS
- Not import external packages that aren't available in the iframe context

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run build        # Build production bundle
npm run lint         # Run ESLint
npm run test         # Run all tests with Vitest
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Force reset Prisma database
```

**Single test file:**
```bash
npm test -- src/lib/__tests__/file-system.test.ts
```

**Watch mode:**
```bash
npm test -- --watch
```

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to a `MockLanguageModel` that returns static responses — useful for development without burning API quota.

## Architecture

UIGen is a Next.js 15 (App Router) app where users describe React components in a chat interface and get a live preview. There are no files written to disk during generation — everything lives in a virtual in-memory file system.

### Data flow

1. User sends a message → `POST /api/chat` → Vercel AI SDK streams a response from Anthropic Claude (Haiku 4.5)
2. Claude calls tools (`file-manager`, `str-replace`) to create or modify files
3. Tool calls update `VirtualFileSystem` (via `FileSystemContext`)
4. `PreviewFrame` picks up the new file tree, transforms JSX to browser-executable code via Babel standalone, and renders it in an `<iframe>` with an import map
5. If the user is authenticated, the project (messages + file system) is serialized and saved to SQLite via Prisma

### Key modules

| Path | Role |
|------|------|
| `src/lib/file-system.ts` | `VirtualFileSystem` class — all in-memory file operations |
| `src/lib/provider.ts` | Loads the Anthropic model; `MockLanguageModel` fallback |
| `src/lib/transform/jsx-transformer.ts` | Compiles JSX → iframe HTML using Babel standalone + import maps |
| `src/lib/contexts/` | `ChatProvider` (chat state + AI integration) and `FileSystemProvider` (file tree state) |
| `src/lib/tools/` | AI tool definitions: `file-manager` (create/delete files) and `str-replace` (edit files) |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/app/api/chat/route.ts` | Streaming chat endpoint |
| `src/actions/` | Server actions for project CRUD (load, save, create) |

### State management

Both `ChatContext` and `FileSystemContext` are React Contexts wrapping most of the app. `ChatProvider` depends on `FileSystemProvider` — it calls `useFileSystem()` to apply tool call results. The contexts are set up in `src/app/main-content.tsx`.

### Database

SQLite + Prisma. Two models:
- `User` — email/password auth (bcrypt), owns many projects
- `Project` — stores `messages` (JSON), `fileData` (serialized `VirtualFileSystem`), and optional `userId`

Auth is JWT-based via `src/lib/auth.ts`, stored in cookies.

### UI structure

Three-panel layout (resizable via `react-resizable-panels`):
1. **Chat panel** — `ChatInterface` with `MessageList` + `MessageInput`
2. **Editor panel** — Monaco editor (`CodeEditor`) with `FileTree`
3. **Preview panel** — `PreviewFrame` (iframe)

UI primitives come from Shadcn UI (Radix-based) in `src/components/ui/`.

### Path alias

`@/*` maps to `./src/*` (set in `tsconfig.json`). Use this everywhere instead of relative imports that traverse up more than one level.

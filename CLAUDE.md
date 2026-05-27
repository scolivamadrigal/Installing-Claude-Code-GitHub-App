# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Reset database
npm run db:reset

# After changing prisma/schema.prisma
npx prisma migrate dev --name <migration-name>
npx prisma generate
```

## Architecture

UIGen is a Next.js 15 App Router app where users chat with Claude to generate React components that render live in a sandboxed preview.

### Request flow

1. User types a prompt → `ChatInterface` → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route calls `getLanguageModel()` (real Claude or `MockLanguageModel` when no API key) and streams a response via Vercel AI SDK `streamText`
3. Claude calls two tools during generation:
   - `str_replace_editor` — create/edit files via `VirtualFileSystem` methods (`viewFile`, `createFileWithParents`, `replaceInFile`, `insertInFile`)
   - `file_manager` — rename/delete files
4. Tool call results stream back to the client; `ChatContext` (`src/lib/contexts/chat-context.tsx`) calls `handleToolCall` on `FileSystemContext` to apply mutations to the in-memory VFS
5. `PreviewFrame` re-renders whenever `refreshTrigger` increments: it calls `createImportMap` which Babel-transpiles every file in the VFS into blob URLs, builds an import map, and injects them into an `<iframe>` srcdoc
6. For authenticated users, `onFinish` in the route handler persists messages + VFS state to SQLite via Prisma

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is the core data structure — an in-memory tree with a flat `Map<path, FileNode>` for O(1) lookups. It is instantiated on the server in the API route (to let Claude operate on it during a request) and separately maintained on the client inside `FileSystemContext`. The client-side instance is the source of truth for the preview; the server-side instance is reconstructed from the serialized `files` payload on each POST.

### Preview pipeline

`createImportMap` in `src/lib/transform/jsx-transformer.ts`:
- Transpiles all `.js/.jsx/.ts/.tsx` files with `@babel/standalone` (React + optional TypeScript presets)
- Creates blob URLs for each and maps path variants (`/Foo`, `Foo`, `@/Foo`, without extension, etc.)
- Third-party imports are rewritten to `https://esm.sh/<pkg>`
- Missing local imports get a silent placeholder component
- CSS files are inlined via a `<style>` tag
- Syntax errors surface as a styled error panel inside the iframe instead of crashing

The preview HTML (`createPreviewHTML`) includes Tailwind CDN and an import-map `<script>` so components can use Tailwind classes without a build step.

### Auth

Custom JWT auth (`src/lib/auth.ts`) using `jose`. Sessions are stored in an `httpOnly` cookie (`auth-token`, 7-day expiry). `verifySession` in `src/middleware.ts` guards `/api/projects` and `/api/filesystem`. `getSession()` is `server-only`. The `use-auth` hook and `HeaderActions` component drive the sign-up/sign-in dialog on the client.

Anonymous users can generate freely; their work is tracked in `sessionStorage` (`anon-work-tracker.ts`) and can be claimed when they sign up.

### Data model

Projects store `messages` and `data` (VFS snapshot) as JSON strings in SQLite. `userId` is nullable so anonymous projects are representable, though only authenticated users have their projects persisted server-side.

### Key conventions

- The system prompt (`src/lib/prompts/generation.tsx`) requires every project to have `/App.jsx` as its entrypoint and mandates `@/` import aliases for local files
- `src/components/ui/` contains shadcn/ui primitives — don't edit these manually
- Prisma client is generated to `src/generated/prisma/` (not the default location); import from there or via `src/lib/prisma.ts`
- Tests use Vitest + jsdom + React Testing Library; test files live next to their source in `__tests__/` directories

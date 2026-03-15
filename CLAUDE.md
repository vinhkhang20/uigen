# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

- Use comments sparingly. Only comment complex code.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack
npm run dev:daemon   # Start dev server in background (logs to logs.txt)

# Build & Production
npm run build
npm run start

# Database
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (destructive)
npx prisma generate  # Regenerate Prisma client after schema changes
npx prisma migrate dev --name <name>  # Create and apply a migration

# Testing
npm test             # Run all tests (Vitest)
npx vitest run <path>  # Run a single test file

# Linting
npm run lint
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in chat, Claude AI generates code using tools, and the output is rendered in a sandboxed iframe.

### Data Flow

1. User sends a message in `ChatInterface`
2. `ChatProvider` (`src/lib/contexts/chat-context.tsx`) forwards it to `POST /api/chat`
3. The API route streams a response from Claude (`claude-haiku-4-5`) using Vercel AI SDK
4. Claude uses two tools to write code: `str_replace_editor` and `file_manager`
5. Tool calls manipulate a **VirtualFileSystem** (in-memory, no disk writes)
6. The serialized VFS state is sent back to the client
7. `PreviewFrame` renders files in a sandboxed iframe using Babel standalone for JSX compilation and Skypack CDN for React
8. On finish, if authenticated, project state (messages + VFS JSON) is persisted to SQLite via Prisma

### Key Modules

- **`src/lib/file-system.ts`** — In-memory VirtualFileSystem. All component code lives here; serialized as JSON into the database `Project.data` field.
- **`src/lib/provider.ts`** — Claude model setup. Falls back to `MockLanguageModel` (generates demo components) when no API key is present.
- **`src/lib/tools/str-replace.ts`** — `str_replace_editor` tool: view, create, str_replace, insert, undo_edit commands.
- **`src/lib/tools/file-manager.ts`** — `file_manager` tool: rename and delete commands.
- **`src/lib/transform/jsx-transformer.ts`** — Converts VFS files to an iframe-renderable HTML blob with an import map, injecting Tailwind CSS and Babel standalone.
- **`src/lib/auth.ts`** — JWT session management via `jose`. Server-only.
- **`src/actions/index.ts`** — Server actions: signUp, signIn, signOut, getUser (passwords hashed with bcrypt).

### Layout

The main UI (`src/app/main-content.tsx`) uses `react-resizable-panels`:
- **Left panel**: `ChatInterface` (chat history + message input)
- **Right panel**: tabbed between `PreviewFrame` (live iframe) and `FileTree` + `CodeEditor` (Monaco)

### Database Schema

Always reference `prisma/schema.prisma` for the authoritative data model. Two models (SQLite):
- `User`: id, email, password (bcrypt), timestamps
- `Project`: id, name, userId (FK), `messages` (JSON), `data` (JSON serialized VFS), timestamps

### Authentication

JWT-based; tokens stored in cookies. Middleware (`src/middleware.ts`) protects `/api/chat` and project routes. Anonymous users can generate components but cannot persist them.

### Testing

Tests live in `__tests__` directories alongside the code they test. Uses Vitest + Testing Library with jsdom. The `vitest.config.mts` sets up `@vitejs/plugin-react` and `vite-tsconfig-paths`.

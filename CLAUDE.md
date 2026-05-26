# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** is an AI-powered React component generator with live preview. It uses Claude (via the Anthropic API) to generate React components based on user descriptions in natural language.

**Key Architecture:**
- Next.js 15 full-stack application with App Router
- React 19 + TypeScript frontend
- Virtual file system (no files written to disk) — components are generated in memory
- Prisma ORM with SQLite database for user authentication and project persistence
- Streaming text generation with tool use via Vercel AI SDK and Anthropic API
- Mock provider fallback when API key is not configured

## Development Commands

```bash
# Install dependencies and initialize database
npm run setup

# Start development server with Turbopack
npm run dev

# Start development server in background (logs to logs.txt)
npm run dev:daemon

# Build for production
npm run build

# Start production server
npm start

# Lint code
npm lint

# Run tests
npm test

# Run specific test file
npm test -- src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database (destructive)
npm run db:reset

# Generate Prisma client
npx prisma generate

# Run database migrations
npx prisma migrate dev
```

## Architecture & Key Patterns

### Virtual File System
The app operates on a **virtual file system** — not real files on disk. The `VirtualFileSystem` class (src/lib/file-system.ts) manages an in-memory file tree. Projects are serialized as JSON and stored in the database. The system is designed to hold generated React components that users can preview and iterate on.

### Message-Driven Component Generation
The chat interface collects user prompts and sends them to `/api/chat` along with:
- **messages**: conversation history
- **files**: serialized virtual file system
- **projectId**: for authenticated users to save their work

The API route uses `streamText()` from the Vercel AI SDK with **tool use** to generate components:
- **str_replace_editor**: creates, views, and edits files in the virtual FS
- **file_manager**: renames and deletes files

Each tool call result is streamed back to the client for real-time UI updates.

### AI Provider Pattern
`src/lib/provider.ts` exports `getLanguageModel()`:
- **Real provider**: Uses `anthropic(MODEL)` from @ai-sdk/anthropic if `ANTHROPIC_API_KEY` is set
- **Mock provider**: Falls back to `MockLanguageModel` for canned responses if no key or placeholder key is set
- The mock provider generates realistic tool calls for testing without API costs

The system prompt is injected at API time (not build time) with ephemeral cache control enabled.

### Authentication & Sessions
JWT-based sessions stored in HTTP-only cookies:
- `createSession()` signs a JWT with userId and email, expires in 7 days
- `getSession()` verifies the JWT from cookies
- Anonymous users can still use the app but cannot persist projects to the database
- On sign-in, any anonymous work (tracked in localStorage) is migrated to a persisted project

### UI Layout
The main interface is a **ResizablePanelGroup** with three regions:
1. **Left panel (35%)**: Chat interface with scrollable message list and input
2. **Right panel (65%)**: Toggles between preview and code views
   - **Preview**: Renders components via iframe (PreviewFrame)
   - **Code**: File tree (30%) + Monaco code editor (70%)

The file tree shows the virtual FS structure; the code editor is Monaco (via @monaco-editor/react).

### Styling
- **Tailwind CSS v4** with custom color palette (neutral base color)
- **shadcn/ui** components (Radix UI primitives + Tailwind styling)
- **React Resizable Panels** for draggable dividers
- **Lucide icons** for UI elements

## Database Schema

Two main models:

```
User
  id, email, password, createdAt, updatedAt
  relationships: Project[]

Project
  id, name, userId?, messages (JSON string), data (JSON string), createdAt, updatedAt
  relationships: User
```

Projects can be anonymous (userId = null) or owned by a user. Messages and data are stored as JSON strings for flexibility.

## Configuration Files

- **tsconfig.json**: Strict mode enabled, `@/*` path alias points to `./src/*`
- **.eslintrc.json**: Extends Next.js ESLint config
- **vitest.config.mts**: jsdom test environment, React plugin, tsconfig paths
- **components.json**: shadcn/ui configuration with New York style, RSC enabled
- **next.config.ts**: Turbopack configuration (pins workspace root to prevent module resolution hijacking)
- **postcss.config.mjs**: Tailwind CSS v4 PostCSS plugin

## Important Notes

- **Do not run `npm audit fix`** — dependencies are pinned to specific compatible versions
- The app works without an API key (mock provider) but will return canned responses
- Set a real `ANTHROPIC_API_KEY` in `.env` to generate actual components with Claude
- The virtual file system serializes to JSON; no actual files are created on the user's machine
- Authenticated projects are persisted; anonymous work is tracked in localStorage and migrated on sign-in
- The app uses prompt caching (ephemeral) on the system message to reduce API costs for repeated interactions

## Testing

Tests use **Vitest** with React Testing Library:
- Test files live in `__tests__` folders adjacent to source files
- Run all: `npm test`
- Run specific file: `npm test -- path/to/file.test.tsx`
- Environment is jsdom (browser-like)

Key areas with tests:
- Chat components (MessageList, MessageInput, ChatInterface, MarkdownRenderer)
- File tree component
- Virtual file system logic
- Context providers (chat, file system)


# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe UI components in natural language chat, and an AI agent generates the code with real-time preview.

**Tech stack**: Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS v4, Prisma + SQLite, AI SDK (Vercel), Anthropic Claude.

## Common Commands

```bash
# Initial setup (install deps + Prisma generate + migrate)
npm run setup

# Development server with Turbopack
npm run dev

# Production build
npm run build

# Run tests (Vitest)
npm run test

# Run a specific test file
npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts

# Reset database (destructive)
npm run db:reset
```

All scripts require `NODE_OPTIONS='--require ./node-compat.cjs'` for dev/build/start — this is baked into the npm scripts.

## Architecture

### High-Level Structure

```
src/
├── app/              # Next.js App Router
│   ├── [projectId]/  # Project page (chat + editor + preview)
│   ├── api/chat/     # Streaming chat endpoint (AI agent)
│   ├── page.tsx      # Home (redirects to /[projectId] or shows auth)
│   └── layout.tsx    # Root layout + metadata
├── components/       # React components
│   ├── chat/         # ChatInterface, MessageList, MessageInput, MarkdownRenderer
│   ├── editor/       # CodeEditor (Monaco), FileTree
│   ├── preview/      # PreviewFrame (iframe-based live preview)
│   ├── auth/         # AuthDialog, SignInForm, SignUpForm
│   └── ui/           # Radix UI primitives (shadcn-style)
├── lib/
│   ├── provider.ts   # AI model provider (Anthropic with mock fallback)
│   ├── file-system.ts # In-memory virtual file system
│   ├── auth.ts       # JWT session management (jose, httpOnly cookies)
│   ├── transform/    # Babel-based JSX/TSX → JS transformer
│   ├── prompts/      # AI system prompts for code generation
│   └── tools/        # AI tool definitions (str-replace, file-manager)
├── actions/          # Server actions (auth, CRUD)
├── hooks/            # Custom hooks (use-auth)
└── middleware.ts     # Route protection for /api/projects, /api/filesystem
```

### Key Flows

**AI Chat → Code Generation**: The `/api/chat/route.ts` endpoint uses the Vercel AI SDK to stream responses from an AI agent. The agent is given file system tools (str-replace, file-manager) and a system prompt from `lib/prompts/generation.tsx`. It generates React components into the in-memory virtual file system (`lib/file-system.ts`).

**Live Preview**: `PreviewFrame` renders generated components in an iframe. Code is transformed via Babel (`lib/transform/jsx-transformer.ts`), wrapped with an import map for React, and loaded as a blob URL in the iframe. Tailwind CDN is injected for styling.

**Authentication**: JWT-based sessions stored in httpOnly cookies via `lib/auth.ts`. Server actions (`src/actions/index.ts`) handle signIn/signUp/signOut using bcrypt for password hashing. Middleware protects `/api/projects` and `/api/filesystem` routes. Anonymous users are tracked via `lib/anon-work-tracker.ts`.

### Database

Prisma + SQLite. Two models:
- **User**: id, email, password (bcrypt), timestamps
- **Project**: id, name, userId (nullable for anon), messages (JSON string), data (JSON string), timestamps

Prisma client is generated to `src/generated/prisma` and accessed via `lib/prisma.ts`.

### AI Configuration

- Provider: `@ai-sdk/anthropic` (model: `claude-haiku-4-5`)
- Falls back to a mock model when `ANTHROPIC_API_KEY` is not set
- Uses the Vercel AI SDK `ai` package v6+

### Testing

Vitest with jsdom environment. Config in `vitest.config.ts`. Tests live in `__tests__` directories adjacent to source files (e.g., `lib/transform/__tests__/`, `lib/contexts/__tests__/`).

### Environment Variables

Key `.env` variables: `ANTHROPIC_API_KEY`, `JWT_SECRET`, `DATABASE_URL` (SQLite file path).

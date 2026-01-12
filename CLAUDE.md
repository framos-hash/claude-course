# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Anthropic API) to generate React components through a chat interface, displays them in a live preview using a virtual file system, and persists projects for authenticated users.

**Tech Stack**: Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS v4, Prisma (SQLite), Anthropic Claude AI, Vercel AI SDK

## Commands

### Development
```bash
npm run dev              # Start development server with Turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
npm run build            # Build for production
npm run start            # Start production server
npm run lint             # Run ESLint
npm run test             # Run Vitest tests
```

### Database
```bash
npm run setup            # Install deps + generate Prisma client + run migrations
npm run db:reset         # Reset database (force)
npx prisma generate      # Generate Prisma client (outputs to src/generated/prisma)
npx prisma migrate dev   # Create and apply new migration
```

### Testing
```bash
npm run test             # Run all tests
npm run test -- <file>   # Run specific test file
```

## Architecture

### Virtual File System (VFS)

The core of UIGen is a **client-side virtual file system** that stores generated React components in memory without writing to disk. Key aspects:

- **Location**: `src/lib/file-system.ts` - Implements `VirtualFileSystem` class
- **Structure**: Tree-based structure using `Map<string, FileNode>` for O(1) lookups
- **Persistence**: Serializes to JSON and stores in database (`Project.data` field) for authenticated users
- **Operations**: Create, read, update, delete, rename files/directories with automatic parent directory creation
- **Context**: `src/lib/contexts/file-system-context.tsx` provides React context for VFS operations

The VFS is sent with every chat API request and reconstructed server-side to enable AI tool calls.

### AI Integration Flow

1. **Client**: User sends message via `ChatProvider` (`src/lib/contexts/chat-context.tsx`)
2. **API Route**: `/api/chat/route.ts` receives message and serialized VFS
3. **LLM Provider**: `src/lib/provider.ts` returns either Claude Haiku 4.5 (if API key exists) or `MockLanguageModel` (static fallback)
4. **Tools**: AI has access to two tools:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) - Create files, view files, replace strings, insert lines
   - `file_manager` (`src/lib/tools/file-manager.ts`) - Rename and delete files
5. **System Prompt**: `src/lib/prompts/generation.tsx` instructs AI to create React components with Tailwind CSS, always starting with `/App.jsx`
6. **Tool Execution**: Tools modify VFS server-side
7. **Streaming**: Results stream back to client via Vercel AI SDK
8. **Client Updates**: `onToolCall` handler in `FileSystemProvider` applies VFS changes to client-side state
9. **Persistence**: `onFinish` hook saves messages and VFS to database for authenticated users

### Component Preview System

Generated components are previewed in an iframe:

- **Location**: `src/components/preview/PreviewFrame.tsx`
- **Transform**: `src/lib/transform/jsx-transformer.ts` uses Babel standalone to transpile JSX/TSX to JavaScript in the browser
- **Import Map**: Dynamically creates ES Module import maps pointing to `esm.sh` CDN for React and dependencies
- **Blob URLs**: Transformed code is converted to blob URLs and injected into iframe via script tags
- **Auto-refresh**: Preview updates automatically when VFS changes (via `refreshTrigger` in `FileSystemProvider`)

### Authentication & Projects

- **Session Management**: JWT-based sessions using `jose` library (`src/lib/auth.ts`)
  - Token stored in `session` cookie (httpOnly, secure, sameSite: lax)
  - Middleware (`src/middleware.ts`) validates session on protected routes
- **Anonymous Users**: Can use the app without signing up
  - Work tracked in localStorage (`src/lib/anon-work-tracker.ts`)
  - Prompted to sign up to save work
- **Database Schema** (`prisma/schema.prisma`):
  - `User`: email, hashed password (bcrypt)
  - `Project`: name, userId (nullable), messages (JSON string), data (JSON string for VFS)
  - Prisma client generated to `src/generated/prisma` (non-standard location)
- **Actions**: Server actions in `src/actions/` handle project CRUD operations

### Directory Structure

```
src/
├── actions/              # Server actions (create/get projects)
├── app/                  # Next.js App Router pages
│   ├── api/chat/         # Streaming chat endpoint
│   ├── [projectId]/      # Dynamic project page
│   └── main-content.tsx  # Main app layout with split panels
├── components/
│   ├── auth/             # Sign in/up forms, auth dialog
│   ├── chat/             # Chat interface, message list/input, markdown renderer
│   ├── editor/           # Code editor (Monaco), file tree
│   ├── preview/          # Preview iframe component
│   └── ui/               # shadcn/ui components
├── lib/
│   ├── contexts/         # React contexts (chat, file-system)
│   ├── prompts/          # AI system prompts
│   ├── tools/            # AI tool definitions for Vercel AI SDK
│   ├── transform/        # JSX/TSX to JS transformation (Babel)
│   ├── file-system.ts    # Virtual file system implementation
│   ├── provider.ts       # LLM provider (Claude or mock)
│   ├── auth.ts           # Session management
│   └── prisma.ts         # Prisma client singleton
└── middleware.ts         # Route protection
```

## Key Implementation Details

### Mock Provider Behavior
When `ANTHROPIC_API_KEY` is not set, the `MockLanguageModel` returns static components in a multi-step flow:
1. Creates `/App.jsx`
2. Creates component file (Counter, Form, or Card based on keywords)
3. Enhances component with styling update
4. Final summary message

### File System Context Pattern
The `FileSystemProvider` uses a React Context to share a single VFS instance across components. The `refreshTrigger` state variable forces re-renders when VFS changes, since the VFS itself is mutable and doesn't trigger React updates.

### Import Alias
All non-library imports use `@/` alias mapping to `src/` directory (configured in `tsconfig.json`). Generated components must follow this convention.

### Tailwind CSS v4
This project uses Tailwind CSS v4 (still in beta). Configuration is in `src/app/globals.css` using `@import` directives rather than a `tailwind.config.js` file.

## Development Notes

- Preview frame may show errors if components have invalid JSX syntax or missing dependencies
- Database is SQLite (`prisma/dev.db`) - good for development, should be replaced with PostgreSQL for production
- Session tokens expire based on `maxAge` in `src/lib/auth.ts` (currently 7 days)
- AI tools expect specific command structures - see `src/lib/tools/` for schemas
- Max API route duration is 120 seconds (`maxDuration = 120` in chat route)
- Monaco Editor is client-side only - wrapped with dynamic import and `ssr: false`

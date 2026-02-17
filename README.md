<p align="center">
  <img src="https://raw.githubusercontent.com/yourusername/colony/main/docs/assets/colony-logo.png" alt="Colony Logo" width="120" />
</p>

<h1 align="center">Colony</h1>

<p align="center">
  <strong>The Rails for Electron</strong>
</p>

<p align="center">
  Build desktop apps with TypeScript. Define once. Generate GUI, CLI, and API.
</p>

<p align="center">
  <a href="https://github.com/yourusername/colony/actions"><img src="https://img.shields.io/github/actions/workflow/status/yourusername/colony/ci.yml?branch=main" alt="Build Status"></a>
  <a href="https://www.npmjs.com/package/@aspect/colony"><img src="https://img.shields.io/npm/v/@aspect/colony" alt="npm"></a>
  <a href="https://www.npmjs.com/package/@aspect/colony"><img src="https://img.shields.io/node/v/@aspect/colony" alt="Node Version"></a>
  <a href="https://github.com/yourusername/colony/blob/main/LICENSE"><img src="https://img.shields.io/github/license/yourusername/colony" alt="License"></a>
</p>

---

## What is Colony?

Colony is a TypeScript framework for building Electron desktop applications. Define your commands, queries, and data models once—Colony generates a desktop GUI, command-line interface, and TypeScript API that all stay in sync.

```typescript
import { createApp, command, query, entity } from '@aspect/colony';
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';
import { z } from 'zod';

const app = createApp({
  name: 'tasks',
  cli: { command: 'tasks' },
  window: { title: 'Task Manager' },
});

// Define your data model with Drizzle
const tasks = sqliteTable('tasks', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  completed: integer('completed', { mode: 'boolean' }).default(false),
});

entity(app, { table: tasks });

// Define commands with Zod schemas
const addTask = command(app, {
  name: 'add',
  description: 'Add a new task',
  params: z.object({ title: z.string() }),
  returns: z.object({ id: z.number(), title: z.string() }),
  
  async execute(ctx, { title }) {
    const [task] = await ctx.db.insert(tasks).values({ title }).returning();
    return task;
  },
});

// Define queries
const listTasks = query(app, {
  name: 'list',
  params: z.object({ completed: z.boolean().optional() }),
  returns: z.array(TaskSchema),
  
  async execute(ctx, { completed }) {
    return ctx.db.select().from(tasks).where(
      completed !== undefined ? eq(tasks.completed, completed) : undefined
    );
  },
});
```

From this single specification, Colony generates:

- **Desktop GUI** — Electron app with React, command palette, and type-safe hooks
- **CLI** — `tasks add "Buy milk"` and `tasks list --json` for agents and scripts
- **TypeScript API** — Import and call commands directly with full type safety

---

## Why Colony?

Building Electron apps today means stitching together React, state management, IPC patterns, database access, and packaging manually. The boilerplate is significant. The patterns are inconsistent. Security is an afterthought.

Colony provides an opinionated, integrated framework:

- **Unified specification** — One definition generates GUI, CLI, and API
- **Type-safe IPC** — Zod schemas validate inputs; TypeScript flows end-to-end
- **Built-in database** — Drizzle ORM with Bun's native SQLite
- **Command palette** — Every command searchable via Cmd+K
- **Security by default** — Context isolation, sandboxing, input validation
- **One-command distribution** — Build signed, notarized apps for Mac/Win/Linux

Colony is designed for AI-assisted development. TypeScript's strong typing provides context for code generation. The declarative spec is easy for language models to reason about. The CLI enables AI agents to interact with your app programmatically.

---

## Quick Start

Colony requires [Bun](https://bun.sh) v1.0 or later.

```bash
# Install Colony CLI
bun add -g @aspect/colony

# Create a new project
colony new myapp
cd myapp

# Start development
colony dev
```

The dev server runs your app with hot reload. Try it:

```bash
# Use the CLI
myapp add "My first task"
myapp list --json

# Or press Cmd+K in the app to open command palette
```

---

## Core Concepts

### Entities

Define your data model with Drizzle ORM. No code generation—your schema IS TypeScript.

```typescript
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';
import { entity } from '@aspect/colony';

const notes = sqliteTable('notes', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  content: text('content'),
  createdAt: integer('created_at', { mode: 'timestamp' }).default(sql`(unixepoch())`),
});

entity(app, { table: notes });
```

### Commands

Commands are actions that modify state. They become CLI commands, GUI actions, and API functions.

```typescript
import { command } from '@aspect/colony';
import { z } from 'zod';

const createNote = command(app, {
  name: 'create',
  description: 'Create a new note',
  params: z.object({
    title: z.string().min(1),
    content: z.string().optional(),
  }),
  returns: NoteSchema,
  entities: [notes], // For cache invalidation
  
  async execute(ctx, params) {
    const [note] = await ctx.db.insert(notes).values(params).returning();
    return note;
  },
});
```

This generates:

```bash
# CLI
myapp create "Meeting Notes" --content "Discuss Q1 plans"
myapp create "Meeting Notes" --json

# React hook
const mutation = useCommand(createNote);
mutation.mutate({ title: 'Meeting Notes' });

# Direct API
const note = await createNote.execute(ctx, { title: 'Meeting Notes' });
```

### Queries

Queries are read operations. They support caching and automatic invalidation.

```typescript
import { query } from '@aspect/colony';

const searchNotes = query(app, {
  name: 'search',
  params: z.object({ query: z.string() }),
  returns: z.array(NoteSchema),
  entities: [notes],
  cache: { ttl: 60 },
  
  async execute(ctx, { query }) {
    return ctx.db.select().from(notes)
      .where(like(notes.title, `%${query}%`));
  },
});
```

In React:

```typescript
const { data, isLoading } = useQuery(searchNotes, { query: 'meeting' });
```

When a command targeting the same entity executes, queries automatically refetch.

### Windows

Define windows with React components. Colony handles Electron setup.

```typescript
import { window } from '@aspect/colony';
import { useQuery, useCommand } from '@aspect/colony/react';

const MainWindow = window(app, {
  name: 'main',
  default: true,
  config: { width: 1200, height: 800 },
  
  component: () => {
    const { data } = useQuery(listNotes, {});
    const createMutation = useCommand(createNote);
    
    return (
      <div className="flex flex-col h-screen">
        <TitleBar title="Notes" />
        <main className="flex-1 p-4">
          <NoteList notes={data ?? []} />
        </main>
        <CommandPalette />
      </div>
    );
  },
});
```

---

## Framework Components

Colony provides pre-built components that integrate with your spec.

### Command Palette

Exposes all commands in a searchable interface. Press `Cmd+K` (or `Ctrl+K`).

```typescript
import { CommandPalette } from '@aspect/colony/react';

// In your window component
<CommandPalette />
```

### Dialogs

Type-safe dialogs that match system style.

```typescript
const { confirm, prompt } = useDialogs();

const confirmed = await confirm({
  title: 'Delete Note',
  message: 'This cannot be undone.',
  destructive: true,
});
```

### Title Bar & Status Bar

Native-feeling chrome for your app.

```typescript
<TitleBar title="My App" draggable />

<StatusBar>
  <StatusBar.Item>{noteCount} notes</StatusBar.Item>
</StatusBar>
```

---

## System Integration

### Menu Bar

Define native menus that trigger your commands.

```typescript
const app = createApp({
  // ...
  menu: {
    template: [
      {
        label: 'File',
        submenu: [
          { label: 'New Note', accelerator: 'CmdOrCtrl+N', command: 'create' },
          { type: 'separator' },
          { role: 'quit' },
        ],
      },
    ],
  },
});
```

### System Tray

Run your app in the background.

```typescript
const app = createApp({
  // ...
  tray: {
    icon: './assets/tray-icon.png',
    tooltip: 'My App',
    menu: [
      { label: 'Open', click: 'showMainWindow' },
      { label: 'Quit', role: 'quit' },
    ],
  },
});
```

### Auto-Updates

Automatic updates via GitHub releases.

```typescript
const app = createApp({
  // ...
  updates: {
    provider: 'github',
    owner: 'yourusername',
    repo: 'myapp',
  },
});
```

---

## CLI Reference

```bash
colony new <n>       # Create new project
colony dev              # Development with hot reload
colony build            # Build for all platforms
colony build --mac      # Build for macOS only
colony build --win      # Build for Windows only

colony db push          # Push schema changes to database
colony db studio        # Open Drizzle Studio

colony generate         # Regenerate all generated code
colony spec export      # Export spec as JSON Schema
```

---

## Technology Stack

| Component | Technology | Why |
|-----------|------------|-----|
| Runtime | [Bun](https://bun.sh) | TypeScript-native, built-in SQLite, fast |
| Desktop | [Electron](https://www.electronjs.org/) | Cross-platform desktop apps |
| Build | [electron-vite](https://electron-vite.org/) | Vite for Electron, fast HMR, proper multi-process bundling |
| UI | [React](https://react.dev) | Component model, ecosystem (no meta-frameworks) |
| Database | [Drizzle](https://orm.drizzle.team/) | TypeScript-first, no codegen |
| Schema | [Zod](https://zod.dev) | Runtime validation, type inference |
| State | [TanStack Query](https://tanstack.com/query) | Async state, caching |
| Styling | [Tailwind CSS](https://tailwindcss.com) | Utility-first CSS |

Colony intentionally uses "just React" with Vite rather than meta-frameworks like Next.js. Server-side rendering, API routes, and file-based routing don't apply in Electron's local application context. Colony's opinions live in the specification layer, not the React framework.

---

## Optional Interfaces

### MCP Server

Expose commands as tools for AI agents.

```typescript
const app = createApp({
  mcp: true, // Enable MCP server
});
```

```bash
myapp-mcp  # Start MCP server
```

### REST API

Expose commands as HTTP endpoints.

```typescript
const app = createApp({
  rest: true, // Enable REST API
});
```

```bash
myapp-api  # Start API server
curl -X POST localhost:3000/api/create -d '{"title":"Note"}'
```

---

## Security

Colony enforces Electron security best practices:

- **Context isolation** — Renderer can't access Node.js directly
- **Sandbox** — Renderer runs in restricted environment
- **Input validation** — All IPC inputs validated with Zod
- **Preload script** — Only safe IPC methods exposed

---

## Part of the Hive Ecosystem

Colony is the TypeScript/Electron counterpart to [Hive](https://github.com/yourusername/hive), a Python framework for terminal-native applications. Both frameworks share:

- Conformance-Driven Specification (CDS) principles
- Triple interface paradigm (UI + CLI + API)
- Entity-based cache invalidation
- Optional MCP/REST extensions

| Framework | Language | Primary UI | Target |
|-----------|----------|------------|--------|
| Hive | Python | TUI (Textual) | Terminal-native apps |
| Colony | TypeScript | Desktop GUI (Electron) | Desktop-native apps |

---

## Documentation

- [Getting Started](https://colony.dev/docs/getting-started)
- [Tutorial: Building a Note App](https://colony.dev/docs/tutorial)
- [API Reference](https://colony.dev/docs/api)
- [Security Guide](https://colony.dev/docs/security)

---

## Examples

See the [examples](./examples) directory:

- **tasks** — Simple task manager
- **notes** — Note-taking app with search
- **api-client** — App integrating external API

---

## Contributing

Contributions welcome! See [CONTRIBUTING.md](./CONTRIBUTING.md).

```bash
git clone https://github.com/yourusername/colony.git
cd colony
bun install
bun test
```

---

## License

MIT © [Your Name](https://github.com/yourusername)

---

<p align="center">
  <em>Build desktop apps at the speed of thought.</em>
</p>

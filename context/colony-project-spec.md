# Colony: A Framework for Desktop-Agent-Native TypeScript Applications

**Project Specification Document**

Version 0.1.0 | January 2026

---

## Executive Summary

Colony is a TypeScript framework for building desktop applications that serve three distinct consumers from a single specification: graphical user interfaces for human users, command-line interfaces for terminal-native AI agents and power users, and TypeScript APIs for programmatic integration. The framework follows the same "define once, expose everywhere" philosophy as Hive, adapted for the desktop application context.

Colony targets Electron as the desktop runtime, React as the UI framework, and Bun as the TypeScript runtime and toolchain. Where Hive Python generates CLI + TUI + Python API for terminal-native applications, Colony generates Desktop GUI + CLI + TypeScript API for desktop-native applications.

The framework addresses a significant gap in the desktop application ecosystem: there is no "Rails for Electron." Developers building Electron applications today piece together React, state management, IPC patterns, database access, and distribution tooling manually. The boilerplate is significant, the patterns are undocumented, and the result is often inconsistent and insecure. Colony provides an opinionated, integrated framework that handles these concerns, enabling developers to focus on application logic.

Colony is designed for the emerging paradigm of hyper-personal software—applications built rapidly by individuals or small teams using AI assistance, tailored to specific needs rather than mass-market appeal. The framework optimizes for AI-assisted development: TypeScript's strong typing provides context for code generation, the declarative specification layer is easily reasoned about by language models, and the CLI interface enables AI agents to interact with Colony applications programmatically.

---

## Vision: The Hyper-Personal Software Era

We are entering an era where sophisticated computer users will build their own software. The combination of powerful language models, code generation capabilities, and agent-based development tools is democratizing software creation. The abstraction level is rising: from machine code to assembly to high-level languages to frameworks to AI-assisted development.

Colony is infrastructure for this future. It provides the abstractions that enable a single developer, working with AI assistance, to build complete, professional desktop applications in days rather than months. The framework handles the accidental complexity of Electron development—IPC, security, packaging, updates—so developers can focus on essential complexity: what their application actually does.

This vision aligns with a broader thesis: personal computing is ascending. While web applications serve the masses, desktop applications serve individuals with specific, sophisticated needs. Developer tools, creative applications, productivity systems, data analysis tools—these categories thrive on the desktop because they require deep integration with the local system, offline capability, and performance that web applications cannot match.

The major AI companies have validated this thesis. Claude Desktop, ChatGPT for Mac, Cursor, Perplexity—the most sophisticated AI interfaces are Electron applications. Colony aims to make building such applications accessible to individual developers.

---

## Core Philosophy

### The Triple Interface Paradigm (Desktop Edition)

Colony applications expose functionality through three interfaces, each optimized for its primary consumer.

The **desktop graphical interface** serves as the primary interface for human users. Built with React and rendered by Electron, it provides rich visual interaction, keyboard navigation, and deep system integration. Windows, dialogs, menus, and system tray integration create a native-feeling experience.

The **command-line interface** serves AI agents and power users who operate through the terminal. Every command supports structured output (JSON) enabling agents to parse results reliably. The CLI provides the same capabilities as the GUI, enabling automation and scripting.

The **TypeScript API** provides direct programmatic access for developers integrating Colony applications into larger systems. Functions mirror command signatures with full type safety, enabling IDE autocomplete and static analysis.

Two additional interfaces extend reach when needed.

The **MCP server** (optional) exposes commands as tools for AI agents operating outside terminal environments. This enables AI assistants like Claude Desktop to invoke application commands through the Model Context Protocol.

The **REST API** (optional) exposes commands as HTTP endpoints for web integrations and cross-language access.

### Conformance-Driven Specification

Colony applies the same CDS principles as Hive: the TypeScript code serves as a machine-readable specification from which all interfaces are derived. Zod schemas define data contracts, decorated functions define operations, and the framework generates consistent implementations across all access patterns.

The specification captures command names, parameter types, return types, documentation, and relationships to data entities. From this specification, the framework generates Electron IPC handlers, React hooks, CLI commands, and optional MCP/REST interfaces.

### Desktop-First, Agent-Accessible

The desktop GUI is the primary interface, reflecting Colony's focus on rich human interaction. However, every capability is also accessible through the CLI, ensuring that AI agents can operate Colony applications as effectively as human users.

This is critical for the hyper-personal software thesis. An individual developer building with AI assistance needs their tools to be usable by their AI collaborator. Colony applications are inherently agent-accessible because they expose CLI interfaces alongside their GUIs.

---

## Architecture Overview

### Technology Stack

Colony builds upon a carefully selected set of technologies that share philosophical alignment and optimize for AI-assisted development.

**Bun** serves as the TypeScript runtime and toolchain. Bun provides native TypeScript execution without compilation, built-in SQLite, a fast package manager, test runner, and bundler. Its all-in-one nature reduces toolchain complexity. Anthropic's investment in Bun (Claude Code uses it) signals confidence in its future.

**Electron** provides the desktop application shell. Electron enables cross-platform desktop applications using web technologies. Despite criticism of its resource usage, Electron remains the pragmatic choice for cross-platform desktop development. The framework enforces security best practices (context isolation, sandboxing) by default.

**electron-vite** provides the build tooling for Electron applications. It handles bundling for all three Electron contexts (main process, preload scripts, renderer process) with Vite's speed and developer experience. Colony uses electron-vite rather than raw Vite or webpack because it understands Electron's multi-process architecture natively, providing proper hot module replacement for both main and renderer processes, correct externalization of Electron and Node.js built-ins, and optimized production builds. This is the standard tooling choice in the Electron ecosystem.

**React** provides the UI framework for the renderer process. React's component model, ecosystem, and familiarity make it the natural choice for Electron applications. Colony uses React 18+ with concurrent features. Notably, Colony does not use React meta-frameworks like Next.js or TanStack Start—these are designed for server-rendered web applications with features (SSR, SSG, API routes, file-based routing) that don't apply in Electron's local application context. The renderer is "just React" with Vite, keeping the stack simple and avoiding framework features that fight Electron's model.

**Drizzle ORM** provides the data layer. Drizzle is TypeScript-first with no code generation step—the schema IS TypeScript. It's lightweight (no separate engine process like Prisma), has first-class Bun support, and provides a SQL-like API that's transparent rather than magical.

**SQLite** serves as the default database via Bun's built-in `bun:sqlite`. SQLite is the standard for desktop application data storage—reliable, fast, zero-configuration, and embedded. For applications requiring it, PostgreSQL is supported via Drizzle's dialect system.

**Zod** provides runtime validation and schema definition. Zod schemas serve as the single source of truth for data shapes, providing runtime validation, TypeScript type inference, and JSON Schema export. The `drizzle-zod` integration derives Zod schemas from Drizzle tables automatically.

**TanStack Query** (React Query) provides async state management in the renderer process. It handles caching, background refetching, and optimistic updates for data fetched via IPC.

**Zustand** provides client-side state management for UI state that doesn't come from the database.

**Tailwind CSS** provides styling. Utility-first CSS works excellently in component-based applications and is well-understood by language models.

**Commander.js** provides CLI parsing. It's mature, well-documented, and has excellent TypeScript support.

**FastMCP** (optional) provides MCP server generation, matching Hive's approach.

**Hono** (optional) provides REST API generation. Hono is a lightweight, fast web framework that runs on Bun natively, making it preferable to Express for this context.

### System Architecture

Colony applications have a natural split between the Electron main process (Node/Bun environment with system access) and renderer process (browser environment with React). The framework provides a type-safe bridge between them.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Specification Layer                          │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────────┐   │
│  │ command() │ │  query()  │ │ entity()  │ │   window()    │   │
│  └───────────┘ └───────────┘ └───────────┘ └───────────────┘   │
│                              │                                  │
│                    Application Registry                         │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Framework Core                           │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐   │
│  │ Spec Parser     │ │ Zod Integration │ │ Type Inference  │   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                               │
       ┌───────────────────────┼───────────────────────┐
       ▼                       ▼                       ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│ Main Process    │  │ Renderer        │  │ CLI Generator       │
│ Generator       │  │ Generator       │  │                     │
│ (IPC handlers)  │  │ (React hooks)   │  │                     │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
       │                       │                       │
       ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Runtime Layer                            │
│                                                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│  │     Main Process        │  │     Renderer Process        │  │
│  │  ┌─────────────────┐    │  │  ┌─────────────────────┐    │  │
│  │  │ Database (Bun)  │    │  │  │ React Application   │    │  │
│  │  │ File System     │    │  │  │ TanStack Query      │    │  │
│  │  │ Native APIs     │    │  │  │ Zustand State       │    │  │
│  │  │ IPC Handlers    │◄───┼──┼──│ IPC Client          │    │  │
│  │  └─────────────────┘    │  │  └─────────────────────┘    │  │
│  └─────────────────────────┘  └─────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
       │                                               │
       ▼                                               ▼
┌─────────────────┐                          ┌─────────────────┐
│      CLI        │                          │   Desktop GUI   │
│  myapp export   │                          │   (Electron)    │
│  myapp list     │                          │                 │
└─────────────────┘                          └─────────────────┘
```

### The IPC Bridge

Electron's IPC (Inter-Process Communication) is the critical bridge between main and renderer processes. Colony provides a type-safe abstraction over this bridge, similar to tRPC's approach for HTTP.

Commands and queries defined in the specification layer become:
- IPC handlers in the main process (where database/system access lives)
- Type-safe hooks in the renderer process (where React lives)
- CLI commands (which invoke the same handlers)

The type safety flows end-to-end: Zod schemas validate inputs, TypeScript infers types, and the generated code maintains these types across the IPC boundary.

---

## Specification System Design

### The App Object

Every Colony application begins with an App instance defining global configuration.

```typescript
// src/app.ts
import { createApp } from '@colony/core';

export const app = createApp({
  name: 'taskmaster',
  version: '0.1.0',
  description: 'Task management for humans and agents',
  
  // CLI configuration
  cli: {
    command: 'tasks',
    description: 'Task management CLI',
  },
  
  // Desktop window configuration
  window: {
    title: 'Task Master',
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600,
  },
  
  // Database configuration
  database: 'sqlite://~/.taskmaster/data.db',
  
  // Optional interfaces
  mcp: false,
  rest: false,
});
```

The App object accepts the following configuration:

**name**: The application name, used for packaging and data directories.

**version**: Semantic version string.

**description**: Human-readable description for CLI help and about dialogs.

**cli.command**: The CLI command name. Users invoke the application with this name.

**window**: Default window configuration including title, dimensions, and constraints.

**database**: Database connection URL. Defaults to SQLite in a platform-appropriate location.

**mcp**: Boolean, defaults to false. When true, generates an MCP server.

**rest**: Boolean, defaults to false. When true, generates a REST API via Hono.

### Entity Definitions

Entities define the data model using Drizzle ORM. The `entity()` function registers tables with the application.

```typescript
// src/entities/task.ts
import { entity } from '@colony/core';
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';
import { createInsertSchema, createSelectSchema } from 'drizzle-zod';
import { sql } from 'drizzle-orm';
import { app } from '../app';

// Drizzle table definition - this IS your schema, no codegen
export const tasks = sqliteTable('tasks', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  description: text('description'),
  completed: integer('completed', { mode: 'boolean' }).notNull().default(false),
  priority: text('priority', { enum: ['low', 'medium', 'high'] }).default('medium'),
  createdAt: integer('created_at', { mode: 'timestamp' })
    .notNull()
    .default(sql`(unixepoch())`),
  completedAt: integer('completed_at', { mode: 'timestamp' }),
});

// Zod schemas derived automatically from Drizzle
export const TaskInsert = createInsertSchema(tasks);
export const TaskSelect = createSelectSchema(tasks);

// TypeScript types inferred from Drizzle
export type Task = typeof tasks.$inferSelect;
export type NewTask = typeof tasks.$inferInsert;

// Register with Colony
entity(app, {
  name: 'Task',
  table: tasks,
  schemas: {
    insert: TaskInsert,
    select: TaskSelect,
  },
});
```

The `drizzle-zod` integration is powerful: your Drizzle table definition automatically produces Zod schemas, which provide both runtime validation and TypeScript types. There's no separate schema definition step.

### Command Definitions

Commands represent actions that modify state or interact with external systems.

```typescript
// src/commands/task-commands.ts
import { command } from '@colony/core';
import { z } from 'zod';
import { eq } from 'drizzle-orm';
import { app } from '../app';
import { tasks, Task, TaskInsert } from '../entities/task';

// Define parameter and result schemas with Zod
const AddTaskParams = z.object({
  title: z.string().min(1, 'Title is required'),
  description: z.string().optional(),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
});

const AddTaskResult = z.object({
  task: TaskSelect,
  message: z.string(),
});

export const addTask = command(app, {
  name: 'add',
  description: 'Add a new task',
  params: AddTaskParams,
  returns: AddTaskResult,
  entities: [tasks],
  
  async execute(ctx, params) {
    const [task] = await ctx.db
      .insert(tasks)
      .values({
        title: params.title,
        description: params.description,
        priority: params.priority,
      })
      .returning();
    
    return {
      task,
      message: `Created task #${task.id}: ${task.title}`,
    };
  },
});

const CompleteTaskParams = z.object({
  id: z.number().int().positive(),
});

const CompleteTaskResult = z.object({
  task: TaskSelect,
  message: z.string(),
});

export const completeTask = command(app, {
  name: 'complete',
  description: 'Mark a task as completed',
  params: CompleteTaskParams,
  returns: CompleteTaskResult,
  entities: [tasks],
  
  async execute(ctx, params) {
    const [task] = await ctx.db
      .update(tasks)
      .set({
        completed: true,
        completedAt: new Date(),
      })
      .where(eq(tasks.id, params.id))
      .returning();
    
    if (!task) {
      throw new CommandError(`Task #${params.id} not found`);
    }
    
    return {
      task,
      message: `Completed task #${task.id}: ${task.title}`,
    };
  },
});

const DeleteTaskParams = z.object({
  id: z.number().int().positive(),
  force: z.boolean().default(false),
});

const DeleteTaskResult = z.object({
  id: z.number(),
  message: z.string(),
});

export const deleteTask = command(app, {
  name: 'delete',
  description: 'Delete a task',
  params: DeleteTaskParams,
  returns: DeleteTaskResult,
  entities: [tasks],
  
  // Commands can request confirmation in GUI context
  confirm: (params) => ({
    title: 'Delete Task',
    message: `Are you sure you want to delete task #${params.id}?`,
    destructive: true,
  }),
  
  async execute(ctx, params) {
    const [deleted] = await ctx.db
      .delete(tasks)
      .where(eq(tasks.id, params.id))
      .returning({ id: tasks.id });
    
    if (!deleted) {
      throw new CommandError(`Task #${params.id} not found`);
    }
    
    return {
      id: deleted.id,
      message: `Deleted task #${params.id}`,
    };
  },
});
```

Command configuration includes:

**name**: Command name for CLI and command palette.

**description**: Help text and command palette description.

**params**: Zod schema defining input parameters.

**returns**: Zod schema defining the return type.

**entities**: Array of tables the command interacts with (for cache invalidation).

**confirm**: Optional function returning confirmation dialog configuration for destructive actions.

**execute**: The implementation function receiving context and validated parameters.

### Query Definitions

Queries represent read operations without side effects. They support caching.

```typescript
// src/queries/task-queries.ts
import { query } from '@colony/core';
import { z } from 'zod';
import { eq, desc, and } from 'drizzle-orm';
import { app } from '../app';
import { tasks, TaskSelect } from '../entities/task';

const ListTasksParams = z.object({
  completed: z.boolean().optional(),
  priority: z.enum(['low', 'medium', 'high']).optional(),
  limit: z.number().int().positive().default(100),
});

const ListTasksResult = z.object({
  tasks: z.array(TaskSelect),
  total: z.number(),
});

export const listTasks = query(app, {
  name: 'list',
  description: 'List tasks with optional filters',
  params: ListTasksParams,
  returns: ListTasksResult,
  entities: [tasks],
  cache: { ttl: 60 }, // Cache for 60 seconds
  
  async execute(ctx, params) {
    const conditions = [];
    
    if (params.completed !== undefined) {
      conditions.push(eq(tasks.completed, params.completed));
    }
    if (params.priority) {
      conditions.push(eq(tasks.priority, params.priority));
    }
    
    const where = conditions.length > 0 ? and(...conditions) : undefined;
    
    const results = await ctx.db
      .select()
      .from(tasks)
      .where(where)
      .orderBy(desc(tasks.createdAt))
      .limit(params.limit);
    
    return {
      tasks: results,
      total: results.length,
    };
  },
});

const GetTaskParams = z.object({
  id: z.number().int().positive(),
});

export const getTask = query(app, {
  name: 'get',
  description: 'Get a single task by ID',
  params: GetTaskParams,
  returns: TaskSelect.nullable(),
  entities: [tasks],
  
  async execute(ctx, params) {
    const [task] = await ctx.db
      .select()
      .from(tasks)
      .where(eq(tasks.id, params.id))
      .limit(1);
    
    return task ?? null;
  },
});
```

### Window Definitions

Windows define the application's UI using React components.

```typescript
// src/windows/main-window.tsx
import { window } from '@colony/core';
import { useQuery, useCommand } from '@colony/react';
import { app } from '../app';
import { listTasks, getTask } from '../queries/task-queries';
import { addTask, completeTask, deleteTask } from '../commands/task-commands';

export const MainWindow = window(app, {
  name: 'main',
  default: true, // This is the main window
  
  // Window-specific configuration overrides
  config: {
    title: 'Task Master',
    width: 1200,
    height: 800,
  },
  
  // The React component
  component: function MainWindowComponent() {
    // Type-safe query hook - params and return type inferred from listTasks
    const { 
      data, 
      isLoading, 
      refetch 
    } = useQuery(listTasks, { completed: false });
    
    // Type-safe command hooks
    const addMutation = useCommand(addTask);
    const completeMutation = useCommand(completeTask);
    const deleteMutation = useCommand(deleteTask);
    
    const [newTaskTitle, setNewTaskTitle] = useState('');
    
    const handleAddTask = async () => {
      if (!newTaskTitle.trim()) return;
      
      await addMutation.mutateAsync({ title: newTaskTitle });
      setNewTaskTitle('');
      // Query automatically refetches due to entity invalidation
    };
    
    return (
      <div className="flex flex-col h-screen bg-gray-50">
        {/* Title bar area - can be draggable */}
        <TitleBar title="Task Master" />
        
        {/* Main content */}
        <main className="flex-1 overflow-auto p-6">
          {/* Add task input */}
          <div className="flex gap-2 mb-6">
            <input
              type="text"
              value={newTaskTitle}
              onChange={(e) => setNewTaskTitle(e.target.value)}
              onKeyDown={(e) => e.key === 'Enter' && handleAddTask()}
              placeholder="Add a new task..."
              className="flex-1 px-4 py-2 border rounded-lg"
            />
            <button
              onClick={handleAddTask}
              disabled={addMutation.isPending}
              className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600"
            >
              Add
            </button>
          </div>
          
          {/* Task list */}
          {isLoading ? (
            <div className="flex justify-center py-8">
              <Spinner />
            </div>
          ) : (
            <TaskList
              tasks={data?.tasks ?? []}
              onComplete={(id) => completeMutation.mutate({ id })}
              onDelete={(id) => deleteMutation.mutate({ id })}
            />
          )}
        </main>
        
        {/* Status bar */}
        <StatusBar>
          {data && <span>{data.total} tasks</span>}
        </StatusBar>
        
        {/* Command palette - framework provided */}
        <CommandPalette />
      </div>
    );
  },
});
```

Window configuration includes:

**name**: Window identifier for programmatic access.

**default**: If true, this window opens on application launch.

**config**: Window-specific configuration (dimensions, title, frame options).

**component**: The React component rendering the window content.

### Secondary Windows

Applications can define multiple windows for different purposes.

```typescript
// src/windows/settings-window.tsx
import { window } from '@colony/core';
import { app } from '../app';

export const SettingsWindow = window(app, {
  name: 'settings',
  
  config: {
    title: 'Settings',
    width: 600,
    height: 400,
    resizable: false,
    modal: true, // Opens as modal over main window
  },
  
  component: function SettingsWindowComponent() {
    return (
      <div className="p-6">
        <h1 className="text-xl font-bold mb-4">Settings</h1>
        {/* Settings UI */}
      </div>
    );
  },
});
```

Windows can be opened programmatically:

```typescript
// In a component
const { openWindow } = useWindows();

<button onClick={() => openWindow('settings')}>
  Settings
</button>
```

### Service Definitions

Services represent external API integrations with managed lifecycle.

```typescript
// src/services/api-client.ts
import { service } from '@colony/core';
import { app } from '../app';

export const apiClient = service(app, {
  name: 'api',
  
  // Credentials from system keychain
  credentials: {
    apiKey: 'keychain:myapp/api-key',
  },
  
  // Service factory
  create(credentials) {
    return {
      async fetchData(endpoint: string) {
        const response = await fetch(`https://api.example.com${endpoint}`, {
          headers: { 'Authorization': `Bearer ${credentials.apiKey}` },
        });
        return response.json();
      },
    };
  },
  
  // Cleanup on shutdown
  async dispose(client) {
    // Any cleanup needed
  },
});
```

Services are accessed through the execution context:

```typescript
async execute(ctx, params) {
  const data = await ctx.services.api.fetchData('/users');
  // ...
}
```

---

## Execution Context

Commands, queries, and services receive an execution context providing access to framework capabilities.

```typescript
interface ExecutionContext {
  // Database access (Drizzle)
  db: DrizzleDatabase;
  
  // Registered services
  services: ServiceRegistry;
  
  // Application configuration
  config: AppConfig;
  
  // File system operations
  files: {
    read(path: string): Promise<Buffer>;
    write(path: string, data: Buffer): Promise<void>;
    pick(options?: FilePickerOptions): Promise<string | null>;
    save(data: unknown, format: 'json' | 'csv'): Promise<string>;
  };
  
  // Dialog operations (main process)
  dialogs: {
    message(options: MessageOptions): Promise<void>;
    confirm(options: ConfirmOptions): Promise<boolean>;
    prompt(options: PromptOptions): Promise<string | null>;
  };
  
  // Clipboard access
  clipboard: {
    read(): Promise<string>;
    write(text: string): Promise<void>;
  };
  
  // Notifications
  notifications: {
    show(options: NotificationOptions): Promise<void>;
  };
  
  // Logging
  log: Logger;
}
```

---

## Generated Code

### Main Process (Electron Main)

Colony generates IPC handlers that connect commands and queries to the renderer process.

```typescript
// Generated: src/generated/main/ipc-handlers.ts
import { ipcMain } from 'electron';
import { db } from './database';
import { services } from './services';
import { addTask, completeTask, deleteTask } from '../../commands/task-commands';
import { listTasks, getTask } from '../../queries/task-queries';

const ctx = { db, services, config, files, dialogs, clipboard, notifications, log };

// Command handlers
ipcMain.handle('command:add', async (event, params) => {
  const validated = addTask.params.parse(params);
  const result = await addTask.execute(ctx, validated);
  return addTask.returns.parse(result);
});

ipcMain.handle('command:complete', async (event, params) => {
  const validated = completeTask.params.parse(params);
  const result = await completeTask.execute(ctx, validated);
  return completeTask.returns.parse(result);
});

ipcMain.handle('command:delete', async (event, params) => {
  const validated = deleteTask.params.parse(params);
  const result = await deleteTask.execute(ctx, validated);
  return deleteTask.returns.parse(result);
});

// Query handlers
ipcMain.handle('query:list', async (event, params) => {
  const validated = listTasks.params.parse(params);
  const result = await listTasks.execute(ctx, validated);
  return listTasks.returns.parse(result);
});

ipcMain.handle('query:get', async (event, params) => {
  const validated = getTask.params.parse(params);
  const result = await getTask.execute(ctx, validated);
  return getTask.returns.parse(result);
});
```

### Renderer Process (React Hooks)

Colony generates type-safe React hooks for invoking commands and queries.

```typescript
// Generated: src/generated/renderer/hooks.ts
import { useQuery as useTanstackQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { ipcRenderer } from 'electron';
import type { Command, Query } from '@colony/core';

// Type-safe query hook
export function useQuery<TParams, TResult>(
  query: Query<TParams, TResult>,
  params: TParams,
  options?: UseQueryOptions,
) {
  return useTanstackQuery({
    queryKey: [query.name, params],
    queryFn: () => ipcRenderer.invoke(`query:${query.name}`, params) as Promise<TResult>,
    ...options,
  });
}

// Type-safe command hook with automatic cache invalidation
export function useCommand<TParams, TResult>(
  command: Command<TParams, TResult>,
  options?: UseMutationOptions,
) {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (params: TParams) => 
      ipcRenderer.invoke(`command:${command.name}`, params) as Promise<TResult>,
    onSuccess: () => {
      // Invalidate queries that depend on affected entities
      command.entities.forEach(entity => {
        queryClient.invalidateQueries({ 
          predicate: (query) => query.meta?.entities?.includes(entity.name),
        });
      });
    },
    ...options,
  });
}

// Window management hook
export function useWindows() {
  return {
    openWindow: (name: string, params?: Record<string, unknown>) =>
      ipcRenderer.invoke('window:open', { name, params }),
    closeWindow: () =>
      ipcRenderer.invoke('window:close'),
  };
}
```

### CLI

Colony generates a CLI that invokes the same command handlers.

```typescript
// Generated: src/generated/cli/index.ts
#!/usr/bin/env bun
import { Command } from 'commander';
import { db } from '../main/database';
import { services } from '../main/services';
import { addTask, completeTask, deleteTask } from '../../commands/task-commands';
import { listTasks, getTask } from '../../queries/task-queries';

const ctx = { db, services, config, files, dialogs, clipboard, notifications, log };

const program = new Command()
  .name('tasks')
  .description('Task management for humans and agents')
  .version('0.1.0');

// Add command
program
  .command('add <title>')
  .description('Add a new task')
  .option('-d, --description <text>', 'Task description')
  .option('-p, --priority <level>', 'Priority level', 'medium')
  .option('--json', 'Output as JSON')
  .action(async (title, options) => {
    try {
      const params = { title, description: options.description, priority: options.priority };
      const validated = addTask.params.parse(params);
      const result = await addTask.execute(ctx, validated);
      
      if (options.json) {
        console.log(JSON.stringify(result));
      } else {
        console.log(result.message);
      }
    } catch (error) {
      if (options.json) {
        console.log(JSON.stringify({ error: error.message }));
      } else {
        console.error(`Error: ${error.message}`);
      }
      process.exit(1);
    }
  });

// Complete command
program
  .command('complete <id>')
  .description('Mark a task as completed')
  .option('--json', 'Output as JSON')
  .action(async (id, options) => {
    const params = { id: parseInt(id, 10) };
    const validated = completeTask.params.parse(params);
    const result = await completeTask.execute(ctx, validated);
    
    if (options.json) {
      console.log(JSON.stringify(result));
    } else {
      console.log(result.message);
    }
  });

// List query
program
  .command('list')
  .description('List tasks')
  .option('-c, --completed', 'Show completed tasks')
  .option('-p, --priority <level>', 'Filter by priority')
  .option('-l, --limit <n>', 'Limit results', '100')
  .option('--json', 'Output as JSON')
  .action(async (options) => {
    const params = {
      completed: options.completed,
      priority: options.priority,
      limit: parseInt(options.limit, 10),
    };
    const validated = listTasks.params.parse(params);
    const result = await listTasks.execute(ctx, validated);
    
    if (options.json) {
      console.log(JSON.stringify(result));
    } else {
      // Pretty table output
      console.table(result.tasks.map(t => ({
        ID: t.id,
        Title: t.title,
        Priority: t.priority,
        Status: t.completed ? '✓' : ' ',
      })));
      console.log(`\n${result.total} tasks`);
    }
  });

program.parse();
```

---

## Framework-Provided Components

Colony provides standard components that integrate with the framework.

### Command Palette

The command palette exposes all registered commands in a searchable interface.

```typescript
// Framework-provided component
import { CommandPalette } from '@colony/react';

// In your window component
<CommandPalette
  shortcut="mod+k"  // Cmd+K on Mac, Ctrl+K on Windows/Linux
  placeholder="Type a command..."
/>
```

The command palette automatically:
- Lists all registered commands with descriptions
- Provides fuzzy search
- Shows keyboard shortcuts where defined
- Opens parameter input dialogs for commands with required params
- Handles command execution and error display

### Title Bar

A customizable title bar with window controls.

```typescript
import { TitleBar } from '@colony/react';

<TitleBar
  title="Task Master"
  draggable
  showControls  // min/max/close buttons
/>
```

### Status Bar

A status bar for displaying application state.

```typescript
import { StatusBar } from '@colony/react';

<StatusBar>
  <StatusBar.Item>Ready</StatusBar.Item>
  <StatusBar.Spacer />
  <StatusBar.Item>{taskCount} tasks</StatusBar.Item>
</StatusBar>
```

### Dialogs

Pre-built dialog components that match the system style.

```typescript
import { ConfirmDialog, PromptDialog, MessageDialog } from '@colony/react';

// Declarative usage
<ConfirmDialog
  open={showConfirm}
  title="Delete Task"
  message="Are you sure?"
  onConfirm={handleDelete}
  onCancel={() => setShowConfirm(false)}
/>

// Or imperative via hook
const { confirm, prompt, message } = useDialogs();

const confirmed = await confirm({
  title: 'Delete Task',
  message: 'Are you sure?',
  destructive: true,
});
```

---

## System Integration

### Menu Bar

Colony applications can define native menu bars.

```typescript
// src/app.ts
export const app = createApp({
  // ...
  
  menu: {
    template: [
      {
        label: 'File',
        submenu: [
          { label: 'New Task', accelerator: 'CmdOrCtrl+N', command: 'add' },
          { type: 'separator' },
          { label: 'Export...', command: 'export' },
          { type: 'separator' },
          { role: 'quit' },
        ],
      },
      {
        label: 'Edit',
        submenu: [
          { role: 'undo' },
          { role: 'redo' },
          { type: 'separator' },
          { role: 'cut' },
          { role: 'copy' },
          { role: 'paste' },
        ],
      },
      {
        label: 'View',
        submenu: [
          { label: 'Show Completed', command: 'toggleCompleted', type: 'checkbox' },
          { type: 'separator' },
          { role: 'toggleDevTools' },
        ],
      },
    ],
  },
});
```

### System Tray

Colony applications can run in the system tray.

```typescript
export const app = createApp({
  // ...
  
  tray: {
    icon: './assets/tray-icon.png',
    tooltip: 'Task Master',
    menu: [
      { label: 'Open', click: 'showMainWindow' },
      { type: 'separator' },
      { label: 'Quick Add...', command: 'quickAdd' },
      { type: 'separator' },
      { label: 'Quit', role: 'quit' },
    ],
  },
});
```

### Auto-Updates

Colony integrates with `electron-updater` for automatic updates.

```typescript
export const app = createApp({
  // ...
  
  updates: {
    provider: 'github',
    owner: 'yourusername',
    repo: 'taskmaster',
    autoDownload: true,
    autoInstallOnQuit: true,
  },
});
```

### Global Shortcuts

Register global keyboard shortcuts that work when the app is in background.

```typescript
export const app = createApp({
  // ...
  
  globalShortcuts: {
    'CmdOrCtrl+Shift+T': 'quickAdd',  // Maps to command
    'CmdOrCtrl+Shift+Space': 'showMainWindow',
  },
});
```

---

## Project Structure

A Colony project follows the standard electron-vite structure, with Colony's specification layer on top:

```
myapp/
├── package.json
├── tsconfig.json
├── tsconfig.node.json        # TypeScript config for main process
├── tsconfig.web.json         # TypeScript config for renderer
├── electron.vite.config.ts   # electron-vite configuration
├── colony.config.ts          # Colony-specific configuration
├── src/
│   ├── main/                 # Electron main process (Bun/Node)
│   │   ├── index.ts          # Main entry point
│   │   ├── database.ts       # Database initialization
│   │   └── ipc-handlers.ts   # Generated IPC handlers
│   ├── preload/              # Preload scripts (context bridge)
│   │   └── index.ts          # Exposes safe IPC to renderer
│   ├── renderer/             # Electron renderer (React + Vite)
│   │   ├── index.html        # HTML entry point
│   │   ├── main.tsx          # React entry point
│   │   ├── App.tsx           # Root component
│   │   └── styles.css        # Tailwind entry
│   ├── spec/                 # Colony specification layer
│   │   ├── app.ts            # App definition
│   │   ├── entities/         # Drizzle entities
│   │   │   └── task.ts
│   │   ├── commands/         # Command implementations
│   │   │   └── task-commands.ts
│   │   ├── queries/          # Query implementations
│   │   │   └── task-queries.ts
│   │   └── windows/          # Window definitions
│   │       ├── main-window.tsx
│   │       └── settings-window.tsx
│   ├── components/           # Shared React components
│   │   └── task-list.tsx
│   ├── services/             # External service clients
│   │   └── api-client.ts
│   └── generated/            # Generated code (gitignored)
│       ├── ipc-handlers.ts   # Main process IPC handlers
│       ├── hooks.ts          # React hooks for IPC
│       └── cli.ts            # CLI implementation
├── assets/
│   ├── icon.png
│   ├── icon.icns             # macOS icon
│   ├── icon.ico              # Windows icon
│   └── tray-icon.png
├── tests/
│   └── commands.test.ts
├── out/                      # electron-vite build output
│   ├── main/
│   ├── preload/
│   └── renderer/
└── dist/                     # electron-builder output (installers)
```

The structure separates concerns according to Electron's process model:

- **src/main/** — Main process code (Node.js/Bun APIs, database, system access)
- **src/preload/** — Preload scripts (bridge between main and renderer)
- **src/renderer/** — Renderer process code (React, UI, Vite dev server)
- **src/spec/** — Colony specification layer (entities, commands, queries, windows)
- **src/generated/** — Code generated from the specification (gitignored, regenerated on build)

---

## CLI Tooling

Colony provides a CLI for project management.

```bash
colony new <name>           # Create a new project
colony dev                  # Start development with hot reload
colony build                # Build for distribution
colony build --mac          # Build for macOS
colony build --win          # Build for Windows
colony build --linux        # Build for Linux
colony publish              # Publish to GitHub releases

colony db migrate <msg>     # Generate database migration
colony db push              # Push schema changes to database
colony db studio            # Open Drizzle Studio for database inspection

colony spec export          # Export specification as JSON Schema
colony spec diff <a> <b>    # Compare specifications

colony generate             # Regenerate all generated code
```

### Development Mode

`colony dev` wraps electron-vite's development server with Colony-specific enhancements:

- **Renderer hot reload** — Vite HMR for instant React updates without losing state
- **Main process restart** — Automatic restart when main process code changes
- **Preload script rebuild** — Automatic rebuild and reload of preload scripts
- **Code generation** — Regenerates IPC handlers and hooks when spec changes
- **Database sync** — Pushes schema changes to database on entity file changes
- **TypeScript checking** — Runs type checking in watch mode
- **Source maps** — Full source map support for debugging both processes

Under the hood, `colony dev` runs:
1. `colony generate --watch` — Watches spec files, regenerates on change
2. `electron-vite dev` — Runs the Vite dev server and Electron

```bash
$ colony dev

🐝 Colony development server starting...

  electron-vite v2.x.x
  
  ➜ Main:       Watching src/main/ for changes
  ➜ Preload:    Watching src/preload/ for changes  
  ➜ Renderer:   http://localhost:5173
  ➜ CLI:        myapp (available in shell)
  ➜ Spec:       Watching src/spec/ for changes
  
  Shortcuts:
    r  — Restart main process
    o  — Open DevTools
    c  — Clear console
    q  — Quit

✓ Ready in 1.2s
```

### Build Process

`colony build` uses electron-vite for bundling and Electron Builder for packaging:

```bash
$ colony build --mac

🐝 Colony build starting...

  Step 1/4: Generating code from specification...
  ✓ Generated IPC handlers, React hooks, CLI

  Step 2/4: Building with electron-vite...
  ✓ Main process built (423ms)
  ✓ Preload scripts built (156ms)
  ✓ Renderer built (2.1s)

  Step 3/4: Packaging with electron-builder...
  ✓ macOS x64 (12.3s)
  ✓ macOS arm64 (11.8s)

  Step 4/4: Code signing and notarization...
  ✓ Signed with Developer ID
  ✓ Notarized with Apple

  Output:
    dist/MyApp-1.0.0-mac-x64.dmg (89 MB)
    dist/MyApp-1.0.0-mac-arm64.dmg (87 MB)
    dist/MyApp-1.0.0-mac-x64.zip (86 MB)
    dist/MyApp-1.0.0-mac-arm64.zip (84 MB)

✓ Build complete in 28.4s
```

---

## Distribution

Colony uses Electron Builder for creating distributable applications.

### Build Configuration

```typescript
// colony.config.ts
import { defineConfig } from '@colony/core';

export default defineConfig({
  build: {
    appId: 'com.yourcompany.taskmaster',
    productName: 'Task Master',
    
    mac: {
      category: 'public.app-category.productivity',
      icon: 'assets/icon.icns',
      target: ['dmg', 'zip'],
      hardenedRuntime: true,
      notarize: true,  // Requires Apple credentials
    },
    
    win: {
      icon: 'assets/icon.ico',
      target: ['nsis', 'portable'],
    },
    
    linux: {
      icon: 'assets/icon.png',
      target: ['AppImage', 'deb'],
      category: 'Utility',
    },
  },
});
```

### Code Signing

Colony handles code signing configuration:

```typescript
export default defineConfig({
  build: {
    // macOS signing
    mac: {
      identity: 'Developer ID Application: Your Name (TEAM_ID)',
      notarize: {
        teamId: 'TEAM_ID',
      },
    },
    
    // Windows signing
    win: {
      certificateFile: process.env.WIN_CERT_FILE,
      certificatePassword: process.env.WIN_CERT_PASSWORD,
    },
  },
});
```

---

## Security

Colony enforces Electron security best practices by default.

### Context Isolation

Renderer processes run with context isolation enabled. The preload script exposes only the IPC bridge, not the full Electron API.

```typescript
// Generated preload script
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('colony', {
  invoke: (channel: string, ...args: unknown[]) => {
    const validChannels = ['command:*', 'query:*', 'window:*'];
    if (validChannels.some(pattern => matchChannel(channel, pattern))) {
      return ipcRenderer.invoke(channel, ...args);
    }
    throw new Error(`Invalid channel: ${channel}`);
  },
  on: (channel: string, callback: (...args: unknown[]) => void) => {
    const validChannels = ['notification:*', 'update:*'];
    if (validChannels.some(pattern => matchChannel(channel, pattern))) {
      ipcRenderer.on(channel, (event, ...args) => callback(...args));
    }
  },
});
```

### Sandboxing

Renderer processes run sandboxed by default. System access goes through the main process via IPC.

### Input Validation

All IPC inputs are validated with Zod schemas before execution. Invalid inputs are rejected before reaching command/query implementations.

---

## Comparison: Colony vs Hive

| Aspect | Hive (Python) | Colony (TypeScript) |
|--------|---------------|---------------------|
| Primary UI | TUI (Textual) | Desktop GUI (Electron/React) |
| Runtime | Python + asyncio | Bun |
| ORM | SQLModel | Drizzle |
| Schema | Pydantic / Type hints | Zod |
| CLI Framework | Typer | Commander.js |
| IPC | N/A | Electron IPC |
| Distribution | pip / PyPI | Electron Builder |
| Target | Terminal-native apps | Desktop-native apps |

Both frameworks share:
- CDS principles (spec-driven, derived implementations)
- Triple interface paradigm (UI + CLI + API)
- Optional MCP/REST extensions
- Entity-based cache invalidation
- Type-safe end-to-end

---

## Implementation Roadmap

### Phase 1: Core Framework

**Milestone 1.1: Project Structure and Tooling**

Set up the monorepo structure with packages for core, react, and cli. Implement `colony new` scaffolding with electron-vite as the build foundation. Configure Bun workspace, TypeScript (with separate configs for main/renderer), Tailwind CSS, and ESLint. Create the standard electron-vite project structure (main, preload, renderer directories) with Colony's spec layer on top.

Deliverables: Working project creation with electron-vite template, `colony dev` running electron-vite under the hood.

**Milestone 1.2: Specification System**

Implement `createApp()`, `entity()`, `command()`, `query()`, and `window()` functions. Build the application registry that collects definitions at import time.

Deliverables: Working specification system with type inference.

**Milestone 1.3: Database Layer**

Integrate Drizzle ORM with Bun SQLite. Implement migration system. Integrate `drizzle-zod` for automatic schema generation.

Deliverables: Working database with migrations.

**Milestone 1.4: IPC Bridge**

Implement main process IPC handlers. Implement preload script with context isolation. Create renderer-side IPC client.

Deliverables: Type-safe IPC communication.

### Phase 2: Interface Generation

**Milestone 2.1: React Hooks**

Implement `useQuery` and `useCommand` hooks with TanStack Query integration. Implement cache invalidation based on entities.

Deliverables: Type-safe React hooks for commands and queries.

**Milestone 2.2: CLI Generation**

Implement CLI generator using Commander.js. Support all commands and queries with `--json` flag.

Deliverables: Working CLI with structured output.

**Milestone 2.3: Framework Components**

Implement CommandPalette, TitleBar, StatusBar, and dialog components. Create consistent styling with Tailwind.

Deliverables: Framework component library.

### Phase 3: System Integration

**Milestone 3.1: Menu and Tray**

Implement menu bar configuration. Implement system tray support. Wire menu items to commands.

Deliverables: Native menu and tray integration.

**Milestone 3.2: Auto-Updates**

Integrate electron-updater. Implement update configuration. Add update notifications.

Deliverables: Automatic update system.

**Milestone 3.3: Distribution**

Configure Electron Builder. Implement code signing. Create GitHub Actions workflow for releases.

Deliverables: One-command distribution builds.

### Phase 4: Optional Interfaces

**Milestone 4.1: MCP Server**

Implement optional FastMCP server generation. Map commands to MCP tools.

Deliverables: MCP server for AI agent integration.

**Milestone 4.2: REST API**

Implement optional Hono server generation. Map commands to POST endpoints, queries to GET.

Deliverables: REST API for web integration.

### Phase 5: Documentation and Examples

**Milestone 5.1: Documentation**

Write getting started guide. Document all APIs. Create tutorial application.

Deliverables: Comprehensive documentation.

**Milestone 5.2: Example Applications**

Build reference implementations: task manager, note-taking app, API client.

Deliverables: Example projects demonstrating capabilities.

---

## Open Questions

**Monorepo structure**: Should Colony be a single package or multiple (`@colony/core`, `@colony/react`, `@colony/cli`)? Multiple packages allow tree-shaking but add complexity.

**Preload script generation**: How much of the preload script should be generated vs. user-configurable? Security requires careful control here.

**State management**: Should Zustand integration be built-in, optional, or left to the user? TanStack Query handles server state; client state is less defined.

**Styling**: Should Tailwind be required, default, or optional? It's excellent for AI-assisted development but some users may prefer other approaches.

**Window management**: How should multi-window applications work? Current design shows single windows; complex apps may need more sophisticated patterns.

---

## Conclusion

Colony provides the "Rails for Electron" that the desktop development ecosystem lacks. By applying CDS principles to TypeScript and Electron, it enables developers to build complete, professional desktop applications with AI assistance in days rather than months.

The framework handles the accidental complexity of Electron development—IPC, security, packaging, updates—so developers can focus on essential complexity: what their application actually does. The unified specification layer ensures consistency across GUI, CLI, and API interfaces while enabling AI agents to understand and interact with Colony applications.

Colony is infrastructure for the hyper-personal software era. It empowers individuals to build sophisticated desktop applications tailored to their specific needs, leveraging the same technologies used by the major AI companies for their flagship products.

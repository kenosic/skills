# MARS SDK API Reference (App-Type)

Complete TypeScript SDK API surface for MARS v1.0 **app-type** mini-apps.
Read this when generating backend code (`main.ts`) or when the user asks about specific SDK types/interfaces.
For game-type SDK patterns, see the `mars-cocos-game-generator` or `mars-unity-game-generator` skills.

## Table of Contents
1. [Package Import](#1-package-import)
2. [Core Types](#2-core-types)
3. [MarsContext — Capability Injection](#3-marscontext--capability-injection)
4. [serve() — Backend Entry Point](#4-serve--backend-entry-point)
5. [MarsDB — Database](#5-marsdb--database)
6. [MarsStorage — Blob Storage](#6-marsstorage--blob-storage)
7. [MarsAuth — Authentication](#7-marsauth--authentication)
8. [MarsHTTP — Outbound HTTP](#8-marshttp--outbound-http)
9. [Media Capabilities](#9-media-capabilities)
10. [Connectivity Capabilities](#10-connectivity-capabilities)
11. [System Capabilities](#11-system-capabilities)
12. [Frontend Transport & Client Pattern](#12-frontend-transport--client-pattern)
13. [Complete Backend Example](#13-complete-backend-example)

---

## 1. Package Import

Compatibility checklist (must satisfy all):
- Backend code imports only from `@mars/sdk`
- TypeScript frontend client/transport imports only from `@mars/sdk`
- Only documented package entries are allowed: `@mars/sdk` and `@mars/sdk/game`
- For app-type generation, prefer `@mars/sdk` only unless the user explicitly asks for game frontend code

```typescript
// Backend imports
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";

// Frontend — TypeScript apps can use MarsClient with a transport
import { MarsClient, HttpTransport } from "@mars/sdk";
// or WsTransport, PostMessageTransport

// Wire-protocol naming helpers (for cross-language boundary mapping)
import { camelToSnake, snakeToCamel, mapKeys } from "@mars/sdk";
```

---

## 2. Core Types

```typescript
type MarsID = string;

type MarsErrorCode =
    | "INVALID_PARAMS"
    | "UNAUTHORIZED"
    | "PERMISSION_DENIED"
    | "NOT_FOUND"
    | "TIMEOUT"
    | "RESOURCE_EXHAUSTED"
    | "INTERNAL_ERROR";

interface MarsError {
    code: MarsErrorCode;
    message: string;
    details: Record<string, unknown>;
}

// Success: { data: T, error: null }
// Failure: { data: null, error: MarsError }
type MarsResponse<T = unknown> =
    | { data: T; error: null }
    | { data: null; error: MarsError };

interface MarsRPCRequest {
    requestId: string;    // UUID, must be unique
    method: string;       // e.g. "todo.create"
    params?: unknown;     // arbitrary payload
}
```

---

## 3. MarsContext — Capability Injection

`MarsContext` is the single entry point for all platform capabilities. Every RPC handler receives it.

```typescript
interface MarsContext {
    readonly requestId: string;
    /** `true` when the caller is the app author accessing via the admin interface (`/a/:appId/__admin/`). Use this to protect admin-only RPC methods instead of re-verifying identity. */
    readonly isAdmin: boolean;

    // Core capabilities (always available)
    readonly db: MarsDB;
    readonly storage: MarsStorage;
    readonly auth: MarsAuth;
    readonly env: Record<string, string>;

    // Optional capabilities (require declaration + permission)
    readonly http?: MarsHTTP;
    readonly media?: MarsMedia;
    readonly bluetooth?: MarsBluetooth;
    readonly nfc?: MarsNFC;
    readonly clipboard?: MarsClipboard;
    readonly filesystem?: MarsFilesystem;
    readonly calendar?: MarsCalendar;
}
```

Rules:
- Context is the ONLY way to access platform capabilities
- Optional fields are `undefined` when not declared in capabilities or not granted
- `env` includes runtime environment variables
- `MarsContext` is an alias for `MarsAppContext` (which extends `MarsBaseContext`)

---

## 4. serve() — Backend Entry Point

```typescript
interface MarsHandler {
    rpc?: (method: string, params: unknown, ctx: MarsContext) => Promise<unknown>;
    fetch?: (req: Request, ctx: MarsContext) => Promise<Response>;
}

function serve(handler: MarsHandler): void;
```

`serve()` is the only valid way to register a backend handler. The runtime:
1. Constructs `MarsContext` before calling the handler
2. Catches exceptions and converts them to `MarsError` structure
3. Returns the result as an RPC response

Standard backend skeleton:

```typescript
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";

serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        switch (method) {
            case "entity.action":
                return await handleAction(params, ctx);
            default:
                throw new MarsException({
                    code: "NOT_FOUND",
                    message: `Unknown method: ${method}`,
                    details: { method },
                });
        }
    },
});
```

### exceptionToMarsError

Utility function exported by the SDK for runtime implementors to convert arbitrary exceptions to `MarsError`:

```typescript
function exceptionToMarsError(err: unknown): MarsError;
// MarsException → { code, message, details }
// Other Error → { code: "INTERNAL_ERROR", message: err.message, details: {} }
```

---

## 5. MarsDB — Database

```typescript
interface MarsDB {
    /** SQL query returning rows */
    query<T = unknown>(sql: string, params?: unknown[]): Promise<T[]>;

    /** SQL execute (INSERT/UPDATE/DELETE), no rows returned */
    execute(sql: string, params?: unknown[]): Promise<void>;

    /** Convenience: read rows from a table with optional filter */
    read<T = unknown>(table: string, filter?: Record<string, unknown>): Promise<T[]>;

    /** Convenience: insert a row into a table */
    write(table: string, payload: Record<string, unknown>): Promise<void>;
}
```

**Critical**: Must call `CREATE TABLE IF NOT EXISTS` before any `db.read()` / `db.write()` / `db.query()` call.
The devTool sandbox enforces table existence — calling read/write on a non-existent table throws `TABLE_NOT_FOUND`.
Always define an `ensureSchema()` helper and call it at the top of `rpc()`.

**⚠️ API Distinction** — `db.read()`/`db.write()` vs `db.query()`/`db.execute()`:
- `db.query(sql, params)` — takes a **SQL string** + bind params array → for `SELECT` queries
- `db.execute(sql, params)` — takes a **SQL string** + bind params array → for `INSERT`/`UPDATE`/`DELETE`/DDL
- `db.read(table, filter)` — takes a **table name string** + filter object → convenience read
- `db.write(table, payload)` — takes a **table name string** + row object → convenience insert

Passing SQL to `db.read()` (e.g., `db.read("SELECT ... FROM todos", [])`) is a **common bug** —
the runtime interprets `"SELECT"` as the table name and throws `TABLE_NOT_FOUND`.

Usage patterns:
```typescript
// ⚠️ MUST create table before any read/write — otherwise TABLE_NOT_FOUND
await ctx.db.execute(
    `CREATE TABLE IF NOT EXISTS todos (
        id TEXT PRIMARY KEY,
        text TEXT NOT NULL,
        userId TEXT NOT NULL,
        done INTEGER NOT NULL DEFAULT 0
    )`,
    [],
);

// Insert
await ctx.db.write("todos", { id, text, userId: user.id, done: false });

// Read with filter
const todos = await ctx.db.read<TodoItem>("todos", { userId: user.id });

// SQL query
const results = await ctx.db.query("SELECT * FROM orders WHERE status = ?", ["pending"]);

// SQL execute
await ctx.db.execute("UPDATE todos SET done = true WHERE id = ? AND userId = ?", [id, userId]);
```

---

## 6. MarsStorage — Blob Storage

```typescript
interface MarsStorage {
    put(key: string, value: Blob | Uint8Array): Promise<void>;
    get(key: string): Promise<Blob | null>;
    delete(key: string): Promise<void>;
    getSignedUrl(key: string, expiresInSeconds: number): Promise<string>;
}
```

---

## 7. MarsAuth — Authentication

```typescript
interface MarsUser {
    id: MarsID;
    roles: string[];
    nickname?: string;   // requires user.profile.read
    avatar?: string;     // requires user.profile.read
    email?: string;      // requires user.email.read (sensitive)
    phone?: string;      // requires user.phone.read (sensitive)
}

interface MarsAuth {
    /** Get current user, or null if not authenticated */
    getUser(): Promise<MarsUser | null>;

    /** Get current user, throws UNAUTHORIZED if not authenticated */
    requireUser(): Promise<MarsUser>;
}
```

Pattern: Always call `requireUser()` before data operations:
```typescript
const user = await ctx.auth.requireUser();
// now safe to read/write user-scoped data
```

---

## 8. MarsHTTP — Outbound HTTP

```typescript
interface MarsHTTP {
    fetch(input: string | URL, init?: RequestInit): Promise<Response>;
}
```

- Follows Web Fetch API semantics
- Runtime checks `network.*.connect` permission and `allowHosts` constraint
- Unauthorized requests return `PERMISSION_DENIED`

---

## 9. Media Capabilities

```typescript
interface MarsMedia {
    chooseImage(options?: ChooseImageOptions): Promise<ImageFile[]>;
    saveImageToAlbum(tempFilePath: string): Promise<void>;
}

interface ChooseImageOptions {
    /** Max selectable images, default 9 */
    maxCount?: number;
    /** Image source */
    sourceType?: ("album" | "camera")[];
}

interface ImageFile {
    /** Temporary file path */
    tempFilePath: string;
    /** File size in bytes */
    size: number;
    /** MIME type */
    mimeType: string;
}
```

---

## 10. Connectivity Capabilities

```typescript
interface BluetoothDevice {
    deviceId: string;
    name: string;
    rssi: number;
}

interface BluetoothConnectOptions {
    deviceId: string;
    serviceUUID: string;
}

interface MarsBluetooth {
    /** Scan nearby BLE devices, returns stop-scan function */
    startScan(callback: (device: BluetoothDevice) => void): () => void;
    /** Connect to a device; serviceUUID must be in constraints.serviceUUIDs */
    connect(options: BluetoothConnectOptions): Promise<void>;
    disconnect(deviceId: string): Promise<void>;
    write(deviceId: string, data: ArrayBuffer): Promise<void>;
    subscribe(deviceId: string, callback: (data: ArrayBuffer) => void): () => void;
}

interface NFCTagInfo {
    tagId: string;
    techType: "ndef" | "mifare" | "isodep" | "unknown";
}

interface NDEFRecord {
    type: string;
    payload: Uint8Array;
}

interface MarsNFC {
    readTag(): Promise<NFCTagInfo>;
    readNDEF(): Promise<NDEFRecord[]>;
}
```

---

## 11. System Capabilities

```typescript
interface MarsClipboard {
    /** Read clipboard text; MUST show a prompt to the user */
    getText(): Promise<string | null>;
    /** Write text to clipboard */
    setText(text: string): Promise<void>;
}

interface FileInfo {
    path: string;
    size: number;
    lastModified: string;
    isDirectory: boolean;
}

interface MarsFilesystem {
    /** Read file from user's public directory */
    readFile(path: string): Promise<ArrayBuffer>;
    /** Write file to user's public directory */
    writeFile(path: string, data: ArrayBuffer | Uint8Array): Promise<void>;
    /** Get file/directory info */
    getFileInfo(path: string): Promise<FileInfo>;
    /** List contents of a directory */
    listDir(path: string): Promise<FileInfo[]>;
}

interface CalendarEvent {
    id?: string;
    title: string;
    startTime: string;    // ISO8601
    endTime: string;      // ISO8601
    location?: string;
    notes?: string;
    allDay?: boolean;
}

interface GetEventsOptions {
    startTime: string;    // ISO8601
    endTime: string;      // ISO8601
}

interface MarsCalendar {
    getEvents(options: GetEventsOptions): Promise<CalendarEvent[]>;
    /** Returns new event ID */
    addEvent(event: CalendarEvent): Promise<string>;
    removeEvent(eventId: string): Promise<void>;
}
```

---

## 12. Frontend Transport & Client Pattern

The SDK provides multiple transport layers and a `MarsClient` for structured RPC calls.

### Transport Selection

| Transport | Use Case |
|-----------|----------|
| `HttpTransport` | Standard request-response (recommended default) |
| `WsTransport` | Full-duplex: real-time sync |
| `PostMessageTransport` | WebView-embedded apps communicating with host page |

### Using MarsClient (recommended for TypeScript apps)

```typescript
import { MarsClient, HttpTransport } from "@mars/sdk";

const transport = new HttpTransport("https://api.example.com", {
    rpcPath: "/rpc",
    credentials: "include",
});
const client = new MarsClient(transport);

// RPC call
const result = await client.invoke<{ id: string }>("todo.create", { text: "Buy milk" });

// Event subscription (WsTransport/PostMessageTransport only)
const unsub = client.subscribe("order.updated", (data) => {
    console.log(data);
});
```

### Using window.__marsTransport (inline HTML apps)

For single-file `index.html` frontends, the runtime injects `window.__marsTransport`:

```javascript
const transport = window.__marsTransport;

async function invoke(method, params) {
    const requestId = crypto.randomUUID();
    const res = await transport.send({ requestId, method, params });
    if (res.error) throw new Error(res.error.message);
    return res.data;
}

// Usage
const { items } = await invoke("todo.list");
const { id } = await invoke("todo.create", { text: "new task" });
```

### Transport Interface

```typescript
interface Transport {
    send<T = unknown>(req: MarsRPCRequest): Promise<MarsRPCResult<T>>;
    /** Subscribe to server-push events (optional, full-duplex transports only) */
    onEvent?(callback: (event: MarsEvent) => void): () => void;
}

interface MarsEvent {
    event: string;
    data?: unknown;
}
```

---

## 13. Complete Backend Example

A full backend for a simple notes app:

```typescript
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";

interface Note {
    id: string;
    title: string;
    content: string;
    userId: string;
    createdAt: string;
    updatedAt: string;
}

// 建表：必须在使用 db.read/db.write 之前执行 CREATE TABLE
async function ensureSchema(ctx: MarsContext) {
    await ctx.db.execute(
        `CREATE TABLE IF NOT EXISTS notes (
            id TEXT PRIMARY KEY,
            title TEXT NOT NULL,
            content TEXT NOT NULL DEFAULT '',
            userId TEXT NOT NULL,
            createdAt TEXT NOT NULL,
            updatedAt TEXT NOT NULL
        )`,
        [],
    );
}

serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        await ensureSchema(ctx);
        switch (method) {
            case "note.create":
                return await createNote(params as { title: string; content: string }, ctx);
            case "note.list":
                return await listNotes(ctx);
            case "note.update":
                return await updateNote(params as { id: string; title?: string; content?: string }, ctx);
            case "note.delete":
                return await deleteNote(params as { id: string }, ctx);
            default:
                throw new MarsException({
                    code: "NOT_FOUND",
                    message: `Unknown method: ${method}`,
                    details: { method },
                });
        }
    },
});

async function createNote(params: { title: string; content: string }, ctx: MarsContext) {
    const user = await ctx.auth.requireUser();

    if (!params.title?.trim()) {
        throw new MarsException({
            code: "INVALID_PARAMS",
            message: "title is required",
            details: { field: "title" },
        });
    }

    const id = crypto.randomUUID();
    const now = new Date().toISOString();

    await ctx.db.write("notes", {
        id,
        title: params.title.trim(),
        content: params.content ?? "",
        userId: user.id,
        createdAt: now,
        updatedAt: now,
    });

    return { id };
}

async function listNotes(ctx: MarsContext) {
    const user = await ctx.auth.requireUser();
    const notes = await ctx.db.read<Note>("notes", { userId: user.id });
    return { items: notes };
}

async function updateNote(params: { id: string; title?: string; content?: string }, ctx: MarsContext) {
    const user = await ctx.auth.requireUser();
    const [note] = await ctx.db.read<Note>("notes", { id: params.id, userId: user.id });

    if (!note) {
        throw new MarsException({
            code: "NOT_FOUND",
            message: "Note not found",
            details: { id: params.id },
        });
    }

    const updates: string[] = [];
    const values: unknown[] = [];

    if (params.title !== undefined) {
        updates.push("title = ?");
        values.push(params.title.trim());
    }
    if (params.content !== undefined) {
        updates.push("content = ?");
        values.push(params.content);
    }
    updates.push("updatedAt = ?");
    values.push(new Date().toISOString());

    values.push(params.id, user.id);

    await ctx.db.execute(
        `UPDATE notes SET ${updates.join(", ")} WHERE id = ? AND userId = ?`,
        values,
    );

    return { id: params.id, updated: true };
}

async function deleteNote(params: { id: string }, ctx: MarsContext) {
    const user = await ctx.auth.requireUser();

    await ctx.db.execute(
        "DELETE FROM notes WHERE id = ? AND userId = ?",
        [params.id, user.id],
    );

    return { id: params.id, deleted: true };
}
```

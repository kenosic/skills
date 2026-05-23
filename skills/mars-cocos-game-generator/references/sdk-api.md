# MARS SDK API Reference (Cocos Creator Game-Type)

SDK API surface for MARS v1.0 **Cocos Creator game-type** mini-apps.
For app-type SDK details, see the `mars-app-generator` skill.

## Table of Contents
1. [Package Import](#1-package-import)
2. [Core Types](#2-core-types)
3. [MarsGameClient — Game Frontend Client](#3-marsgameclient--game-frontend-client)
4. [NativeBridgeTransport](#4-nativebridgetransport)
5. [MarsGameFrontendContext](#5-marsgamefrontendcontext)
6. [serve() — Backend Entry Point](#6-serve--backend-entry-point)
7. [MarsDB — Database](#7-marsdb--database)
8. [MarsStorage — Blob Storage](#8-marsstorage--blob-storage)
9. [MarsAuth — Authentication](#9-marsauth--authentication)
10. [Complete Game Frontend Example](#10-complete-game-frontend-example)
11. [Complete Game Backend Example](#11-complete-game-backend-example)

---

## 1. Package Import

Compatibility checklist (must satisfy all):
- Cocos game frontend imports `MarsGameClient` from `@mars/sdk/game`
- Game backend imports `serve`, `MarsException`, and `MarsContext` from `@mars/sdk`
- Only documented package entries are allowed: `@mars/sdk` and `@mars/sdk/game`
- Keep frontend and backend imports separated by runtime role (frontend vs backend)

```typescript
// Game frontend — use the dedicated game sub-path
import { MarsGameClient } from "@mars/sdk/game";

// Backend imports (same as app-type)
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";

// Transport (for advanced use)
import { NativeBridgeTransport } from "@mars/sdk";
import type { MarsNativeBridge } from "@mars/sdk";
```

> **Important**: Game frontends import from `@mars/sdk/game`, NOT from `@mars/sdk`.

---

## 2. Core Types

```typescript
type MarsID = string;

type AppType = "app" | "game";
type GameEngine = "cocos" | "unity";

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

type MarsResponse<T = unknown> =
    | { data: T; error: null }
    | { data: null; error: MarsError };

interface MarsRPCRequest {
    requestId: string;
    method: string;
    params?: unknown;
}
```

---

## 3. MarsGameClient — Game Frontend Client

The primary API for Cocos Creator game frontends to access MARS platform services.

```typescript
class MarsGameClient {
    /** The game frontend context (auth, storage, db, env) */
    readonly context: MarsGameFrontendContext;

    /**
     * Factory method — auto-discovers the native bridge from globalThis.__marsBridge
     * and context from globalThis.__marsGameContext (injected by RN host before game starts).
     * Throws synchronously if context is not available.
     */
    static create(): MarsGameClient;

    /** Call a backend RPC method */
    invoke<T = unknown>(method: string, params?: unknown): Promise<T>;

    /** Subscribe to real-time server events. Returns unsubscribe function */
    subscribe(event: string, callback: (data: unknown) => void): () => void;
}
```

### Usage

```typescript
import { MarsGameClient } from "@mars/sdk/game";

const mars = MarsGameClient.create();  // synchronous — host injects context before game starts

// RPC call to backend
const result = await mars.invoke<{ entries: any[] }>("leaderboard.top", { limit: 10 });

// Event subscription
const unsub = mars.subscribe("match.update", (data) => {
    console.log("Match update:", data);
});

// Context access
const user = await mars.context.auth.requireUser();
const env = mars.context.env;
```

---

## 4. NativeBridgeTransport

Low-level transport for game engine ↔ React Native host communication.
`MarsGameClient` uses this internally — you rarely need to use it directly.

```typescript
interface MarsNativeBridge {
    postMessage(data: string): void;
    onMessage(callback: (data: string) => void): void;
    offMessage(callback: (data: string) => void): void;
}

class NativeBridgeTransport implements Transport {
    /** Auto-discovers globalThis.__marsBridge, or accepts an explicit bridge */
    constructor(bridge?: MarsNativeBridge);

    send<T = unknown>(req: MarsRPCRequest): Promise<MarsRPCResult<T>>;
    onEvent?(callback: (event: MarsEvent) => void): () => void;
    dispose(): void;
}
```

The RN host injects `globalThis.__marsBridge` before the Cocos Creator game starts.

---

## 5. MarsGameFrontendContext

Restricted context for game engine frontends — platform services only.

```typescript
interface MarsGameFrontendContext {
    readonly auth: MarsAuth;
    readonly storage: MarsStorage;
    readonly db: MarsDB;
    readonly env: Record<string, string>;
}
```

**No rendering/audio/sensor** — Cocos Creator manages these directly through its native APIs.

---

## 6. serve() — Backend Entry Point

Same as app-type — game backends use the standard `serve()` pattern:

```typescript
interface MarsHandler {
    rpc?: (method: string, params: unknown, ctx: MarsContext) => Promise<unknown>;
    fetch?: (req: Request, ctx: MarsContext) => Promise<Response>;
}

function serve(handler: MarsHandler): void;
```

Backend `MarsContext` provides: `requestId`, `isAdmin`, `db`, `storage`, `auth`, `env`, optional `http`.

> **Note**: `ctx.isAdmin` is `true` when the caller is the app author accessing via the admin interface (`/a/:appId/__admin/`). Use it to protect admin-only RPC methods.
>
> **Note**: `MarsContext` is the public export alias (`= MarsAppContext`). For game backends the runtime
> injects only the base platform services (`MarsGameHandlerContext = MarsBaseContext`) — the optional
> fields `media`, `bluetooth`, `nfc`, etc. are **always `undefined`** in game handlers.
> You may alternatively import `import type { MarsBaseContext } from "@mars/sdk"` as a more precise type.

---

## 7. MarsDB — Database

```typescript
interface MarsDB {
    query<T = unknown>(sql: string, params?: unknown[]): Promise<T[]>;
    execute(sql: string, params?: unknown[]): Promise<void>;
    read<T = unknown>(table: string, filter?: Record<string, unknown>): Promise<T[]>;
    write(table: string, payload: Record<string, unknown>): Promise<void>;
}
```

**Critical**: Must call `CREATE TABLE IF NOT EXISTS` before any read/write.

**⚠️ API Distinction** — `db.read()`/`db.write()` vs `db.query()`/`db.execute()`:
- `db.query(sql, params)` — takes a **SQL string** + bind params array → for `SELECT` queries
- `db.execute(sql, params)` — takes a **SQL string** + bind params array → for `INSERT`/`UPDATE`/`DELETE`/DDL
- `db.read(table, filter)` — takes a **table name string** + filter object → convenience read
- `db.write(table, payload)` — takes a **table name string** + row object → convenience insert

Passing SQL to `db.read()` (e.g., `db.read("SELECT ... FROM scores", [])`) is a **common bug** —
the runtime interprets `"SELECT"` as the table name and throws `TABLE_NOT_FOUND`.

---

## 8. MarsStorage — Blob Storage

```typescript
interface MarsStorage {
    put(key: string, value: Blob | Uint8Array): Promise<void>;
    get(key: string): Promise<Blob | null>;
    delete(key: string): Promise<void>;
    getSignedUrl(key: string, expiresInSeconds: number): Promise<string>;
}
```

---

## 9. MarsAuth — Authentication

```typescript
interface MarsUser {
    id: MarsID;
    roles: string[];
    nickname?: string;
    avatar?: string;
    email?: string;
    phone?: string;
}

interface MarsAuth {
    getUser(): Promise<MarsUser | null>;
    requireUser(): Promise<MarsUser>;
}
```

---

## 10. Complete Game Frontend Example

```typescript
import { MarsGameClient } from "@mars/sdk/game";

// === Initialize MARS platform services ===
const mars = MarsGameClient.create();  // synchronous — host injects context before game starts
const user = await mars.context.auth.requireUser();

// === Player initialization ===
console.log(`Player ${user.nickname ?? user.id} connected`);

// Load cloud save
const { data: saveData } = await mars.invoke<{ data: { level: number; score: number } | null }>("save.get");
if (saveData) {
    console.log(`Resuming from level ${saveData.level}`);
}

// === Game loop integration ===
// Call these from Cocos Creator game logic:

async function onGameOver(score: number) {
    // Submit score to leaderboard
    await mars.invoke("score.submit", { score });

    // Save progress
    await mars.invoke("save.put", { data: { level: currentLevel, score } });
}

async function showLeaderboard() {
    const { entries } = await mars.invoke<{
        entries: { userId: string; bestScore: number }[];
    }>("leaderboard.top", { limit: 20 });
    // Render in Cocos Creator UI
    return entries;
}

// === Real-time multiplayer (if needed) ===
const unsub = mars.subscribe("match.playerJoined", (data) => {
    console.log("New player joined:", data);
});
```

---

## 11. Complete Game Backend Example

```typescript
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";

async function ensureSchema(ctx: MarsContext) {
    await ctx.db.execute(
        `CREATE TABLE IF NOT EXISTS scores (
            id TEXT PRIMARY KEY,
            userId TEXT NOT NULL,
            score INTEGER NOT NULL,
            createdAt TEXT NOT NULL
        )`,
        [],
    );
}

serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        await ensureSchema(ctx);
        switch (method) {
            case "score.submit":
                return await submitScore(params as { score: number }, ctx);
            case "leaderboard.top":
                return await getLeaderboard(params as { limit?: number }, ctx);
            case "save.put":
                return await putSave(params as { data: unknown }, ctx);
            case "save.get":
                return await getSave(ctx);
            default:
                throw new MarsException({
                    code: "NOT_FOUND",
                    message: `Unknown method: ${method}`,
                    details: { method },
                });
        }
    },
});

async function submitScore(params: { score: number }, ctx: MarsContext) {
    const user = await ctx.auth.requireUser();
    if (typeof params.score !== "number" || params.score < 0) {
        throw new MarsException({
            code: "INVALID_PARAMS",
            message: "score must be a non-negative number",
            details: { field: "score" },
        });
    }
    const id = crypto.randomUUID();
    await ctx.db.write("scores", {
        id,
        userId: user.id,
        score: params.score,
        createdAt: new Date().toISOString(),
    });
    return { id };
}

async function getLeaderboard(params: { limit?: number }, ctx: MarsContext) {
    const limit = params.limit ?? 10;
    const entries = await ctx.db.query(
        "SELECT userId, MAX(score) as bestScore FROM scores GROUP BY userId ORDER BY bestScore DESC LIMIT ?",
        [limit],
    );
    return { entries };
}

async function putSave(params: { data: unknown }, ctx: MarsContext) {
    const user = await ctx.auth.requireUser();
    await ctx.db.execute(
        `INSERT OR REPLACE INTO saves (userId, data, updatedAt) VALUES (?, ?, ?)`,
        [user.id, JSON.stringify(params.data), new Date().toISOString()],
    );
    return { saved: true };
}

async function getSave(ctx: MarsContext) {
    const user = await ctx.auth.requireUser();
    const [row] = await ctx.db.read<{ data: string }>("saves", { userId: user.id });
    return { data: row ? JSON.parse(row.data) : null };
}
```

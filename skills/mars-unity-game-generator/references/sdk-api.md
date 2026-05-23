# MARS SDK API — Unity Game Frontend & Backend

This reference covers the SDK API for Unity game-type mini-apps:
- **Game Frontend**: `MarsGameClient` from `@mars/sdk/game`
- **Game Backend**: `serve()` from `@mars/sdk`
- **Transport**: `NativeBridgeTransport` (auto-discovered)

Compatibility checklist (must satisfy all):
- Unity game frontend imports from `@mars/sdk/game`
- Game backend imports from `@mars/sdk`
- Only documented package entries are allowed: `@mars/sdk` and `@mars/sdk/game`
- Keep frontend and backend imports aligned with their runtime role

## 1. MarsGameClient

The primary client for Unity game frontends. Imported from `@mars/sdk/game`.

```typescript
import { MarsGameClient } from "@mars/sdk/game";
```

### Factory

```typescript
const mars = MarsGameClient.create();
```

- Auto-discovers `globalThis.__marsBridge` injected by RN host before game starts
- Throws synchronously if `globalThis.__marsGameContext` is not available
- Returns a ready-to-use client with `context`, `invoke()`, and `subscribe()`

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `mars.context` | `MarsGameFrontendContext` | Platform services (auth, storage, db, env) |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `invoke` | `invoke<T>(method: string, params?: unknown): Promise<T>` | Call backend RPC method |
| `subscribe` | `subscribe(event: string, cb: (data: unknown) => void): () => void` | Subscribe to real-time event; returns unsubscribe fn |

## 2. MarsGameFrontendContext

The context available to game frontends. Subset of full MarsContext — only platform services.

```typescript
interface MarsGameFrontendContext {
    auth: AuthAPI;      // Player identity
    storage: StorageAPI; // Object storage
    db: DatabaseAPI;     // Database
    env: EnvAPI;         // Environment variables
}
```

**Not available in game frontends** (Unity handles natively):
- `media`, `bluetooth`, `nfc`, `clipboard`, `filesystem`, `calendar`
- `sensor`, `haptics`, `gpu`, `audio`

### auth

```typescript
// Get current user (returns null if not logged in)
const user = await mars.context.auth.getUser();

// Require authentication (throws if not logged in)
const user = await mars.context.auth.requireUser();
// user: { id: string; name: string; avatar?: string }
```

### storage

```typescript
// Write a file
await mars.context.storage.put("saves/slot1.json", data);

// Read a file
const data = await mars.context.storage.get("saves/slot1.json");

// Delete a file
await mars.context.storage.delete("saves/slot1.json");

// List files
const files = await mars.context.storage.list("saves/");
```

### db

```typescript
// Query
const rows = await mars.context.db.query("SELECT * FROM scores ORDER BY score DESC LIMIT 10", []);

// Read by condition
const entries = await mars.context.db.read<ScoreEntry>("scores", { userId: "abc" });

// Write (insert)
await mars.context.db.write("scores", { id: "uuid", userId: "abc", score: 100, createdAt: "..." });

// Execute arbitrary SQL
await mars.context.db.execute("CREATE TABLE IF NOT EXISTS scores (id TEXT PRIMARY KEY, ...)", []);
```

### env

```typescript
const cdnBase = mars.context.env.MARS_CDN_BASE_URL;
const assetUrl = `${cdnBase}/assets/levels.assetbundle`;
```

## 3. NativeBridgeTransport

The transport layer for game engines communicating with the RN host. Auto-discovered by
`MarsGameClient.create()` — you rarely need to use it directly.

```typescript
import { NativeBridgeTransport } from "@mars/sdk";
```

### How It Works

1. RN host injects `globalThis.__marsBridge` with `{ postMessage(msg: string): void; onMessage(cb: (msg: string) => void): void }`
2. `MarsGameClient.create()` discovers this bridge and creates a `NativeBridgeTransport`
3. Messages are JSON-RPC 2.0 strings over JSI/MessageChannel

### Manual Usage (advanced)

```typescript
import { NativeBridgeTransport } from "@mars/sdk";

const bridge = globalThis.__marsBridge;
const transport = new NativeBridgeTransport(bridge);
await transport.connect();
transport.send({ jsonrpc: "2.0", method: "score.submit", params: { score: 100 }, id: 1 });
transport.onMessage((msg) => console.log(msg));
```

### MarsNativeBridge Interface

```typescript
interface MarsNativeBridge {
    postMessage(message: string): void;
    onMessage(callback: (message: string) => void): void;
}
```

## 4. Backend API — serve()

Game backends use the same `serve()` as app-type, but handle game-specific RPC.

```typescript
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";
```

### serve()

```typescript
serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        switch (method) {
            case "score.submit":
                // handle...
            default:
                throw new MarsException({ code: "NOT_FOUND", message: `Unknown: ${method}` });
        }
    },
});
```

### MarsContext (Backend)

Context injected into game backend handlers. For game backends, only platform services are available:

```typescript
interface MarsContext {
    db: DatabaseAPI;
    storage: StorageAPI;
    auth: AuthAPI;
    env: Record<string, string>;
    http?: HttpAPI;  // optional, requires 'http' capability
    // Note: media, bluetooth, nfc, clipboard, etc. are ALWAYS undefined in game backends
}
```

> `MarsContext` is the public alias for `MarsAppContext`; for game handlers the runtime only injects
> the base platform services. Use `db`, `storage`, `auth`, `env`, and `http` — all other fields are `undefined`.
>
> **`ctx.isAdmin`** is `true` when the caller is the app author accessing via the admin interface (`/a/:appId/__admin/`). Use it to protect admin-only RPC methods.

### MarsException

```typescript
throw new MarsException({
    code: "INVALID_PARAMS",   // Error code
    message: "score must be a non-negative number",  // Human-readable
    details: { field: "score" },  // Optional extra data
});
```

Standard codes: `NOT_FOUND`, `INVALID_PARAMS`, `UNAUTHORIZED`, `PERMISSION_DENIED`, `INTERNAL_ERROR`

### Database API (Backend)

```typescript
// Execute DDL or DML
await ctx.db.execute(
    `CREATE TABLE IF NOT EXISTS scores (
        id TEXT PRIMARY KEY,
        userId TEXT NOT NULL,
        score INTEGER NOT NULL,
        createdAt TEXT NOT NULL
    )`,
    [],
);

// Insert/upsert
await ctx.db.write("scores", { id, userId, score, createdAt });

// Query with SQL
const rows = await ctx.db.query(
    "SELECT userId, MAX(score) as bestScore FROM scores GROUP BY userId ORDER BY bestScore DESC LIMIT ?",
    [10],
);

// Read by condition
const entries = await ctx.db.read<SaveEntry>("saves", { userId: user.id });
```

**Critical**: Must call `CREATE TABLE IF NOT EXISTS` before any `db.read()` / `db.write()` / `db.query()` call.

**⚠️ API Distinction** — `db.read()`/`db.write()` vs `db.query()`/`db.execute()`:
- `db.query(sql, params)` — takes a **SQL string** + bind params array → for `SELECT` queries
- `db.execute(sql, params)` — takes a **SQL string** + bind params array → for `INSERT`/`UPDATE`/`DELETE`/DDL
- `db.read(table, filter)` — takes a **table name string** + filter object → convenience read
- `db.write(table, payload)` — takes a **table name string** + row object → convenience insert

Passing SQL to `db.read()` (e.g., `db.read("SELECT ... FROM scores", [])`) is a **common bug** —
the runtime interprets `"SELECT"` as the table name and throws `TABLE_NOT_FOUND`.

### Auth API (Backend)

```typescript
const user = await ctx.auth.requireUser();
// user.id — unique player identifier
// user.name — display name
// user.avatar — avatar URL (optional)
```

## 5. Complete Frontend Example (game.ts)

```typescript
import { MarsGameClient } from "@mars/sdk/game";

const mars = await MarsGameClient.create();

// --- Player ---
const user = await mars.context.auth.requireUser();
console.log(`Player: ${user.name} (${user.id})`);

// --- Leaderboard ---
const { entries } = await mars.invoke<{
    entries: { userId: string; bestScore: number }[];
}>("leaderboard.top", { limit: 20 });

// --- Submit Score ---
await mars.invoke("score.submit", { score: 5000 });

// --- Cloud Saves ---
// Save
await mars.invoke("save.put", {
    data: { level: 3, towers: ["archer", "mage", "bomb"], gold: 1500 },
});

// Load
const { data } = await mars.invoke<{ data: unknown }>("save.get");

// --- CDN Assets ---
const cdnBase = mars.context.env.MARS_CDN_BASE_URL;
const levelsUrl = `${cdnBase}/assets/levels.assetbundle`;
// Load AssetBundle via Unity's native networking: UnityWebRequest.GetAssetBundle(levelsUrl)

// --- Real-time Events ---
const unsub = mars.subscribe("match.update", (event) => {
    console.log("Match event:", event);
});
// Later: unsub();
```

## 6. Complete Backend Example (backend/main.ts)

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
    await ctx.db.execute(
        `CREATE TABLE IF NOT EXISTS saves (
            userId TEXT PRIMARY KEY,
            data TEXT NOT NULL,
            updatedAt TEXT NOT NULL
        )`,
        [],
    );
}

serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        await ensureSchema(ctx);
        switch (method) {
            case "score.submit": {
                const user = await ctx.auth.requireUser();
                const { score } = params as { score: number };
                if (typeof score !== "number" || score < 0) {
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
                    score,
                    createdAt: new Date().toISOString(),
                });
                return { id };
            }
            case "leaderboard.top": {
                const { limit = 10 } = (params ?? {}) as { limit?: number };
                const entries = await ctx.db.query(
                    "SELECT userId, MAX(score) as bestScore FROM scores GROUP BY userId ORDER BY bestScore DESC LIMIT ?",
                    [limit],
                );
                return { entries };
            }
            case "save.put": {
                const user = await ctx.auth.requireUser();
                const { data } = params as { data: unknown };
                await ctx.db.execute(
                    `INSERT OR REPLACE INTO saves (userId, data, updatedAt) VALUES (?, ?, ?)`,
                    [user.id, JSON.stringify(data), new Date().toISOString()],
                );
                return { saved: true };
            }
            case "save.get": {
                const user = await ctx.auth.requireUser();
                const [row] = await ctx.db.read<{ data: string }>("saves", { userId: user.id });
                return { data: row ? JSON.parse(row.data) : null };
            }
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

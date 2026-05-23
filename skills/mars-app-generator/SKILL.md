---
name: mars-app-generator
description: >
    Generate MARS-compliant app-type mini-app (应用类微应用) packages from scratch. Use this skill whenever the
    user wants to create a new app-type mini-app — including todo apps, utilities, tools, e-commerce
    mini-programs, social features, or any other non-game mini-app. Also use when the user mentions
    "manifest.json", "permissions.json", "mars sdk", "serve()", "MarsContext", "MarsAppContext",
    "MarsHandler", "backend/main.ts", "@mars/sdk", "WebView", "PostMessageTransport", or asks about
    mini-app package structure for app-type applications. This skill covers app-type mini-apps with
    WebView frontend + TypeScript backend. For game-type mini-apps (Cocos Creator or Unity), use the
    mars-cocos-game-generator or mars-unity-game-generator skills instead.
    Trigger this skill proactively even if the user simply says "create an app" or "build a small application"
    within a MARS project context (unless they specifically mention games, Cocos, or Unity).
---

# MARS App-Type Mini-App Generator

Generate complete, protocol-compliant MARS app-type mini-app packages. This skill encodes the full MARS v1.0
protocol standard, SDK API surface, permission model, and packaging conventions so that every generated app
is ready for the MARS runtime without manual fixups.

This skill is for **app-type** mini-apps only (WebView frontend + backend). For game-type mini-apps, see:
- `mars-cocos-game-generator` — Cocos Creator games on Expo/React Native
- `mars-unity-game-generator` — Unity games on Expo/React Native

Read the reference files in this skill directory for detailed specifications:
- `references/protocol.md` — Package structure, manifest schema, RPC protocol
- `references/permissions.md` — Permission scopes, naming conventions, consent rules
- `references/sdk-api.md` — SDK types, context capabilities, handler patterns

## SDK Compatibility Guardrails

Always generate code using the current SDK export surface. Do not invent or guess SDK subpaths.

- Allowed backend imports:
    - `import { serve, MarsException } from "@mars/sdk"`
    - `import type { MarsContext } from "@mars/sdk"`
- Allowed frontend SDK usage (optional TS frontend):
    - `import { MarsClient, HttpTransport } from "@mars/sdk"`
    - `import { MarsClient, WsTransport } from "@mars/sdk"`
    - `import { MarsClient, PostMessageTransport } from "@mars/sdk"`
- Forbidden imports:
    - `@mars/sdk/common` (not an SDK export)
    - any unlisted subpath such as `@mars/sdk/*` except documented entries

If user code contains old imports:
1. Replace `@mars/sdk/common` with `@mars/sdk`.
2. Re-run with devTool import map enabled (`.devtool/deno.json` should map `@mars/sdk`).
3. Do NOT suggest `deno add jsr:@mars/sdk/common`.

## Package File Count Constraint (Hard Limit)

> **⚠️ MANDATORY**: The platform enforces a **maximum of 2,000 files** per ZIP package.
> Packages with more files are rejected with a clear error. Never generate apps that bundle
> large numbers of individual data files.

### Forbidden patterns

```
❌ backend/data/characters/一.json   ← individual JSON per record
❌ backend/data/characters/二.json
❌ backend/data/characters/三.json
   ... (hundreds/thousands more)
```

### Required patterns for bulk data

| Data volume | Correct approach |
|-------------|------------------|
| Seed data (records) | One merged JSON file → seed into `ctx.db` on first run |
| Static lookup table (small, < 1 MB) | Single merged JSON, loaded via `Deno.readTextFile` |
| Large lookup table (> 5 MB, read on demand) | Single merged JSON with **module-level cache** |
| Per-user runtime data | Always use `ctx.db` (SQLite via `db` capability) |

### Module-level cache pattern (for large merged JSON files)

When a backend needs to do on-demand lookup from a large data file (e.g., a 30 MB character
dictionary), use a module-level variable so the file is parsed **only once** per deployment
version (the sandbox caches loaded handlers):

```typescript
// ✅ CORRECT: module-level cache — file parsed once, reused on all subsequent RPC calls
let _dataCache: Record<string, unknown> | null = null;

async function getDataCache(): Promise<Record<string, unknown>> {
    if (_dataCache) return _dataCache;
    const baseDir = new URL(".", import.meta.url).pathname
        .replace(/^\/([A-Za-z]:)/, "$1");
    const raw = await Deno.readTextFile(baseDir + "data/my_data.json");
    _dataCache = JSON.parse(raw) as Record<string, unknown>;
    return _dataCache;
}

async function handleLookup(params: unknown, _ctx: MarsContext) {
    const cache = await getDataCache();
    const key = requireString(params, "key");
    return { data: cache[key] ?? null };
}
```

```
✅ backend/data/my_data.json          ← single merged file
✅ backend/data/entries.json          ← all records in one JSON array
✅ ctx.db queries for per-user data
```

## Database Schema Initialization (Hard Constraint)

> **⚠️ MANDATORY**: Every backend that uses `ctx.db` MUST call `CREATE TABLE IF NOT EXISTS` **before** any
> `db.read()`, `db.write()`, `db.query()`, or `db.execute()` (non-DDL) call. The devTool sandbox enforces
> table existence at runtime — calling read/write on a non-existent table throws `TABLE_NOT_FOUND`.

The required pattern:
1. Define an `ensureSchema(ctx)` async helper that executes all `CREATE TABLE IF NOT EXISTS` statements
2. Call `await ensureSchema(ctx)` as the **first line** inside the `rpc()` handler, before the `switch`
3. Table names must be valid identifiers (e.g., `todos`, `notes`, `orders`) — never SQL keywords

```typescript
async function ensureSchema(ctx: MarsContext) {
    await ctx.db.execute(
        `CREATE TABLE IF NOT EXISTS todos (
            id TEXT PRIMARY KEY,
            text TEXT NOT NULL,
            userId TEXT NOT NULL,
            done INTEGER NOT NULL DEFAULT 0,
            createdAt TEXT NOT NULL
        )`,
        [],
    );
}

serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        await ensureSchema(ctx);   // ← MUST be first
        switch (method) { /* ... */ }
    },
});
```

**Never** generate backend code that calls `db.read()` or `db.write()` without a preceding `ensureSchema()` call.

### db API Method Distinction (Common Source of Bugs)

| Method | First Parameter | Second Parameter | Use For |
|--------|----------------|-----------------|--------|
| `db.query(sql, params)` | SQL string | `unknown[]` bind params | `SELECT` queries with SQL |
| `db.execute(sql, params)` | SQL string | `unknown[]` bind params | `INSERT`/`UPDATE`/`DELETE`/`CREATE TABLE` |
| `db.read(table, filter)` | **table name** (not SQL!) | `Record<string, unknown>` filter | Simple reads by field match |
| `db.write(table, payload)` | **table name** (not SQL!) | `Record<string, unknown>` row data | Simple inserts |

> **⚠️ CRITICAL**: `db.read()` and `db.write()` take a **plain table name** (e.g., `"todos"`), NOT a SQL statement.
> Passing SQL like `db.read("SELECT ... FROM todos", [])` is **WRONG** — the runtime interprets `"SELECT"` as
> the table name and throws `TABLE_NOT_FOUND`. Use `db.query()` for SQL queries instead.

```typescript
// ✅ CORRECT: db.read takes table name + filter object
const todos = await ctx.db.read("todos", { userId: user.id });

// ✅ CORRECT: db.query takes SQL string + params array
const todos = await ctx.db.query(
    "SELECT * FROM todos WHERE userId = ? ORDER BY createdAt DESC LIMIT 50",
    [user.id],
);

// ❌ WRONG: passing SQL to db.read — will throw TABLE_NOT_FOUND on table "SELECT"
const todos = await ctx.db.read(
    "SELECT * FROM todos WHERE userId = ?",
    [user.id],
);
```

## When to Use

- User wants to **create a new app-type mini-app** (utility, tool, social, e-commerce, etc.)
- User wants to **scaffold** the directory structure for a MARS app
- User wants to **add a backend** or **frontend** to an existing MARS app
- User asks about **manifest.json** or **permissions.json** format for app-type
- User needs help with **@mars/sdk** usage patterns (serve, MarsContext, MarsClient)
- User asks about **PostMessageTransport**, **HttpTransport**, or **WsTransport**

> **Not for games!** If the user wants to build a game with Cocos Creator or Unity,
> use the `mars-cocos-game-generator` or `mars-unity-game-generator` skill instead.

## Package Structure

```
my-app/
├── manifest.json          # App metadata & capabilities
├── permissions.json       # Permission declarations
├── frontend/
│   └── index.html         # Frontend entry (single HTML, inline JS/CSS)
├── backend/
│   └── main.ts            # Backend entry (TypeScript, uses @mars/sdk)
└── admin/                 # (optional) Author-only admin UI
    └── index.html         # Admin frontend entry (same MARS SDK pattern)
```

### Optional: Admin Interface (`admin/`)

If the app needs a management UI only visible to the app author:

1. Add an `admin/` directory with its own `index.html` (or subdirectory like `admin/dist/index.html`).
2. Declare `entry.admin` in `manifest.json`:
   ```json
   "entry": {
     "frontend": "frontend/index.html",
     "backend": "backend/main.ts",
     "admin": "admin/index.html"
   }
   ```
3. The platform serves the admin UI at `/a/:appId/__admin/` — **author-only access** (401/403 for others).
4. The admin page uses the **same MARS SDK PostMessageTransport** as the regular frontend. No difference in client code.
5. In the backend, distinguish admin calls via `ctx.isAdmin`:
   ```typescript
   case "admin.stats":
     if (!ctx.isAdmin) throw new MarsException({ code: "PERMISSION_DENIED", message: "仅管理员可访问", details: {} });
     return await getAdminStats(ctx);
   ```
6. In Portal Web, the "我的作品" tab shows a ⚙️ 管理 button for apps that have `has_admin_page: true` in the API response (derived from `manifest.entry.admin`).

## Step-by-Step Generation Workflow

Follow these steps in order when generating a new app-type mini-app:

### Step 1: Clarify Requirements

Before writing any code, determine:
1. **App name** — lowercase kebab-case (e.g., `todo-app`, `weather-widget`)
2. **App description** — one-sentence summary of what the app does (Chinese preferred)
3. **App tags** — up to 5 keywords for discoverability (e.g., `["工具", "效率", "待办"]`)
4. **Capabilities needed** — which platform APIs the app uses (db, auth, storage, http, media, etc.)
5. **Permission scopes** — what user data or device access is required

### Step 2: Generate manifest.json

> **⚠️必填字段** — 缺少任一字段将导致 devTool 报错 `missing field` 解析失败：
> `name`, `version`, `runtime`, `entry.frontend`, `capabilities`
>
> 其中 `runtime` 必须为 `"mars@1.0"`，不可省略、不可留空、不可使用其他格式。
> `entry.backend` 为**可选**字段，纯前端应用可省略；`description`、`tags` 非强制验证字段但强烈建议填写。

```json
{
    "name": "<app-name>",
    "description": "<one-sentence app description>",
    "tags": ["<tag1>", "<tag2>"],
    "version": "1.0.0",
    "runtime": "mars@1.0",
    "entry": {
        "frontend": "frontend/index.html",
        "backend": "backend/main.ts"
    },
    "capabilities": ["<cap1>", "<cap2>"]
}
```

Rules:
- `name`: app unique identifier, lowercase. Platform validates: min 3 chars, must start and end with a letter or digit, only hyphens allowed in the middle (e.g., `my-app` ✓, `-app` ✗, `a` ✗)
- `description`: brief human-readable summary of the app's purpose (Chinese preferred)
- `tags`: array of keyword strings for search and categorization, maximum 5 items
- `version`: semantic versioning, start with `"1.0.0"`
- `runtime`: always `"mars@1.0"` for v1.0 apps — **required, never omit**
- `entry`: **must be an object** `{ "frontend": "...", "backend": "..." }`, NOT a string. Passing `"entry": "frontend/index.html"` causes `expected struct ManifestEntry` parse error
- `capabilities`: only list what the app actually uses — principle of least privilege
- Do NOT include `app_id` — the platform assigns it on first upload

Available capabilities for app-type:
- **Core**: `db`, `storage`, `auth`, `http`
- **Media**: `media` (album access)
- **Connectivity**: `bluetooth`, `nfc`
- **System**: `clipboard`, `filesystem`, `calendar`

### Step 3: Generate permissions.json

```json
{
    "name": "<app-name>",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "<domain>.<target>.<action>",
            "purpose": "<human-readable explanation in Chinese>",
            "consentRequired": true | false
        }
    ]
}
```

Permission scope naming convention — always three segments: `<domain>.<target>.<action>`

Common scopes and their consent requirements:

| Scope | consentRequired | Notes |
|-------|----------------|-------|
| `user.id.read` | false | Basic identity |
| `user.profile.read` | true | Nickname, avatar |
| `user.email.read` | true | Sensitive |
| `user.phone.read` | true | Sensitive |
| `location.precise.read` | true | GPS-level, avoid if possible |
| `location.coarse.read` | true | City-level, prefer this |
| `media.album.read` | true | Photo picker |
| `media.album.write` | true | Save to album |
| `camera.stream.capture` | true | Camera access |
| `microphone.audio.record` | true | Microphone |
| `bluetooth.device.connect` | true | Needs serviceUUIDs constraint |
| `nfc.tag.read` | true | Hardware interaction |
| `clipboard.content.read` | true | Must show prompt |
| `clipboard.content.write` | false | Write-only, no privacy risk |
| `filesystem.public.read` | true | Public directory |
| `filesystem.public.write` | true | Public directory |
| `calendar.event.read` | true | Rarely granted |
| `calendar.event.write` | true | Rarely granted |

Rules:
- `purpose` field should be written in Chinese, clearly explaining why the permission is needed
- Optional `dataRetentionDays` field for data that has retention limits
- `contacts.list.read` is DEPRECATED — never use it in new apps
- Network permissions (`network.*.connect`) must include `constraints.allowHosts` array, no wildcard hosts
- Only declare permissions the app actually needs — reviewers will reject unnecessary permissions

### Step 4: Generate Backend (backend/main.ts)

```typescript
import { serve, MarsException } from "@mars/sdk";
import type { MarsContext } from "@mars/sdk";

// 建表：必须在使用 db.read/db.write 之前执行 CREATE TABLE
async function ensureSchema(ctx: MarsContext) {
    await ctx.db.execute(
        `CREATE TABLE IF NOT EXISTS <table_name> (
            <col1> <type>,
            <col2> <type>
        )`,
        [],
    );
}

serve({
    async rpc(method: string, params: unknown, ctx: MarsContext) {
        await ensureSchema(ctx);
        switch (method) {
            case "<namespace>.<action>":
                return await handleAction(params as <ParamType>, ctx);
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

Backend patterns:
- Always register via `serve({ rpc(...) {} })` — this is the only valid entry point
- **Must call `CREATE TABLE IF NOT EXISTS` before any `db.read()`/`db.write()` call** — the devTool sandbox enforces table existence; calling read/write on a non-existent table throws `TABLE_NOT_FOUND`
- Define an `ensureSchema()` helper that creates all required tables, and call it at the top of `rpc()`
- Use `ctx.auth.requireUser()` to get authenticated user before data operations
- Use `ctx.db.read()` / `ctx.db.write()` / `ctx.db.execute()` / `ctx.db.query()` for database
- Use `ctx.storage.put()` / `ctx.storage.get()` for file/blob storage
- Throw `MarsException` with proper error codes for all error cases
- RPC method names should use `<namespace>.<action>` format (e.g., `todo.create`, `order.list`)
- Validate params at the boundary — check required fields exist and have correct types
- Standard error codes: `INVALID_PARAMS`, `UNAUTHORIZED`, `PERMISSION_DENIED`, `NOT_FOUND`, `TIMEOUT`, `RESOURCE_EXHAUSTED`, `INTERNAL_ERROR`

### Step 5: Generate Frontend (frontend/index.html)

Single HTML file with inline CSS and JS. Use this structure:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>App Name</title>
    <style>
        /* Inline styles — mobile-first, responsive */
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; }
    </style>
</head>
<body>
    <!-- App UI -->

    <script type="module">
        // MARS runtime injects transport at window.__marsTransport
        const transport = window.__marsTransport;

        async function invoke(method, params) {
            const requestId = crypto.randomUUID();
            const res = await transport.send({ requestId, method, params });
            if (res.error) throw new Error(res.error.message);
            return res.data;
        }

        // App logic using invoke() to call backend RPCs
    </script>
</body>
</html>
```

**Platform capsule safe zone (top-right corner)**
The host app overlays a read-only translucent capsule ( `···` more + `✕` close ) in the **top-right** of the app viewport, approximately `87 × 38pt`, offset `8pt` from the top-right of the app content area (just below the status bar). Developers must not place tappable controls, navigation, or critical content in this region. The devTool phone simulator shows a matching non-interactive capsule preview so developers can visualize the reserved zone during development.

Frontend patterns:
- The runtime injects `window.__marsTransport` — use it to send RPC requests
- RPC request format: `{ requestId, method, params }`
- RPC response format: `{ requestId, data, error }` — data and error are mutually exclusive
- For TypeScript apps, prefer `MarsClient` + `HttpTransport` (or `WsTransport` / `PostMessageTransport`):
  ```typescript
  import { MarsClient, HttpTransport } from "@mars/sdk";
  const transport = new HttpTransport("https://api.example.com", { rpcPath: "/rpc" });
  const client = new MarsClient(transport);
  const result = await client.invoke("todo.create", { text: "Buy milk" });
  ```
- For inline single-file HTML, use `window.__marsTransport` directly
- All styles and scripts should be inline (single-file deployment)
- Use `lang="zh-CN"` for Chinese apps
- Mobile-first responsive design
- Use `crypto.randomUUID()` for request IDs

## Code Style Requirements

- 4 spaces per indent level, no tabs
- TypeScript for backend code
- Inline HTML/CSS/JS for frontend (single-file `index.html`)
- camelCase for TypeScript interface fields and RPC envelope fields (`requestId`, `method`, `params`, `data`, `error`)
- snake_case for audit/governance HTTP API fields (`app_id`, `user_id`)
- Chinese comments and UI text where appropriate
- Error messages in English for consistency with the SDK

## Common Mistakes to Avoid

1. **Including `app_id` in manifest/permissions** — The platform assigns this; never include it
2. **Using `contacts.list.read`** — This scope is deprecated and will be rejected
3. **Wildcard hosts in network permissions** — `allowHosts` must list specific domains
4. **Trying to access filesystem/subprocess directly** — All access goes through `MarsContext`
5. **Forgetting to validate params in RPC handlers** — Always validate at the boundary
6. **Not calling `ctx.auth.requireUser()` before data operations** — Most operations need authentication
7. **Missing `purpose` field in permissions** — Every permission needs a clear Chinese-language purpose
8. **Using tabs instead of 4-space indentation** — Project convention is 4 spaces
9. **Forgetting to CREATE TABLE before using db.read()/db.write()** — The devTool sandbox requires explicit `db.execute("CREATE TABLE IF NOT EXISTS ...", [])` before any table can be read or written. Without it, you get a `TABLE_NOT_FOUND` error
10. **Using game capabilities in app-type** — `gpu`, `audio`, `sensor`, `gamepad`, `haptics`, `assetCache` are for game-type only. Use `mars-cocos-game-generator` or `mars-unity-game-generator` for games
11. **Importing `@mars/sdk/common`** — This path is not exported by the SDK. Use `@mars/sdk` only
12. **Passing SQL to `db.read()` or `db.write()`** — These methods take a **table name** (e.g., `"todos"`), not a SQL string. Writing `db.read("SELECT ... FROM todos", [])` causes the runtime to look for a table named `"SELECT"` → `TABLE_NOT_FOUND`. Use `db.query()` for SQL SELECT queries
13. **Omitting required manifest.json fields** — `name`, `description`, `tags`, `version`, `runtime`, `entry`, `capabilities` are ALL required. Missing any field (especially `runtime`) causes devTool to throw `missing field` parse error. Always include `"runtime": "mars@1.0"`
14. **Using a string for `entry` instead of an object** — `entry` must be `{ "frontend": "...", "backend": "..." }`. Writing `"entry": "frontend/index.html"` causes `expected struct ManifestEntry` parse error
15. **Placing UI elements in the platform capsule zone** — The host app overlays a `···` / `✕` capsule in the top-right corner of the app content area (~87 × 38pt, 8pt from top-right edge). Never place buttons, headings, tabs, or any interactive element there. See `docs/轻应用开发指南.md` → *前端界面安全区域* for the exact dimensions and a visual diagram. The devTool simulator displays the capsule as a non-interactive overlay to help visualize the reserved area

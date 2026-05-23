---
name: mars-cocos-game-generator
description: >
    Generate MARS-compliant Cocos Creator game-type mini-app (游戏类微应用) packages. Use this skill when
    the user wants to create a game mini-app using Cocos Creator engine. Also use when the user mentions
    "cocos", "cocos creator", "react-native-cocos2dx", "MarsGameClient", "@mars/sdk/game",
    "NativeBridgeTransport", "appType game" with cocos, "cdn-manifest", "game.ts", or asks about
    game mini-app package structure with Cocos. This skill covers Cocos Creator game-type mini-apps
    running natively on Expo/React Native via react-native-cocos2dx, with platform service access
    via MarsGameClient and a TypeScript backend for game services (leaderboard, saves, matchmaking).
    For Unity games, use mars-unity-game-generator. For app-type mini-apps, use mars-app-generator.
---

# MARS Cocos Creator Game Generator

Generate complete, protocol-compliant MARS game-type mini-app packages using **Cocos Creator** engine.
This skill encodes the MARS v1.0 protocol standard, game SDK API surface, permission model, and
packaging conventions for Cocos Creator games running natively on Expo/React Native.

Related skills:
- `mars-unity-game-generator` — Unity games on Expo/React Native
- `mars-app-generator` — App-type mini-apps (WebView frontend + backend)

Read the reference files in this skill directory for detailed specifications:
- `references/protocol.md` — Game package structure, manifest schema, cdn-manifest, RPC protocol
- `references/permissions.md` — Permission scopes for game-type apps
- `references/sdk-api.md` — MarsGameClient, NativeBridgeTransport, backend handler patterns

## SDK Compatibility Guardrails

Always generate game code with the exact SDK entrypoints below. Do not invent subpaths.

- Backend import (required):
    - `import { serve, MarsException } from "@mars/sdk"`
    - `import type { MarsContext } from "@mars/sdk"`
- Cocos game frontend import (required):
    - `import { MarsGameClient } from "@mars/sdk/game"`
- Game frontend — advanced transport (when needed):
    - `import { NativeBridgeTransport } from "@mars/sdk"` — only available here, NOT in `@mars/sdk/game`
    - `import type { MarsNativeBridge } from "@mars/sdk"` — same as above
- Forbidden imports:
    - `@mars/sdk/common` (not exported — does not exist)
    - any other unlisted subpath such as `@mars/sdk/transport`, `@mars/sdk/core`, etc.

When fixing existing projects with SDK mismatch errors:
1. Replace backend `@mars/sdk/common` -> `@mars/sdk`.
2. Keep frontend on `@mars/sdk/game`.
3. Ensure devTool import map includes `@mars/sdk` and `@mars/sdk/game`.
4. Do NOT suggest `deno add jsr:@mars/sdk/common`.

## Package File Count Constraint (Hard Limit)

> **⚠️ MANDATORY**: The platform enforces a **maximum of 2,000 files** per ZIP package.
> This applies to the uploaded package only — CDN-hosted assets (via `cdn-manifest.json`) are
> not subject to this limit. Never generate game packages that bundle per-asset individual files.

### Forbidden patterns

```
❌ assets/levels/level_001.json     ← individual JSON per level
❌ assets/levels/level_002.json
   ... (hundreds more)
❌ backend/data/chars/一.json       ← individual JSON per record
```

### Required patterns for bulk data

| Data type | Correct approach |
|-----------|------------------|
| Game levels / maps | Merged JSON array in one file or `cdn-manifest.json` |
| Character/item data | Single merged JSON + seed into `ctx.db` on first run |
| Large binary assets | Use `cdn-manifest.json` (CDN-hosted, not in ZIP) |
| Per-user save data | `ctx.db` (SQLite via `db` capability) |
| Static lookup (large) | Single merged JSON with module-level cache (see app-generator skill) |

## Database Schema Initialization (Hard Constraint)

> **⚠️ MANDATORY**: Every backend that uses `ctx.db` MUST call `CREATE TABLE IF NOT EXISTS` **before** any
> `db.read()`, `db.write()`, `db.query()`, or `db.execute()` (non-DDL) call. The devTool sandbox enforces
> table existence at runtime — calling read/write on a non-existent table throws `TABLE_NOT_FOUND`.

The required pattern:
1. Define an `ensureSchema(ctx)` async helper that executes all `CREATE TABLE IF NOT EXISTS` statements
2. Call `await ensureSchema(ctx)` as the **first line** inside the `rpc()` handler, before the `switch`
3. Table names must be valid identifiers (e.g., `scores`, `saves`) — never SQL keywords

```typescript
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

> **⚠️ CRITICAL**: `db.read()` and `db.write()` take a **plain table name** (e.g., `"scores"`), NOT a SQL statement.
> Passing SQL like `db.read("SELECT ... FROM scores", [])` is **WRONG** — the runtime interprets `"SELECT"` as
> the table name and throws `TABLE_NOT_FOUND`. Use `db.query()` for SQL queries instead.

```typescript
// ✅ CORRECT: db.read takes table name + filter object
const rows = await ctx.db.read("scores", { userId: user.id });

// ✅ CORRECT: db.query takes SQL string + params array
const rows = await ctx.db.query(
    "SELECT userId, MAX(score) as bestScore FROM scores GROUP BY userId ORDER BY bestScore DESC LIMIT ?",
    [10],
);

// ❌ WRONG: passing SQL to db.read — will throw TABLE_NOT_FOUND on table "SELECT"
const rows = await ctx.db.read(
    "SELECT * FROM scores WHERE userId = ?",
    [user.id],
);
```

## When to Use

- User wants to **create a Cocos Creator game** as a MARS mini-app
- User mentions **Cocos**, **Cocos Creator**, **react-native-cocos2dx**
- User asks about **MarsGameClient** or **@mars/sdk/game** usage
- User wants to **scaffold** a game with leaderboard, saves, or matchmaking
- User asks about **NativeBridgeTransport** or Native Bridge communication
- User asks about **cdn-manifest.json** for game assets

> **Not for app-type!** If the user wants a standard WebView app, use `mars-app-generator`.
> **Not for Unity!** If the user wants a Unity game, use `mars-unity-game-generator`.

## Architecture Overview

Cocos Creator game-type mini-apps run **natively** on the Expo/React Native host via
`react-native-cocos2dx`. The architecture:

```
┌─────────────────────────────────────────────┐
│  Expo / React Native Host App               │
│  ┌───────────────────────────────────────┐  │
│  │  Cocos Creator (react-native-cocos2dx)│  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  game.ts                        │  │  │
│  │  │  └── MarsGameClient             │  │  │
│  │  │      ├── context (auth,db,store)│  │  │
│  │  │      ├── invoke() → backend RPC │  │  │
│  │  │      └── subscribe() → events   │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────┬───────────────────────────┘  │
│              │ Native Bridge (JSI/MC)        │
│  ┌───────────┴───────────────────────────┐  │
│  │  MARS Runtime                         │  │
│  │  └── backend/main.ts (serve())        │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

Key points:
- **Rendering, audio, sensors, haptics** → handled by Cocos Creator natively
- **Platform services (auth, storage, db)** → accessed via `MarsGameClient`
- **Backend (leaderboard, saves, matchmaking)** → standard `serve()` pattern with `MarsContext`
- **Communication** → Native Bridge (JSI/MessageChannel) via `NativeBridgeTransport`

## Package Structure

```
my-cocos-game/
├── manifest.json          # appType: "game", engine: "cocos"
├── permissions.json       # Platform service permissions only
├── cdn-manifest.json      # Large game assets (Wasm, textures, audio)
├── frontend/              # Cocos Creator project directory
│   └── game.ts            # Engine entry using MarsGameClient
├── backend/
│   └── main.ts            # Game backend (leaderboard, saves, matchmaking)
├── admin/                 # (可选) 作者专属管理后台 UI
│   └── index.html
└── build/                 # (可选) Cocos Creator Web 构建产物
    └── web-mobile/        # DevTool 可直接 iframe 预览
        ├── index.html
        └── *.js / *.wasm
```

### Optional: Admin Interface (`admin/`)

Game mini-apps can include an author-only admin UI (e.g. leaderboard management, ban tools, live config):

1. Add `admin/index.html` (or a subdirectory like `admin/dist/index.html`).
2. Declare `entry.admin` in `manifest.json`:
   ```json
   "entry": {
     "frontend": "frontend/",
     "backend": "backend/main.ts",
     "admin": "admin/index.html"
   }
   ```
3. Platform serves the admin UI at `/a/:appId/__admin/` — **author-only**, 401/403 for others.
4. The admin page is a regular WebView (not Cocos) — use MARS SDK `PostMessageTransport` like an app-type frontend.
5. Protect admin RPC methods in the backend with `ctx.isAdmin`:
   ```typescript
   case "admin.banUser":
     if (!ctx.isAdmin) throw new MarsException({ code: "PERMISSION_DENIED", message: "仅管理员可访问", details: {} });
     return await banUser(params, ctx);
   ```
6. Portal Web "我的作品" tab shows a ⚙️ 管理 button for games with `has_admin_page: true`.

## Step-by-Step Generation Workflow

### Step 1: Clarify Requirements

Before writing any code, determine:
1. **Game name** — lowercase kebab-case (e.g., `space-shooter`, `puzzle-quest`)
2. **Game description** — one-sentence summary (Chinese preferred)
3. **Game tags** — up to 5 keywords (e.g., `["游戏", "射击", "休闲"]`)
4. **Backend needs** — leaderboard, cloud saves, matchmaking, in-game purchases?
5. **Permission scopes** — what user data is needed (identity, profile, scores)
6. **Large assets?** — does the game have Wasm binaries, texture packs, or large audio files?

### Step 2: Generate manifest.json

> **⚠️必填字段** — 缺少任一字段将导致 devTool 报错 `missing field` 解析失败：
> `name`, `version`, `runtime`, `appType`, `engine`, `capabilities`
>
> 其中 `runtime` 必须为 `"mars@1.0"`，`appType` 必须为 `"game"`，`engine` 必须为 `"cocos"`。
> `entry.backend` 为**可选**字段，无后端的游戏可省略；`resourceProfile`、`description`、`tags` 非强制验证但强烈建议填写。

```json
{
    "name": "<game-name>",
    "description": "<one-sentence game description>",
    "tags": ["游戏", "<genre>", ...],
    "version": "1.0.0",
    "runtime": "mars@1.0",
    "appType": "game",
    "engine": "cocos",
    "resourceProfile": "game",
    "entry": {
        "frontend": "frontend/",
        "backend": "backend/main.ts"
    },
    "webPreview": "build/web-mobile",
    "capabilities": ["auth", "storage", "db"]
}
```

Rules:
- `appType`: always `"game"` for game mini-apps
- `engine`: always `"cocos"` for Cocos Creator games
- `resourceProfile`: `"game"` for enhanced resource quota
- `entry`: **must be an object** `{ "frontend": "...", "backend": "..." }`, NOT a string. Passing `"entry": "frontend/game.ts"` causes `expected struct ManifestEntry` parse error
- `entry.frontend`: directory path (not a single HTML file) — Cocos project root
- `entry.backend`: path to TypeScript backend entry
- `webPreview`: (可选) 指向 Cocos Creator 的 Web 构建产物目录（如 `build/web-mobile` 或 `build/web-desktop`）。配置后 DevTool 可在 iframe 中直接预览游戏，无需真机
- `capabilities`: ONLY platform service capabilities (`auth`, `storage`, `db`). Do NOT include `gpu`, `audio`, `sensor`, etc. — Cocos Creator handles these natively
- Do NOT include `app_id` — platform assigns it on first upload

### Step 3: Generate permissions.json

```json
{
    "name": "<game-name>",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "user.id.read",
            "purpose": "用于排行榜身份识别",
            "consentRequired": false
        },
        {
            "scope": "user.profile.read",
            "purpose": "用于排行榜展示玩家昵称与头像",
            "consentRequired": true
        },
        {
            "scope": "db.scores.read",
            "purpose": "读取排行榜数据",
            "consentRequired": false
        },
        {
            "scope": "db.scores.write",
            "purpose": "提交游戏得分至排行榜",
            "consentRequired": false
        }
    ]
}
```

Common game permission scopes:

| Scope | consentRequired | Use Case |
|-------|----------------|----------|
| `user.id.read` | false | Player identity for leaderboard/saves |
| `user.profile.read` | true | Show player name/avatar in leaderboard |
| `db.<table>.read` | false | Read leaderboard/save data |
| `db.<table>.write` | false | Write scores/save data |
| `storage.<bucket>.read` | false | Read game save files |
| `storage.<bucket>.write` | false | Write game save files |

Rules:
- Only request platform service permissions — NO rendering/audio/sensor permissions
- `purpose` field in Chinese
- Principle of least privilege

### Step 4: Generate Backend (backend/main.ts)

Game backends handle leaderboard, cloud saves, matchmaking, etc.:

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

Backend patterns:
- Same `serve()` pattern as app-type backends
- **Must call `CREATE TABLE IF NOT EXISTS`** before any db operations
- Use `ctx.auth.requireUser()` for player identity
- Common RPC methods: `score.submit`, `leaderboard.top`, `save.put`, `save.get`, `match.create`, `match.join`
- Validate all params at the boundary

### Step 5: Generate Game Frontend (frontend/game.ts)

Cocos Creator game entry using `MarsGameClient` for platform service access:

```typescript
import { MarsGameClient } from "@mars/sdk/game";

// Auto-discovers Native Bridge and context injected by RN host
const mars = await MarsGameClient.create();

// === Platform Services ===

// Get current player
const user = await mars.context.auth.requireUser();
console.log(`Welcome, player ${user.id}`);

// Read leaderboard via backend RPC
const { entries } = await mars.invoke<{ entries: { userId: string; bestScore: number }[] }>(
    "leaderboard.top",
    { limit: 10 },
);

// Submit score via backend RPC
await mars.invoke("score.submit", { score: 9999 });

// Cloud saves
await mars.invoke("save.put", { data: { level: 5, items: ["shield", "sword"] } });
const { data: saveData } = await mars.invoke<{ data: unknown }>("save.get");

// Subscribe to real-time events (e.g., multiplayer)
const unsub = mars.subscribe("match.update", (data) => {
    console.log("Match update:", data);
});

// === Cocos Creator handles everything else ===
// Rendering, audio, physics, input, sensors → Cocos Creator native APIs
// Do NOT use MarsGameClient for these — they are engine-managed
```

Frontend patterns:
- Import from `@mars/sdk/game`, NOT from `@mars/sdk`
- `MarsGameClient.create()` auto-discovers `globalThis.__marsBridge` injected by RN host
- `MarsGameFrontendContext` only provides: `auth`, `storage`, `db`, `env`
- Use `mars.invoke()` to call backend RPC methods
- Use `mars.subscribe()` for real-time server events
- Rendering, audio, sensors, haptics → Cocos Creator APIs directly
- No WebView, no HTML, no `window.__marsTransport`

### Step 6: Generate cdn-manifest.json

> **必须生成** — devTool 对游戏类应用会检查此文件是否存在，缺少时提示「游戏类应用建议包含 cdn-manifest.json」。
> 即使没有大资产，也应生成一个空 assets 数组的 cdn-manifest.json。

游戏无大资产时的最小文件：
```json
{
    "version": "1",
    "assets": []
}
```

游戏有大资产时（Wasm 二进制、纹理包、音频文件）：
```json
{
    "version": "1",
    "assets": [
        {
            "path": "assets/game.wasm.br",
            "size": 8388608,
            "sha256": "<sha256-hex>",
            "contentType": "application/wasm"
        },
        {
            "path": "assets/textures/sprites.atlas",
            "size": 2097152,
            "sha256": "<sha256-hex>",
            "contentType": "application/octet-stream"
        }
    ]
}
```

Rules:
- Asset paths MUST be under `assets/` directory
- `sha256` is mandatory — platform verifies integrity
- Large assets go here, NOT in `frontend/`
- Platform uploads to CDN and injects `ctx.env.MARS_CDN_BASE_URL` at runtime

### Step 7: Generate Web Build Directory (build/web-mobile/)

**必须生成**此目录，确保 DevTool 可直接预览游戏：

```
build/
└── web-mobile/
    ├── index.html      # Web 入口，全屏 Canvas
    └── game.js         # 纯 Canvas 2D/WebGL 实现的可运行游戏
```

- `index.html`：包含 `<canvas id="GameCanvas">`，全屏布局，引用 `game.js`
- `game.js`：使用 Canvas 2D 或 WebGL 实现完整可玩的游戏逻辑（不是占位符）
- 菜单画面、游戏主循环、输入处理（触摸 + 键盘）、碰撞检测、分数、结算画面等基本要素都应包含
- 此目录模拟 Cocos Creator 的 web-mobile 构建产物，但用纯 JS 实现以便在 DevTool iframe 中直接运行
- **不要**生成纯占位页面（仅显示“placeholder”文字），必须是可交互的实际游戏

参考 `examples/space-shooter/build/web-mobile/` 中的实现。

- 4 spaces per indent level, no tabs
- TypeScript for backend and game entry
- camelCase for TypeScript fields and RPC envelope fields
- snake_case for audit/governance HTTP API fields
- Chinese comments and UI text where appropriate
- Error messages in English for consistency with the SDK

## DevTool Web Build Preview

Cocos Creator 支持构建为 Web 平台（web-mobile / web-desktop），构建产物可在 DevTool 中直接 iframe 预览：

1. 在 Cocos Creator 中执行 **构建发布 → Web Mobile**（或 Web Desktop），输出到 `build/web-mobile/`
2. 在 `manifest.json` 中添加 `"webPreview": "build/web-mobile"`
3. DevTool 启动项目后自动检测该目录，在预览面板中以 iframe 加载游戏
4. 支持设备模拟（尺寸、横竖屏）、缩放、截图、录屏

自动检测：即使不配置 `webPreview` 字段，DevTool 也会自动检测以下常见目录：
- `build/web-mobile/`
- `build/web-desktop/`

> **注意**：Web Build 预览仅用于开发调试。生产环境中 Cocos 游戏通过 `react-native-cocos2dx` 原生运行。

> **重要**：生成 Cocos 游戏项目时，**必须**在 `manifest.json` 中包含 `"webPreview": "build/web-mobile"` 字段，
> 并创建 `build/web-mobile/` 目录（至少含 `index.html` + 游戏代码），确保 DevTool 可直接预览。
> 否则预览面板会显示占位提示「可在 manifest.json 中添加 webPreview 字段」而非实际游戏画面。

## Common Mistakes to Avoid

1. **Including GPU/audio/sensor capabilities** — Cocos Creator handles these natively. Only use `auth`, `storage`, `db` in capabilities
2. **Importing from `@mars/sdk` instead of `@mars/sdk/game`** — Game frontends MUST use `@mars/sdk/game`
3. **Using `window.__marsContext` or `window.__marsTransport`** — These are for WebView apps. Games use `MarsGameClient`
4. **Forgetting the backend** — Most games need a backend for leaderboard/saves
5. **Putting large assets in `frontend/`** — Use `cdn-manifest.json` for CDN hosting
6. **Including `app_id` in manifest** — Platform assigns this
7. **Using `entry.frontend` as an HTML file path** — Game frontend entry is a directory, not `index.html`
8. **Forgetting to CREATE TABLE before db operations** — Call `CREATE TABLE IF NOT EXISTS` first
9. **Not validating params in RPC handlers** — Always validate at the boundary
10. **Missing `purpose` field in permissions** — Every permission needs Chinese-language purpose
11. **Forgetting `webPreview` field in manifest.json** — 生成 Cocos 游戏时必须包含 `"webPreview": "build/web-mobile"` 并创建对应的 `build/web-mobile/` 目录（含 `index.html` + 游戏 JS），否则 DevTool 预览面板只显示占位提示
12. **Importing backend from `@mars/sdk/common`** — Game backend must import from `@mars/sdk`; `@mars/sdk/common` is invalid
13. **Passing SQL to `db.read()` or `db.write()`** — These methods take a **table name** (e.g., `"scores"`), not a SQL string. Writing `db.read("SELECT ... FROM scores", [])` causes the runtime to look for a table named `"SELECT"` → `TABLE_NOT_FOUND`. Use `db.query()` for SQL SELECT queries
14. **Omitting required manifest.json fields** — `name`, `description`, `tags`, `version`, `runtime`, `appType`, `engine`, `resourceProfile`, `entry`, `capabilities` are ALL required. Missing any field (especially `runtime`) causes devTool to throw `missing field` parse error. Always include `"runtime": "mars@1.0"`
15. **Not generating cdn-manifest.json** — devTool 对游戏类应用会检查 cdn-manifest.json 是否存在。即使没有大资产，也必须生成包含空 `assets` 数组的 `{ "version": "1", "assets": [] }`
16. **Using a string for `entry` instead of an object** — `entry` must be `{ "frontend": "...", "backend": "..." }`. Writing `"entry": "frontend/game.ts"` causes `expected struct ManifestEntry` parse error
17. **Placing game HUD elements in the platform capsule zone** — The host app overlays a `···` / `✕` capsule in the top-right corner of the game viewport (~87 × 38pt, 8pt from top-right edge). Place score displays, pause buttons, shop icons, and other HUD elements clear of this zone — otherwise they will be obscured. In Cocos Creator, set the top-right anchor of any Canvas HUD node to leave at least `(95, 50)pt` of clearance from the top-right corner. The devTool phone simulator displays the capsule as a non-interactive overlay so you can verify clearance during development. See `docs/轻应用开发指南.md` → *前端界面安全区域* for the exact dimensions

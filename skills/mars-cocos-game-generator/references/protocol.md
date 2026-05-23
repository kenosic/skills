# MARS Protocol — Game Package Structure & RPC Protocol Reference (Cocos Creator)

Protocol details for MARS v1.0 **Cocos Creator game-type** mini-app packages.
For app-type protocol details, see the `mars-app-generator` skill.

## Table of Contents
1. [Directory Structure](#1-directory-structure)
2. [manifest.json Schema](#2-manifestjson-schema)
3. [cdn-manifest.json Schema](#3-cdn-manifestjson-schema)
4. [RPC Protocol](#4-rpc-protocol)
5. [Error Model](#5-error-model)
6. [Runtime Capability Boundaries](#6-runtime-capability-boundaries)
7. [Field Naming Convention](#7-field-naming-convention)
8. [Complete Example](#8-complete-example)

---

## 1. Directory Structure

```
game.mars/
├── manifest.json
├── permissions.json
├── cdn-manifest.json       # CDN-hosted large assets
├── frontend/               # Cocos Creator project directory
│   └── game.ts             # Engine entry using MarsGameClient
├── backend/
│   └── main.ts             # Game backend (leaderboard, saves, matchmaking)
└── build/                  # (可选) Cocos Creator Web 构建产物
    └── web-mobile/         # DevTool 可直接 iframe 预览
        ├── index.html
        └── *.js / *.wasm
```

Rules:
- Game assets (Wasm, textures, audio) MUST NOT be placed inside `frontend/`
- Large assets MUST be declared in `cdn-manifest.json`
- Game runs natively via `react-native-cocos2dx` on Expo/React Native — NO WebView
- `build/web-mobile/` is optional, used only for DevTool preview during development

---

## 2. manifest.json Schema

### Required Fields

```json
{
    "name": "my-cocos-game",
    "description": "一句话游戏描述",
    "tags": ["游戏", "射击"],
    "version": "1.0.0",
    "runtime": "mars@1.0",
    "appType": "game",
    "engine": "cocos",
    "resourceProfile": "game",
    "entry": {
        "frontend": "frontend/",
        "backend": "backend/main.ts"
    },
    "capabilities": ["auth", "storage", "db"]
}
```

### Field Constraints

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | YES | Unique game name, lowercase kebab-case |
| `description` | string | YES | Brief summary (Chinese preferred) |
| `tags` | string[] | YES | Keywords, max 5. First tag should be `"游戏"` |
| `version` | string | YES | Semantic version (e.g., `"1.0.0"`) |
| `runtime` | string | YES | Always `"mars@1.0"` |
| `appType` | `"game"` | YES | Must be `"game"` |
| `engine` | `"cocos"` | YES | Must be `"cocos"` for Cocos Creator |
| `resourceProfile` | `"game"` | YES | Enhanced resource quota |
| `entry.frontend` | string | YES | Cocos project directory path |
| `entry.backend` | string | NO | Backend TS entry; omit only if no backend needed |
| `capabilities` | string[] | YES | Platform service capabilities ONLY |
| `webPreview` | string | NO | Cocos Web 构建产物目录路径（如 `build/web-mobile`），配置后 DevTool 可直接 iframe 预览 |

**Critical rules:**
- MUST NOT include `app_id` — platform assigns on first upload
- `capabilities` can ONLY include: `auth`, `storage`, `db`
- Do NOT include `gpu`, `audio`, `sensor`, `gamepad`, `haptics`, `assetCache` — Cocos Creator handles these natively

---

## 3. cdn-manifest.json Schema

For games with large assets (Wasm, textures, audio).

```json
{
    "version": "1",
    "assets": [
        {
            "path": "assets/game.wasm.br",
            "size": 8388608,
            "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
            "contentType": "application/wasm"
        },
        {
            "path": "assets/textures/sprites.atlas",
            "size": 2097152,
            "sha256": "a1b2c3d4...",
            "contentType": "application/octet-stream"
        }
    ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | YES | Schema version, currently `"1"` |
| `assets[].path` | string | YES | Must be under `assets/` |
| `assets[].size` | number | YES | File size in bytes |
| `assets[].sha256` | string | YES | SHA-256 hex digest |
| `assets[].contentType` | string | YES | MIME type |

Platform behavior:
- Uploads declared assets to CDN at publish time
- Injects `ctx.env.MARS_CDN_BASE_URL` at runtime
- Verifies `sha256` — publish fails on mismatch

---

## 4. RPC Protocol

### Request Format

```json
{
    "requestId": "2b6fbf95-d9b8-4d07-b9d6-9f76c47e8f7b",
    "method": "score.submit",
    "params": { "score": 9999 }
}
```

- `requestId`: UUID, MUST be unique per request
- `method`: `<namespace>.<action>` format
- `params`: optional JSON payload

### Success Response

```json
{
    "requestId": "2b6fbf95-d9b8-4d07-b9d6-9f76c47e8f7b",
    "data": { "id": "s_123" },
    "error": null
}
```

### Error Response

```json
{
    "requestId": "2b6fbf95-d9b8-4d07-b9d6-9f76c47e8f7b",
    "data": null,
    "error": {
        "code": "INVALID_PARAMS",
        "message": "score must be a non-negative number",
        "details": { "field": "score" }
    }
}
```

---

## 5. Error Model

| Code | When to Use |
|------|-------------|
| `INVALID_PARAMS` | Missing or malformed request parameters |
| `UNAUTHORIZED` | No authenticated user |
| `PERMISSION_DENIED` | Permission scope not granted |
| `NOT_FOUND` | Resource or RPC method not found |
| `TIMEOUT` | Operation exceeded time limit |
| `RESOURCE_EXHAUSTED` | Quota exceeded |
| `INTERNAL_ERROR` | Unexpected server error |

```typescript
class MarsException extends Error {
    readonly name: "MarsException";
    readonly code: MarsErrorCode;
    readonly details: Record<string, unknown>;
    constructor(error: MarsError);
}
```

---

## 6. Runtime Capability Boundaries

The runtime MUST enforce:
- No direct host filesystem access from game code
- No subprocess spawning
- All platform capabilities exposed only through `MarsGameClient` (frontend) or `MarsContext` (backend)
- Permission check on every capability invocation
- Game engine (Cocos Creator) manages its own rendering/audio/sensor pipeline

---

## 7. Field Naming Convention

| Context | Convention | Examples |
|---------|-----------|----------|
| RPC envelope fields | camelCase | `requestId`, `method`, `params` |
| TypeScript SDK interfaces | camelCase | `requestId`, `userId`, `bestScore` |
| Permission scopes | dot-separated lowercase | `user.profile.read`, `db.scores.write` |

---

## 8. Complete Example

### Space Shooter manifest.json

```json
{
    "name": "space-shooter",
    "description": "太空飞船射击游戏",
    "tags": ["游戏", "射击", "休闲", "太空"],
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

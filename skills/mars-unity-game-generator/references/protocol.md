# MARS Protocol — Unity Game-Type Mini-App

This reference covers the package structure, manifest schema, cdn-manifest format, and RPC protocol
for **Unity game-type** mini-apps. For Cocos Creator games, see `mars-cocos-game-generator`.

## 1. Directory Structure

```
<game-name>/
├── manifest.json          # App metadata & capabilities (appType: "game", engine: "unity")
├── permissions.json       # Permission scopes for platform services
├── cdn-manifest.json      # Optional: CDN-hosted large game assets
├── frontend/              # Unity engine bridge directory
│   └── game.ts            # Game frontend entry — uses MarsGameClient
├── backend/
│   └── main.ts            # Game backend — serve() handler for leaderboard, saves, etc.
└── Build/                 # (可选) Unity WebGL 构建产物
    └── WebGL/             # DevTool 可直接 iframe 预览
        ├── index.html
        └── Build/ *.js / *.wasm
```

Key differences from app-type:
- `frontend/` contains `game.ts` (not `index.html`). Unity handles rendering natively.
- `cdn-manifest.json` declares large assets (AssetBundles, 3D models) for CDN hosting.
- The `manifest.json` has `appType: "game"` and `engine: "unity"`.
- `Build/WebGL/` is optional, used only for DevTool iframe preview during development.

## 2. manifest.json Schema

```json
{
    "name": "<kebab-case>",
    "description": "<string>",
    "tags": ["<string>", ...],
    "version": "<semver>",
    "runtime": "mars@1.0",
    "appType": "game",
    "engine": "unity",
    "resourceProfile": "game",
    "entry": {
        "frontend": "frontend/",
        "backend": "backend/main.ts"
    },
    "capabilities": ["auth", "storage", "db"]
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | Unique, kebab-case |
| `description` | string | Yes | One-line game description |
| `tags` | string[] | Yes | Up to 5 keywords for discovery |
| `version` | string | Yes | Semver format. Initial: `"1.0.0"` |
| `runtime` | string | Yes | Always `"mars@1.0"` |
| `appType` | string | Yes | Always `"game"` |
| `engine` | string | Yes | Always `"unity"` for Unity |
| `resourceProfile` | string | No | `"game"` for enhanced CPU/memory/GPU quotas |
| `entry.frontend` | string | Yes | Directory path to Unity bridge entry |
| `entry.backend` | string | No | Backend script path (omit if no backend) |
| `capabilities` | string[] | Yes | Platform service capabilities only |
| `webPreview` | string | No | Unity WebGL 构建产物目录路径（如 `Build/WebGL`），配置后 DevTool 可直接 iframe 预览 |

**Capabilities for game-type apps:**
- `auth` — Player identity (almost always needed)
- `storage` — Object storage (game save files, replays)
- `db` — Database (leaderboards, scored, matchmaking)

**Do NOT include:**
- `gpu`, `audio`, `sensor`, `haptics`, `media` — Unity handles these natively
- `bluetooth`, `nfc`, `calendar`, `clipboard`, `filesystem` — App-type only

## 3. cdn-manifest.json

Declares large assets that the platform uploads to CDN and serves via `MARS_CDN_BASE_URL`.

```json
{
    "version": "1",
    "assets": [
        {
            "path": "assets/levels.assetbundle",
            "size": 20971520,
            "sha256": "<hex-string>",
            "contentType": "application/octet-stream"
        },
        {
            "path": "assets/audio/bgm.ogg",
            "size": 4194304,
            "sha256": "<hex-string>",
            "contentType": "audio/ogg"
        },
        {
            "path": "assets/models/characters.fbx",
            "size": 8388608,
            "sha256": "<hex-string>",
            "contentType": "application/octet-stream"
        }
    ]
}
```

| Field | Type | Notes |
|-------|------|-------|
| `version` | string | Schema version: `"1"` |
| `assets[].path` | string | Must be under `assets/` directory |
| `assets[].size` | number | Exact file size in bytes |
| `assets[].sha256` | string | Hex-encoded SHA-256 for integrity verification |
| `assets[].contentType` | string | MIME type |

Common Unity asset types:
- `.assetbundle` → `application/octet-stream` (Unity AssetBundles)
- `.fbx` → `application/octet-stream` (3D models)
- `.ogg`, `.wav` → `audio/ogg`, `audio/wav`
- `.png`, `.jpg` → `image/png`, `image/jpeg` (textures)

Runtime access: `ctx.env.MARS_CDN_BASE_URL + '/' + asset.path`

## 4. RPC Protocol

Game frontends invoke backend methods via RPC over the Native Bridge.

### Request (from MarsGameClient)

```json
{
    "jsonrpc": "2.0",
    "method": "<namespace>.<action>",
    "params": { ... },
    "id": 1
}
```

### Success Response

```json
{
    "jsonrpc": "2.0",
    "result": { ... },
    "id": 1
}
```

### Error Response

```json
{
    "jsonrpc": "2.0",
    "error": {
        "code": "<ERROR_CODE>",
        "message": "<human-readable>",
        "data": { ... }
    },
    "id": 1
}
```

Standard error codes:

| Code | Meaning |
|------|---------|
| `NOT_FOUND` | Unknown RPC method |
| `INVALID_PARAMS` | Malformed or missing params |
| `AUTH_REQUIRED` | User not authenticated |
| `PERMISSION_DENIED` | Scope not granted |
| `INTERNAL` | Unexpected server error |

### Common Game RPC Methods

| Method | Params | Returns | Notes |
|--------|--------|---------|-------|
| `score.submit` | `{ score: number }` | `{ id: string }` | Submit a score |
| `leaderboard.top` | `{ limit?: number }` | `{ entries: Entry[] }` | Top N scores |
| `save.put` | `{ data: unknown }` | `{ saved: boolean }` | Cloud save (upsert) |
| `save.get` | (none) | `{ data: unknown }` | Load cloud save |
| `match.join` | `{ roomId: string }` | `{ matchId: string }` | Join a match |

## 5. Communication Path

```
Unity (game.ts)
  └── MarsGameClient.invoke("score.submit", { score: 100 })
        └── NativeBridgeTransport (JSI/MessageChannel)
              └── React Native Host
                    └── MARS Runtime
                          └── backend/main.ts → rpc("score.submit", { score: 100 }, ctx)
```

- `MarsGameClient` wraps requests in JSON-RPC 2.0 format
- `NativeBridgeTransport` bridges to the React Native host via `globalThis.__marsBridge`
- The MARS runtime routes to the backend `serve()` handler
- The `ctx` (MarsContext) is injected by the runtime

## 6. Complete Example: Tower Defense

### manifest.json
```json
{
    "name": "tower-defense",
    "description": "策略塔防游戏 — 守卫你的领地",
    "tags": ["游戏", "策略", "塔防", "3D"],
    "version": "1.0.0",
    "runtime": "mars@1.0",
    "appType": "game",
    "engine": "unity",
    "resourceProfile": "game",
    "entry": {
        "frontend": "frontend/",
        "backend": "backend/main.ts"
    },
    "capabilities": ["auth", "storage", "db"]
}
```

### permissions.json
```json
{
    "name": "tower-defense",
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
        },
        {
            "scope": "db.saves.read",
            "purpose": "读取云存档数据",
            "consentRequired": false
        },
        {
            "scope": "db.saves.write",
            "purpose": "写入云存档数据",
            "consentRequired": false
        }
    ]
}
```

### cdn-manifest.json
```json
{
    "version": "1",
    "assets": [
        {
            "path": "assets/levels.assetbundle",
            "size": 20971520,
            "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
            "contentType": "application/octet-stream"
        },
        {
            "path": "assets/models/towers.fbx",
            "size": 8388608,
            "sha256": "c5b07e2a3f149afbf4c8996fb92427ae41e4649b934ca49599f88512e3b0c442",
            "contentType": "application/octet-stream"
        }
    ]
}
```

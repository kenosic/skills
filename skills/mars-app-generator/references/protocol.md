# MARS Protocol — Package Structure & RPC Protocol Reference (App-Type)

This reference contains the full protocol details for MARS v1.0 **app-type** mini-app packages.
For game-type protocol details, see the `mars-cocos-game-generator` or `mars-unity-game-generator` skills.

## Table of Contents
1. [Directory Structure](#1-directory-structure)
2. [manifest.json Schema](#2-manifestjson-schema)
3. [RPC Protocol](#3-rpc-protocol)
4. [Error Model](#4-error-model)
5. [Runtime Capability Boundaries](#5-runtime-capability-boundaries)
6. [Field Naming Convention](#6-field-naming-convention)
7. [Complete Examples](#7-complete-examples)

---

## 1. Directory Structure

```
app.mars/
├── manifest.json
├── permissions.json
├── frontend/
│   └── index.html
└── backend/
    └── main.ts
```

### Package File Count Limit

The platform rejects any ZIP with **more than 2,000 files**. This is a hard limit enforced on upload.

**Correct data packaging:**

| Scenario | Pattern |
|----------|---------|
| Seed data (thousands of records, read once) | Single JSON array → seeded into `ctx.db` at first run |
| Static lookup table (< 2 MB, per-request) | Single merged JSON, loaded via `Deno.readTextFile` |
| Large static lookup (> 2 MB, per-request) | Single merged JSON + **module-level cache** (parse once per handler lifetime) |
| Per-user / per-session data | `ctx.db` (SQLite, managed by platform) |

**Never** split data into per-record individual files (e.g., `data/chars/一.json`, `data/chars/二.json`, ...):
```
❌ backend/data/chars/一.json       ← causes upload rejection and server slowdown
❌ backend/data/chars/二.json
   ... (thousands more)

✅ backend/data/chars.json          ← single merged file
```

---

## 2. manifest.json Schema

### Minimum Required Fields

```json
{
    "name": "my-app",
    "description": "一句话应用描述",
    "tags": ["工具", "效率"],
    "version": "1.0.0",
    "runtime": "mars@1.0",
    "entry": {
        "frontend": "frontend/index.html",
        "backend": "backend/main.ts"
    },
    "capabilities": ["db", "auth"]
}
```

### Field Constraints

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | YES | Unique app name, lowercase. Format: `^[\p{L}\p{N}][\p{L}\p{N}\-]{1,48}[\p{L}\p{N}]$` (min 3 chars, must start and end with letter/number, only hyphens allowed in middle) |
| `description` | string | YES | Brief human-readable summary of the app's purpose (Chinese preferred) |
| `tags` | string[] | YES | Keyword tags for search and categorization, maximum 5 items |
| `version` | string | YES | Semantic version (e.g., `"1.0.0"`) |
| `runtime` | string | YES | Always `"mars@1.0"` for v1.0 |
| `entry.frontend` | string | YES | Path to frontend HTML entry |
| `entry.backend` | string | NO | Path to backend TS entry; omit for pure frontend apps |
| `entry.admin` | string | NO | Path to admin frontend HTML entry (e.g. `"admin/index.html"`). When present, the platform serves a dedicated management UI at `/a/:appId/__admin/`, accessible only by the app author. The admin page uses the same MARS SDK PostMessageTransport as the regular frontend; the backend receives `ctx.isAdmin = true` when called from the admin interface. |
| `capabilities` | string[] | YES | List of required platform capabilities |

**Critical rules:**
- MUST NOT include `app_id` — platform assigns this on first upload
- `entry.backend` MAY be omitted for pure-frontend apps
- `capabilities` should follow principle of least privilege

### Available Capabilities

**Core capabilities** (available to all app types):
- `db` — Database read/write
- `storage` — Blob/file storage
- `auth` — User authentication
- `http` — Outbound HTTP requests (requires network permission with allowHosts)

**Media capabilities** (require corresponding permissions):
- `media` — Album read/write

**Connectivity capabilities**:
- `bluetooth` — Bluetooth device connection
- `nfc` — NFC tag reading

**System capabilities**:
- `clipboard` — Clipboard read/write
- `filesystem` — Public directory access
- `calendar` — Calendar event read/write

**Deprecated capabilities** (accepted by validator but should not be used in new apps):
- `gpu`, `sensor`, `gamepad`, `audio`, `haptics`, `asset_cache` — were for game engines; now managed natively by Cocos Creator / Unity
- `calendar` — Calendar event read/write

---

## 3. RPC Protocol

### Request Format

```json
{
    "requestId": "2b6fbf95-d9b8-4d07-b9d6-9f76c47e8f7b",
    "method": "todo.create",
    "params": { "text": "buy milk" }
}
```

- `requestId`: UUID, MUST be unique per request
- `method`: `<namespace>.<action>` format
- `params`: optional, arbitrary JSON payload

### Success Response

```json
{
    "requestId": "2b6fbf95-d9b8-4d07-b9d6-9f76c47e8f7b",
    "data": { "id": "t_123" },
    "error": null
}
```

- `data` MUST be non-null, `error` MUST be null

### Error Response

```json
{
    "requestId": "2b6fbf95-d9b8-4d07-b9d6-9f76c47e8f7b",
    "data": null,
    "error": {
        "code": "PERMISSION_DENIED",
        "message": "Permission not granted",
        "details": { "scope": "location.precise.read" }
    }
}
```

- `data` MUST be null, `error` MUST contain `code`, `message`, `details`

---

## 4. Error Model

### Error Codes

| Code | When to Use |
|------|-------------|
| `INVALID_PARAMS` | Missing or malformed request parameters |
| `UNAUTHORIZED` | No authenticated user |
| `PERMISSION_DENIED` | Permission scope not granted |
| `NOT_FOUND` | Resource or RPC method not found |
| `TIMEOUT` | Operation exceeded time limit |
| `RESOURCE_EXHAUSTED` | Quota exceeded (e.g., asset cache full) |
| `INTERNAL_ERROR` | Unexpected server error |

### TypeScript Error Types

```typescript
interface MarsError {
    code: MarsErrorCode;
    message: string;
    details: Record<string, unknown>;
}

class MarsException extends Error {
    readonly name: "MarsException";
    readonly code: MarsErrorCode;
    readonly details: Record<string, unknown>;
    constructor(error: MarsError);
}
```

---

## 5. Runtime Capability Boundaries

The runtime MUST enforce:
- No direct host filesystem access
- No subprocess spawning
- No undeclared/unauthorized external network access
- All capabilities exposed only through controlled `MarsContext` injection
- Permission check on every capability invocation
- Immediate revocation enforcement when user revokes consent

---

## 6. Field Naming Convention

| Context | Convention | Examples |
|---------|-----------|----------|
| RPC envelope fields | camelCase | `requestId`, `method`, `params`, `data`, `error` |
| TypeScript SDK interfaces | camelCase | `requestId`, `userId`, `createdAt` |
| Audit/governance HTTP API | snake_case | `app_id`, `user_id` |
| Permission scopes | dot-separated lowercase | `user.profile.read` |

Cross-language payloads: SDK/Runtime SHOULD perform `snake_case ↔ camelCase` mapping at protocol boundaries.

---

## 7. Complete Examples

### Example: Todo App manifest.json

```json
{
    "name": "todo-app",
    "description": "简洁的待办事项管理应用",
    "tags": ["工具", "效率", "待办"],
    "version": "1.0.0",
    "runtime": "mars@1.0",
    "entry": {
        "frontend": "frontend/index.html",
        "backend": "backend/main.ts"
    },
    "capabilities": ["db", "auth"]
}
```

# MARS Permissions Reference (Cocos Creator Game-Type)

Permission scopes and declaration rules for MARS v1.0 **Cocos Creator game-type** mini-apps.
For app-type permissions, see the `mars-app-generator` skill.

## 1. permissions.json Schema

```json
{
    "name": "<game-name>",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "<domain>.<target>.<action>",
            "purpose": "中文说明此权限用途",
            "consentRequired": true,
            "dataRetentionDays": 30
        }
    ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | YES | Must match manifest.json `name` |
| `version` | string | YES | Must match manifest.json `version` |
| `permissions[].scope` | string | YES | Three-segment scope identifier |
| `permissions[].purpose` | string | YES | Human-readable purpose (Chinese) |
| `permissions[].consentRequired` | boolean | YES | Whether user consent popup is needed |
| `permissions[].dataRetentionDays` | number | NO | Data retention period in days (documentation-only; not persisted by current platform validator) |

---

## 2. Scope Naming Convention

Format: `<domain>.<target>.<action>`

- `domain` and `action` MUST be lowercase
- Wildcard scopes are FORBIDDEN
- Each scope maps to a specific capability invocation

---

## 3. Common Game Permission Scopes

### User Information

| Scope | consentRequired | Use Case |
|-------|----------------|----------|
| `user.id.read` | **false** | Player identity for leaderboard/saves |
| `user.profile.read` | **true** | Show player name/avatar in leaderboard |
| `user.email.read` | **true** | Account binding (rare for games) |

### Resource Scopes

| Scope | consentRequired | Use Case |
|-------|----------------|----------|
| `db.<table>.read` | **false** | Read leaderboard/save data |
| `db.<table>.write` | **false** | Write scores/save data |
| `storage.<bucket>.read` | **false** | Read game save files (binary) |
| `storage.<bucket>.write` | **false** | Write game save files (binary) |

### Network Scopes

| Scope | consentRequired | Use Case |
|-------|----------------|----------|
| `network.<alias>.connect` | **false** | Connect to game servers, matchmaking |

---

## 4. What NOT to Include

Game-type apps should **NOT** request the following scopes — Cocos Creator handles these natively:

- ~~`gpu.webgl.access`~~ / ~~`gpu.native.access`~~ — Cocos Creator manages rendering
- ~~`audio.highperf.access`~~ — Cocos Creator manages audio
- ~~`sensor.gyroscope.read`~~ / ~~`sensor.accelerometer.read`~~ — Cocos Creator accesses sensors
- ~~`input.gamepad.read`~~ — Cocos Creator manages input
- ~~`cache.assets.read`~~ / ~~`cache.assets.write`~~ — Cocos Creator manages asset caching
- ~~`device.haptics.vibrate`~~ — Cocos Creator manages haptics

Also NOT typically needed for games:
- `media.album.read` / `media.album.write` — App-only
- `bluetooth.device.connect` — App-only
- `nfc.tag.read` — App-only
- `clipboard.content.read` / `clipboard.content.write` — App-only
- `filesystem.public.read` / `filesystem.public.write` — App-only
- `calendar.event.read` / `calendar.event.write` — App-only

---

## 5. Compliance Rules

1. **Principle of least privilege** — only request permissions the game actually needs
2. **Purpose clarity** — every scope must have a clear, honest Chinese-language purpose
3. **Minimal data** — games typically only need user ID and maybe profile
4. **No silent collection** — never fetch user data without visible UI indication

---

## 6. Complete Example

### Space Shooter Game

```json
{
    "name": "space-shooter",
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

### Multiplayer RPG with Cloud Saves

```json
{
    "name": "dragon-quest",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "user.id.read",
            "purpose": "用于玩家身份识别与匹配",
            "consentRequired": false
        },
        {
            "scope": "user.profile.read",
            "purpose": "用于显示队友昵称与头像",
            "consentRequired": true
        },
        {
            "scope": "db.characters.read",
            "purpose": "读取角色数据",
            "consentRequired": false
        },
        {
            "scope": "db.characters.write",
            "purpose": "保存角色进度",
            "consentRequired": false
        },
        {
            "scope": "storage.saves.read",
            "purpose": "读取云端存档",
            "consentRequired": false
        },
        {
            "scope": "storage.saves.write",
            "purpose": "上传云端存档",
            "consentRequired": false
        }
    ]
}
```

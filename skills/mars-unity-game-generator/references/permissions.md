# MARS Permissions — Unity Game-Type Mini-App

This reference covers the permission model for Unity game-type mini-apps.
Game-type apps only request **platform service** permissions; rendering, audio, physics,
sensors, haptics are managed natively by Unity and do not require MARS permissions.

## 1. Permission Scope Format

```
<domain>.<resource>.<action>
```

Examples:
- `user.id.read` — Read player's unique ID
- `db.scores.write` — Write to the `scores` table
- `storage.replays.read` — Read from the `replays` storage bucket

## 2. Game-Relevant Permission Domains

### User Scopes (Identity & Profile)

| Scope | Action | consentRequired | Notes |
|-------|--------|----------------|-------|
| `user.id.read` | Read player UID | false | Almost always needed for games |
| `user.profile.read` | Read name/avatar | true | For leaderboard display |

### DB Scopes (Leaderboards, Saves, Matchmaking)

| Scope | Action | consentRequired | Notes |
|-------|--------|----------------|-------|
| `db.<table>.read` | Read rows from table | false | e.g., `db.scores.read` |
| `db.<table>.write` | Write/update rows | false | e.g., `db.scores.write` |

Common tables:
- `scores` — Leaderboard entries
- `saves` — Cloud save data
- `matches` — Matchmaking state
- `inventory` — In-game items

### Storage Scopes (Game Save Files, Replays)

| Scope | Action | consentRequired | Notes |
|-------|--------|----------------|-------|
| `storage.<bucket>.read` | Read objects | false | e.g., `storage.saves.read` |
| `storage.<bucket>.write` | Write objects | false | e.g., `storage.saves.write` |

### Network Scopes (External API calls)

| Scope | Action | consentRequired | Notes |
|-------|--------|----------------|-------|
| `network.http.invoke` | Outbound HTTP | false | For external game APIs |

## 3. What NOT to Include

Unity game-type apps must NOT request permissions for:
- `media.*` — Unity handles audio/video natively
- `sensor.*` — Unity handles accelerometer, gyroscope, etc.
- `haptics.*` — Unity handles vibration natively
- `bluetooth.*` — App-type only
- `nfc.*` — App-type only
- `clipboard.*` — App-type only
- `filesystem.*` — App-type only
- `calendar.*` — App-type only
- `gpu.*` — Unity manages GPU natively

## 4. Permission JSON Structure

```json
{
    "name": "<game-name>",
    "version": "<semver>",
    "permissions": [
        {
            "scope": "<domain>.<resource>.<action>",
            "purpose": "<Chinese description of why this permission is needed>",
            "consentRequired": false
        }
    ]
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | Must match manifest.json `name` |
| `version` | string | Yes | Must match manifest.json `version` |
| `permissions[].scope` | string | Yes | Dot-separated scope path |
| `permissions[].purpose` | string | Yes | Chinese description for user consent |
| `permissions[].consentRequired` | boolean | Yes | `true` if explicit user consent needed |

## 5. Example: Tower Defense Game

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

## 6. Example: Multiplayer Racing Game

```json
{
    "name": "racing-legends",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "user.id.read",
            "purpose": "用于多人对战的玩家身份识别",
            "consentRequired": false
        },
        {
            "scope": "user.profile.read",
            "purpose": "用于排行榜和比赛大厅展示玩家信息",
            "consentRequired": true
        },
        {
            "scope": "db.scores.read",
            "purpose": "读取竞速排行榜数据",
            "consentRequired": false
        },
        {
            "scope": "db.scores.write",
            "purpose": "提交赛道成绩到排行榜",
            "consentRequired": false
        },
        {
            "scope": "db.matches.read",
            "purpose": "读取多人匹配房间状态",
            "consentRequired": false
        },
        {
            "scope": "db.matches.write",
            "purpose": "创建和管理多人匹配房间",
            "consentRequired": false
        },
        {
            "scope": "storage.replays.write",
            "purpose": "保存比赛回放数据",
            "consentRequired": false
        },
        {
            "scope": "storage.replays.read",
            "purpose": "读取比赛回放数据",
            "consentRequired": false
        }
    ]
}
```

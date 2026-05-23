# MARS Permissions Reference (App-Type)

Complete permission scope catalog and declaration rules for MARS v1.0 **app-type** mini-apps.
For game-type permission details, see the `mars-cocos-game-generator` or `mars-unity-game-generator` skills.

## Table of Contents
1. [permissions.json Schema](#1-permissionsjson-schema)
2. [Scope Naming Convention](#2-scope-naming-convention)
3. [User Information Scopes](#3-user-information-scopes)
4. [Geolocation Scopes](#4-geolocation-scopes)
5. [Media Device Scopes](#5-media-device-scopes)
6. [Connectivity Scopes](#6-connectivity-scopes)
7. [System Interaction Scopes](#7-system-interaction-scopes)
8. [Resource Scopes](#8-resource-scopes)
9. [Game Engine Scopes](#9-game-engine-scopes)
10. [Network Scopes](#10-network-scopes)
11. [Compliance Rules](#11-compliance-rules)
12. [Complete Examples](#12-complete-examples)

---

## 1. permissions.json Schema

```json
{
    "name": "<app-name>",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "<domain>.<target>.<action>",
            "purpose": "中文说明此权限用途",
            "consentRequired": true,
            "dataRetentionDays": 30,
            "constraints": {}
        }
    ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | YES | Must match manifest.json `name` |
| `version` | string | YES | Must match manifest.json `version` |
| `permissions` | array | YES | List of permission declarations |
| `permissions[].scope` | string | YES | Three-segment scope identifier |
| `permissions[].purpose` | string | YES | Human-readable purpose (Chinese) |
| `permissions[].consentRequired` | boolean | YES | Whether user consent popup is needed |
| `permissions[].dataRetentionDays` | number | NO | Data retention period in days. **Note**: currently stored in the zip but not persisted by the platform validator; treat as documentation-only until the platform adds support. |
| `permissions[].constraints` | object | NO | Additional constraints (e.g., allowHosts) |

---

## 2. Scope Naming Convention

Format: `<domain>.<target>.<action>`

- `domain` and `action` MUST be lowercase
- `target` SHOULD be human-readable
- Wildcard scopes are FORBIDDEN
- Each scope maps to a specific capability invocation

---

## 3. User Information Scopes

| Scope | consentRequired | Description |
|-------|----------------|-------------|
| `user.id.read` | **false** | 用户唯一标识读取 — 账户绑定、身份识别 |
| `user.profile.read` | **true** | 昵称、头像等公开资料 — 个人主页、会员注册 |
| `user.email.read` | **true** | 邮箱读取（敏感） — 账户绑定、订阅通知 |
| `user.phone.read` | **true** | 手机号读取（敏感） — 一键登录、身份验证 |

Compliance notes:
- Nickname/avatar: user must actively choose; no silent fetching
- Phone: requires button click + secondary confirmation; privacy policy must state usage
- Email/phone are sensitive — consent popup MUST state data usage and retention

---

## 4. Geolocation Scopes

| Scope | consentRequired | Description |
|-------|----------------|-------------|
| `location.precise.read` | **true** | 高精度 GPS 定位 — 导航、外卖、运动轨迹 |
| `location.coarse.read` | **true** | 粗略市区级定位 — 附近推荐、天气查询 |

Rules:
- Prefer `location.coarse.read` when city-level accuracy suffices
- `location.precise.read` requires strong justification during app review
- Background usage needs `allowBackground: true` with business justification

---

## 5. Media Device Scopes

| Scope | consentRequired | Description |
|-------|----------------|-------------|
| `media.album.read` | **true** | 相册读取 — 上传头像、发布动态 |
| `media.album.write` | **true** | 相册写入 — 保存海报、保存图片 |
| `camera.stream.capture` | **true** | 摄像头访问 — 扫码、人脸识别 |
| `microphone.audio.record` | **true** | 麦克风录音 — 语音消息、实时通话 |

Notes:
- Camera with face biometrics: must declare in `purpose`
- Microphone: UI must show recording indicator while active

---

## 6. Connectivity Scopes

| Scope | consentRequired | Description |
|-------|----------------|-------------|
| `bluetooth.device.connect` | **true** | 蓝牙设备连接 — 手环、打印机 |
| `nfc.tag.read` | **true** | NFC 标签读取 — 门禁卡、公交卡 |

Constraints:
- Bluetooth MUST include `constraints.serviceUUIDs` array
- NFC requires specific interaction type justification

---

## 7. System Interaction Scopes

| Scope | consentRequired | Description |
|-------|----------------|-------------|
| `clipboard.content.read` | **true** | 剪切板读取 — 粘贴口令、识别邀请码 |
| `clipboard.content.write` | **false** | 剪切板写入 — 复制邀请码、分享链接 |
| `filesystem.public.read` | **true** | 用户公共目录读取 — 预览文档 |
| `filesystem.public.write` | **true** | 用户公共目录写入 — 下载合同 |
| `calendar.event.read` | **true** | 日历事件读取 — 课程表同步 |
| `calendar.event.write` | **true** | 日历事件写入 — 添加会议提醒 |

**DEPRECATED:**
- `contacts.list.read` — NEVER use in new apps. Runtime returns `PERMISSION_DENIED` with `reason: "scope_unavailable"`.

---

## 8. Resource Scopes

Format: `<resource>.<name>.<action>`

- `db.<table>.read` / `db.<table>.write` — Database table access
- `storage.<bucketAlias>.read` / `storage.<bucketAlias>.write` — Blob storage
- `network.<alias>.connect` — External network access

Resource permissions must be minimized to specific tables, buckets, or network aliases.

---

## 9. ~~Game Engine Scopes~~ (Moved)

Game engine scopes are documented in the `mars-cocos-game-generator` and `mars-unity-game-generator` skills.
App-type mini-apps should NOT use game engine scopes.

---

## 10. Network Scopes

Format: `network.<alias>.connect`

```json
{
    "scope": "network.partnerApi.connect",
    "purpose": "调用合作方 API 获取商品数据",
    "consentRequired": false,
    "constraints": {
        "allowHosts": ["api.partner.com", "cdn.partner.com"]
    }
}
```

Rules:
- `constraints.allowHosts` MUST be a non-empty array
- No wildcard hosts (e.g., `"*"` or `"*.example.com"` are FORBIDDEN)
- Each host must be a specific domain

---

## 11. Compliance Rules

1. **Principle of least privilege** — only request permissions the app actually needs
2. **Purpose clarity** — every scope must have a clear, honest Chinese-language purpose
3. **Sensitivity awareness** — email, phone, precise location, camera, microphone are high-sensitivity
4. **Background usage** — add `allowBackground: true` only when genuinely needed, with justification
5. **Data retention** — declare `dataRetentionDays` for personal data
6. **No silent collection** — never fetch user data without visible UI indication

---

## 12. Complete Examples

### Standard Web App

```json
{
    "name": "todo-app",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "user.id.read",
            "purpose": "用于账户绑定与身份识别",
            "consentRequired": false
        },
        {
            "scope": "user.profile.read",
            "purpose": "用于展示用户昵称与头像",
            "consentRequired": true,
            "dataRetentionDays": 30
        }
    ]
}
```

### App with Network Permission

```json
{
    "name": "shop-app",
    "version": "1.0.0",
    "permissions": [
        {
            "scope": "user.id.read",
            "purpose": "用于用户身份识别",
            "consentRequired": false
        },
        {
            "scope": "network.productApi.connect",
            "purpose": "调用商品数据接口获取商品信息",
            "consentRequired": false,
            "constraints": {
                "allowHosts": ["api.shop-partner.com"]
            }
        }
    ]
}
```

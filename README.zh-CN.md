# Shapp Skills

**Language / 语言:** [English](README.md) | [中文](README.zh-CN.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

本仓库收录了一组用于生成 **Shapp**（Mini-App Runtime System，微应用运行时系统）合规包的 [GitHub Copilot Skills](https://code.visualstudio.com/docs/copilot/copilot-customization#_reusable-prompt-files-experimental)，涵盖应用类微应用与基于 Cocos Creator 或 Unity 引擎的游戏类微应用。

---

## 什么是 Skill？

Skill 是可复用的提示词文件（`.md`），用于将领域知识、开发规范与约束条件编码给 GitHub Copilot Agent。启用某个 Skill 后，Copilot 会自动遵循其中嵌入的指令，生成符合平台协议的代码——无需每次重新描述平台规则。

---

## 可用 Skill

| Skill | 描述 | 适用场景 |
|-------|------|----------|
| [`mars-app-generator`](skills/mars-app-generator/SKILL.md) | 生成 Shapp **应用类**微应用包（WebView 前端 + TypeScript 后端） | 构建工具类、电商、社交等非游戏类微应用 |
| [`mars-cocos-game-generator`](skills/mars-cocos-game-generator/SKILL.md) | 生成使用 **Cocos Creator** 引擎的 Shapp **游戏类**微应用包 | 使用 Cocos Creator 引擎开发游戏微应用 |
| [`mars-unity-game-generator`](skills/mars-unity-game-generator/SKILL.md) | 生成使用 **Unity** 引擎的 Shapp **游戏类**微应用包 | 使用 Unity 引擎开发游戏微应用 |

---

## 仓库结构

```
skills/
├── mars-app-generator/
│   ├── SKILL.md                  # Skill 定义与生成规则
│   └── references/
│       ├── protocol.md           # 包结构、manifest 格式、RPC 协议
│       ├── permissions.md        # 权限范围、命名规范、授权规则
│       └── sdk-api.md            # SDK 类型、上下文能力、Handler 模式
├── mars-cocos-game-generator/
│   ├── SKILL.md
│   └── references/
│       ├── protocol.md
│       ├── permissions.md
│       └── sdk-api.md
└── mars-unity-game-generator/
    ├── SKILL.md
    └── references/
        ├── protocol.md
        ├── permissions.md
        └── sdk-api.md
```

---

## 快速上手

### 1. 安装 Skill

将 `skills/` 目录（或所需的单个 Skill 子目录）复制到你的 VS Code 工作区中。

### 2. 在 Copilot Chat 中引用 Skill

在 Copilot Chat 对话框或 `.github/copilot-instructions.md` 文件中使用 `#` 文件引用语法：

```
#skills/mars-app-generator/SKILL.md 创建一个带用户认证的待办事项微应用。
```

或在 `.vscode/settings.json` 中进行全局配置：

```jsonc
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "skills/mars-app-generator/SKILL.md" }
  ]
}
```

### 3. 开始生成

描述你想构建的内容，Copilot 会自动遵循 Skill 中编码的 Shapp 协议规范、SDK 约束与打包限制。

---

## Skill 结构说明

每个 Skill 包含以下组成部分：

- **YAML Front Matter** — 声明 Skill 的 `name` 与 `description`，供 Copilot 自动识别和选用。
- **SDK 兼容性约束** — 列出允许和禁止的导入路径，防止模型使用不存在的 API。
- **包约束** — 强制执行平台限制（例如 ZIP 包最多 2,000 个文件）。
- **Reference 文件** — 包含 Skill 所依赖的协议、权限和 SDK 的详细规格说明。

---

## 参与贡献

1. Fork 本仓库。
2. 在 `skills/<skill-name>/` 下创建或更新 Skill。
3. 保持 `SKILL.md` 自包含——模型所需的所有规则必须存在于该文件或其 `references/` 子目录中。
4. 当 Shapp 协议或 SDK 发生变更时，同步更新 reference 文档。
5. 提交 Pull Request，并清晰描述改动内容与原因。

---

## 许可证

MIT

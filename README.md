# MARS Skills

**Language / 语言:** [English](README.md) | [中文](README.zh-CN.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

A collection of [GitHub Copilot Skills](https://code.visualstudio.com/docs/copilot/copilot-customization#_reusable-prompt-files-experimental) for generating **MARS** (Mini-App Runtime System) compliant packages — including app-type mini-apps and game-type mini-apps powered by Cocos Creator or Unity.

---

## What Are Skills?

Skills are reusable prompt files (`.md`) that encode domain knowledge, conventions, and guardrails for GitHub Copilot Agent. When a skill is active, Copilot follows its embedded instructions to produce consistent, protocol-compliant output — without you having to re-explain the platform rules every time.

---

## Available Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [`mars-app-generator`](skills/mars-app-generator/SKILL.md) | Generates MARS **app-type** mini-app packages (WebView frontend + TypeScript backend) | Building utilities, tools, e-commerce, social features, or any non-game mini-app |
| [`mars-cocos-game-generator`](skills/mars-cocos-game-generator/SKILL.md) | Generates MARS **game-type** mini-app packages using **Cocos Creator** | Building a game mini-app with the Cocos Creator engine |
| [`mars-unity-game-generator`](skills/mars-unity-game-generator/SKILL.md) | Generates MARS **game-type** mini-app packages using **Unity** | Building a game mini-app with the Unity engine |

---

## Repository Structure

```
skills/
├── mars-app-generator/
│   ├── SKILL.md                  # Skill definition & generation rules
│   └── references/
│       ├── protocol.md           # Package structure, manifest schema, RPC protocol
│       ├── permissions.md        # Permission scopes, naming conventions, consent rules
│       └── sdk-api.md            # SDK types, context capabilities, handler patterns
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

## Getting Started

### 1. Install the Skills

Copy the `skills/` folder (or the individual skill sub-folder you need) into your VS Code workspace.

### 2. Reference a Skill in Copilot Chat

Use the `#` file reference syntax in Copilot Chat or in a `.github/copilot-instructions.md` file:

```
#skills/mars-app-generator/SKILL.md Create a todo-list mini-app with user authentication.
```

Or configure it globally in `.vscode/settings.json`:

```jsonc
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "skills/mars-app-generator/SKILL.md" }
  ]
}
```

### 3. Start Generating

Describe what you want to build. Copilot will follow the MARS protocol rules, SDK guardrails, and packaging constraints encoded in the skill automatically.

---

## Skill Anatomy

Each skill follows this structure:

- **YAML Front Matter** — declares the skill `name` and `description` so Copilot can auto-select it.
- **SDK Compatibility Guardrails** — lists allowed/forbidden import paths to prevent hallucinated APIs.
- **Package Constraints** — enforces platform limits (e.g., 2,000-file ZIP limit).
- **Reference Files** — detailed, versioned specs for the protocol, permissions, and SDK that the skill embeds.

---

## Contributing

1. Fork this repository.
2. Create or update a skill under `skills/<skill-name>/`.
3. Keep `SKILL.md` self-contained — all rules the model needs must be present in the file or in its `references/` sub-folder.
4. Update the reference docs when the MARS protocol or SDK changes.
5. Open a pull request with a clear description of what changed and why.

---

## License

MIT

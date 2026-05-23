# MARS Skills

**Language / 言語:** [English](README.md) | [中文](README.zh-CN.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

**MARS**（Mini-App Runtime System）に準拠したパッケージを生成するための [GitHub Copilot Skills](https://code.visualstudio.com/docs/copilot/copilot-customization#_reusable-prompt-files-experimental) コレクションです。アプリ型ミニアプリ、および Cocos Creator または Unity を利用したゲーム型ミニアプリの生成をサポートします。

---

## Skill とは？

Skill はドメイン知識・開発規約・制約条件を GitHub Copilot Agent に伝えるための再利用可能なプロンプトファイル（`.md`）です。Skill が有効になっていると、Copilot はファイル内の指示に従い、プラットフォームのルールを毎回説明しなくても、一貫してプロトコルに準拠したコードを生成します。

---

## 利用可能な Skill

| Skill | 説明 | 利用場面 |
|-------|------|----------|
| [`mars-app-generator`](skills/mars-app-generator/SKILL.md) | MARS **アプリ型**ミニアプリパッケージを生成（WebView フロントエンド + TypeScript バックエンド） | ツール・EC・SNS など非ゲームのミニアプリを構築する場合 |
| [`mars-cocos-game-generator`](skills/mars-cocos-game-generator/SKILL.md) | **Cocos Creator** を使用した MARS **ゲーム型**ミニアプリパッケージを生成 | Cocos Creator エンジンでゲームミニアプリを開発する場合 |
| [`mars-unity-game-generator`](skills/mars-unity-game-generator/SKILL.md) | **Unity** を使用した MARS **ゲーム型**ミニアプリパッケージを生成 | Unity エンジンでゲームミニアプリを開発する場合 |

---

## リポジトリ構成

```
skills/
├── mars-app-generator/
│   ├── SKILL.md                  # Skill 定義と生成ルール
│   └── references/
│       ├── protocol.md           # パッケージ構造・マニフェストスキーマ・RPC プロトコル
│       ├── permissions.md        # 権限スコープ・命名規則・同意ルール
│       └── sdk-api.md            # SDK 型・コンテキスト機能・ハンドラーパターン
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

## クイックスタート

### 1. Skill をインストールする

`skills/` ディレクトリ（または必要な Skill サブフォルダー）を VS Code ワークスペースにコピーします。

### 2. Copilot Chat で Skill を参照する

Copilot Chat または `.github/copilot-instructions.md` で `#` ファイル参照構文を使います。

```
#skills/mars-app-generator/SKILL.md ユーザー認証付きの ToDo ミニアプリを作成してください。
```

または `.vscode/settings.json` でグローバルに設定します。

```jsonc
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "skills/mars-app-generator/SKILL.md" }
  ]
}
```

### 3. 生成を開始する

作りたいものを説明するだけで、Copilot が Skill に記載された MARS プロトコル規則・SDK 制約・パッケージ制限を自動的に適用します。

---

## Skill の構成要素

各 Skill は以下の要素で構成されています。

- **YAML フロントマター** — Copilot が自動選択できるよう `name` と `description` を宣言します。
- **SDK 互換性ガードレール** — 存在しない API の使用を防ぐため、許可・禁止されたインポートパスを明示します。
- **パッケージ制約** — プラットフォーム制限（例: ZIP 最大 2,000 ファイル）を強制します。
- **Reference ファイル** — プロトコル・権限・SDK の詳細仕様を格納します。

---

## コントリビューション

1. このリポジトリを Fork します。
2. `skills/<skill-name>/` 配下に Skill を作成または更新します。
3. `SKILL.md` を自己完結させてください — モデルが必要とするルールはすべてそのファイルまたは `references/` に含める必要があります。
4. MARS プロトコルや SDK が変更された場合は、reference ドキュメントも更新してください。
5. 変更内容と理由を明記した Pull Request を送ってください。

---

## ライセンス

MIT

# Shapp Skills

**Language / 언어:** [English](README.md) | [中文](README.zh-CN.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

**Shapp**（Mini-App Runtime System）에 준거하는 패키지를 생성하기 위한 [GitHub Copilot Skills](https://code.visualstudio.com/docs/copilot/copilot-customization#_reusable-prompt-files-experimental) 모음입니다. 앱형 미니앱과 Cocos Creator 또는 Unity 기반의 게임형 미니앱 생성을 지원합니다.

---

## Skill이란?

Skill은 도메인 지식, 개발 규약, 제약 조건을 GitHub Copilot Agent에 전달하는 재사용 가능한 프롬프트 파일(`.md`)입니다. Skill이 활성화되면 Copilot은 파일에 내장된 지침에 따라, 플랫폼 규칙을 매번 다시 설명하지 않아도 일관되고 프로토콜을 준수하는 코드를 생성합니다.

---

## 제공 Skill 목록

| Skill | 설명 | 사용 시점 |
|-------|------|-----------|
| [`mars-app-generator`](skills/mars-app-generator/SKILL.md) | Shapp **앱형** 미니앱 패키지 생성 (WebView 프론트엔드 + TypeScript 백엔드) | 도구, 전자상거래, 소셜 등 비게임 미니앱을 개발할 때 |
| [`mars-cocos-game-generator`](skills/mars-cocos-game-generator/SKILL.md) | **Cocos Creator**를 사용하는 Shapp **게임형** 미니앱 패키지 생성 | Cocos Creator 엔진으로 게임 미니앱을 개발할 때 |
| [`mars-unity-game-generator`](skills/mars-unity-game-generator/SKILL.md) | **Unity**를 사용하는 Shapp **게임형** 미니앱 패키지 생성 | Unity 엔진으로 게임 미니앱을 개발할 때 |

---

## 저장소 구조

```
skills/
├── mars-app-generator/
│   ├── SKILL.md                  # Skill 정의 및 생성 규칙
│   └── references/
│       ├── protocol.md           # 패키지 구조, 매니페스트 스키마, RPC 프로토콜
│       ├── permissions.md        # 권한 범위, 네이밍 규칙, 동의 규칙
│       └── sdk-api.md            # SDK 타입, 컨텍스트 기능, 핸들러 패턴
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

## 빠른 시작

### 1. Skill 설치

`skills/` 디렉터리(또는 필요한 개별 Skill 하위 폴더)를 VS Code 워크스페이스에 복사합니다.

### 2. Copilot Chat에서 Skill 참조하기

Copilot Chat 또는 `.github/copilot-instructions.md` 파일에서 `#` 파일 참조 구문을 사용합니다.

```
#skills/mars-app-generator/SKILL.md 사용자 인증이 포함된 할 일 목록 미니앱을 만들어 주세요.
```

또는 `.vscode/settings.json`에서 전역으로 설정합니다.

```jsonc
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "skills/mars-app-generator/SKILL.md" }
  ]
}
```

### 3. 생성 시작

만들고 싶은 내용을 설명하면 Copilot이 Skill에 기록된 Shapp 프로토콜 규칙, SDK 제약 조건, 패키징 제한을 자동으로 적용합니다.

---

## Skill 구성 요소

각 Skill은 다음 요소로 구성됩니다.

- **YAML 프론트매터** — Copilot이 자동 선택할 수 있도록 `name`과 `description`을 선언합니다.
- **SDK 호환성 가드레일** — 존재하지 않는 API 사용을 방지하기 위해 허용·금지된 임포트 경로를 명시합니다.
- **패키지 제약 조건** — 플랫폼 제한(예: ZIP 최대 2,000개 파일)을 강제합니다.
- **Reference 파일** — 프로토콜, 권한, SDK에 대한 상세 명세를 포함합니다.

---

## 기여하기

1. 이 저장소를 Fork합니다.
2. `skills/<skill-name>/` 아래에 Skill을 생성하거나 수정합니다.
3. `SKILL.md`를 자기완결적으로 유지하세요 — 모델이 필요로 하는 모든 규칙은 해당 파일 또는 `references/` 하위 폴더에 있어야 합니다.
4. Shapp 프로토콜이나 SDK가 변경되면 reference 문서도 함께 업데이트하세요.
5. 변경 내용과 이유를 명확히 작성한 Pull Request를 제출하세요.

---

## 라이선스

MIT

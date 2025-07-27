# Gemini CLI Package.json 구조 분석

## 📋 개요

Gemini CLI는 **NPM Workspaces 기반 모노레포** 구조로 설계된 프로젝트입니다. 루트 패키지와 3개의 서브 패키지로 구성되어 있으며, 각각 명확한 역할 분담을 통해 모듈화된 아키텍처를 구현합니다.

## 🏗️ 모노레포 구조

```
@google/gemini-cli (루트)
├── packages/cli/                  # 사용자 인터페이스
│   └── @google/gemini-cli
├── packages/core/                 # 비즈니스 로직
│   └── @google/gemini-cli-core
└── packages/vscode-ide-companion/ # VS Code 확장
    └── gemini-cli-vscode-ide-companion
```

## 📦 패키지별 상세 분석

### 1. **루트 패키지 (@google/gemini-cli)**

**역할**: 전체 프로젝트 통합 관리 및 배포

**주요 특징:**
```json
{
  "workspaces": ["packages/*"],
  "private": "true",
  "type": "module",
  "engines": { "node": ">=20.0.0" }
}
```

**핵심 스크립트:**
```bash
# 개발 및 빌드
npm start                    # 개발 모드 실행
npm run build               # 전체 빌드
npm run build:all           # 빌드 + 샌드박스 + VS Code
npm run bundle              # 배포용 번들링

# 테스트
npm test                    # 전체 테스트
npm run test:e2e            # 엔드투엔드 테스트
npm run test:integration:*  # 샌드박스별 통합 테스트

# 품질 관리
npm run lint               # ESLint 검사
npm run format             # Prettier 포맷팅
npm run typecheck          # TypeScript 타입 검사
npm run preflight          # 배포 전 전체 검증
```

**샌드박스 설정:**
```json
{
  "config": {
    "sandboxImageUri": "us-docker.pkg.dev/gemini-code-dev/gemini-cli/sandbox:0.1.13"
  }
}
```

### 2. **CLI 패키지 (@google/gemini-cli)**

**역할**: React + Ink 기반 터미널 UI

**주요 특징:**
```json
{
  "main": "dist/index.js",
  "bin": { "gemini": "dist/index.js" },
  "type": "module"
}
```

**핵심 의존성:**
```json
{
  "dependencies": {
    "@google/gemini-cli-core": "file:../core",  // 로컬 의존성
    "react": "^19.1.0",                         // UI 프레임워크
    "ink": "^6.0.1",                           // 터미널 React 렌더러
    "yargs": "^17.7.2",                        // CLI 인수 파싱
    "dotenv": "^17.1.0",                       // 환경 변수
    "highlight.js": "^11.11.1",                // 코드 하이라이팅
    "lowlight": "^3.3.0"                       // 구문 강조
  }
}
```

**개발 도구:**
- **Testing**: `vitest`, `ink-testing-library`, `@testing-library/react`
- **TypeScript**: 전체 타입 지원
- **React DevTools**: 개발 시 UI 디버깅

### 3. **Core 패키지 (@google/gemini-cli-core)**

**역할**: AI 통신, 도구 실행, 비즈니스 로직

**핵심 의존성:**
```json
{
  "dependencies": {
    "@google/genai": "1.9.0",                        // Google AI SDK
    "@modelcontextprotocol/sdk": "^1.11.0",          // MCP 프로토콜
    "@opentelemetry/*": "^0.52.0",                   // 텔레메트리
    "google-auth-library": "^9.11.0",               // Google 인증
    "simple-git": "^3.28.0",                        // Git 통합
    "micromatch": "^4.0.8",                         // 파일 패턴 매칭
    "html-to-text": "^9.0.5",                       // HTML 파싱
    "ws": "^8.18.0",                                // WebSocket
    "undici": "^7.10.0"                             // HTTP 클라이언트
  }
}
```

**주요 기능:**
- **AI 통신**: Gemini API 클라이언트
- **도구 시스템**: Shell, File, Edit 도구들
- **MCP 서버**: Model Context Protocol 지원
- **보안**: 샌드박스 통합
- **텔레메트리**: OpenTelemetry 기반 모니터링

### 4. **VS Code 확장 (gemini-cli-vscode-ide-companion)**

**역할**: VS Code 통합 및 IDE 컴패니언

**주요 특징:**
```json
{
  "engines": { "vscode": "^1.101.0" },
  "categories": ["AI"],
  "activationEvents": ["onStartupFinished"],
  "preview": true
}
```

**핵심 의존성:**
```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.15.1",  // MCP 통합
    "express": "^5.1.0",                      // HTTP 서버
    "cors": "^2.8.5",                         // CORS 처리
    "zod": "^3.25.76"                         // 스키마 검증
  }
}
```

## 🔄 의존성 관계 다이어그램

```
┌─────────────────────────────────────┐
│      루트 (@google/gemini-cli)      │
│         워크스페이스 관리자           │
└─────────────┬───────────────────────┘
              │
    ┌─────────▼─────────┬─────────────────┐
    │                   │                 │
┌───▼────────┐ ┌────────▼──────┐ ┌────────▼──────────┐
│CLI Package │ │ Core Package  │ │ VS Code Extension │
│(UI Layer)  │ │(Logic Layer)  │ │ (IDE Integration) │
│            │ │               │ │                   │
│React + Ink │ │Gemini API +   │ │Express Server +   │
│Terminal UI │ │Tools + MCP    │ │MCP SDK           │
└────────────┘ └───────────────┘ └───────────────────┘
       │               ▲
       └───────────────┘
         file:../core
```

## ⚙️ 개발 워크플로

### **루트 중심 개발**
```bash
# 프로젝트 루트에서 모든 작업 수행
npm start                    # 개발 서버 시작
npm run build --workspaces   # 모든 패키지 빌드
npm test --workspaces        # 모든 패키지 테스트
```

### **패키지별 개발**
```bash
# 특정 패키지에서 작업 (선택적)
cd packages/cli
npm run build               # CLI만 빌드
npm test                    # CLI만 테스트
```

### **통합 테스트**
```bash
# 샌드박스 환경별 테스트
npm run test:integration:sandbox:none    # 샌드박스 없이
npm run test:integration:sandbox:docker  # Docker 샌드박스
npm run test:integration:sandbox:podman  # Podman 샌드박스
```

## 🚀 빌드 및 배포

### **빌드 프로세스**
```bash
1. npm run generate          # Git 커밋 정보 생성
2. npm run build            # TypeScript → JavaScript
3. node esbuild.config.js   # 번들링 (ESBuild)
4. npm run copy_bundle_assets # 자산 복사
```

### **배포 구조**
```
bundle/
├── gemini.js              # 메인 실행 파일
├── sandbox-macos-*.sb     # macOS 샌드박스 프로파일
└── [기타 자산들]
```

## 📊 버전 관리

**동기화된 버전:**
- 모든 패키지: `0.1.13`
- 루트와 하위 패키지 버전 일치
- VS Code 확장: `99.99.99` (개발용)

**릴리스 스크립트:**
```bash
npm run release:version     # 버전 업데이트
npm run prepare:package     # 패키지 준비
npm run prepare            # 번들 생성 (npm publish 전 자동 실행)
```

## 🎯 범용 AI 에이전트 적용 포인트

### **1. 모노레포 구조 장점**
- **모듈 분리**: UI/Logic/Integration 계층 분리
- **코드 공유**: 공통 타입 및 유틸리티 재사용
- **통합 관리**: 버전, 빌드, 테스트 일원화
- **확장성**: 새로운 패키지 쉽게 추가 가능

### **2. 의존성 관리 전략**
- **로컬 의존성**: `file:../core` 패턴으로 개발 효율성
- **외부 의존성**: 각 계층별 최적화된 라이브러리 선택
- **버전 동기화**: 전체 프로젝트 일관성 유지

### **3. 개발 도구 통합**
- **TypeScript**: 전체 프로젝트 타입 안전성
- **ESLint/Prettier**: 코드 품질 자동화
- **Vitest**: 현대적 테스트 프레임워크
- **ESBuild**: 빠른 빌드 성능

### **4. 다중 AI 제공업체 지원 확장**
```json
{
  "workspaces": [
    "packages/cli",           // 공통 UI
    "packages/core",          // 공통 로직
    "packages/providers/*"    // AI 제공업체별 구현
  ]
}
```

**확장 예시:**
- `packages/providers/openai`
- `packages/providers/anthropic`
- `packages/providers/deepseek`
- `packages/providers/ollama`

## 🔮 개선 제안

### **모듈화 강화**
```
packages/
├── cli/              # UI 계층
├── core/             # 공통 로직
├── providers/        # AI 제공업체들
├── tools/            # 도구 시스템
├── sandbox/          # 샌드박스 관리
└── extensions/       # IDE 확장들
```

### **설정 관리 개선**
```json
{
  "config": {
    "providers": {
      "gemini": { "enabled": true, "apiKey": "env:GEMINI_API_KEY" },
      "openai": { "enabled": true, "apiKey": "env:OPENAI_API_KEY" },
      "deepseek": { "enabled": true, "apiKey": "env:DEEPSEEK_API_KEY" }
    },
    "costOptimization": {
      "primary": "deepseek",
      "fallback": ["gemini", "openai"],
      "budgetLimit": 10.0
    }
  }
}
```

---

**작성일**: 2025-01-27
**목적**: 모노레포 구조 이해 및 범용 AI 에이전트 아키텍처 설계 ✅

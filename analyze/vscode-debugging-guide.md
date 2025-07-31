# VS Code에서 Gemini CLI 디버깅 가이드

이 문서는 VS Code 환경에서 Gemini CLI 프로젝트를 효과적으로 디버깅하는 방법을 설명합니다.

## 프로젝트 구조 이해

Gemini CLI는 다음과 같은 구조를 가진 monorepo입니다:

```
gemini-cli/
├── packages/
│   ├── cli/           # 메인 CLI 애플리케이션
│   ├── core/          # 코어 라이브러리 ⭐ 주요 디버깅 대상
│   └── vscode-ide-companion/  # VS Code 확장
├── scripts/           # 빌드 및 유틸리티 스크립트
├── integration-tests/ # 통합 테스트
└── .vscode/          # VS Code 설정 파일들
```

### packages/core 라이브러리 구조

`packages/core`는 Gemini CLI의 핵심 기능을 담당하는 라이브러리로, 다음과 같은 구조를 가집니다:

```
packages/core/src/
├── core/              # 핵심 로직 (Gemini API 클라이언트, 채팅, 토큰 관리)
├── tools/             # 도구 구현체 (파일 시스템, 웹 검색, 메모리 등)
├── services/          # 서비스 레이어
├── config/            # 설정 관리
├── mcp/               # Model Context Protocol 구현
├── telemetry/         # 텔레메트리
├── prompts/           # 프롬프트 관리
├── ide/               # IDE 통합
└── utils/             # 유틸리티 함수들
```

#### 주요 컴포넌트:
- **core/client.ts**: Gemini API 클라이언트
- **core/geminiChat.ts**: 채팅 인터페이스
- **tools/**: 각종 도구들 (파일 읽기/쓰기, 셸 명령, 웹 검색 등)
- **mcp/**: MCP 서버 및 클라이언트 구현

## 사전 준비

### 1. 의존성 설치
```bash
npm install
```

### 2. 프로젝트 빌드
```bash
npm run build
```

### 3. Gemini API 키 설정 ⭐ 필수
디버깅을 위해서는 실제 Gemini API에 연결이 필요하므로 API 키 설정이 필수입니다.

#### API 키 발급:
1. [Google AI Studio](https://aistudio.google.com/app/apikey)에서 API 키 발급

#### API 키 설정 방법:

**방법 1: 환경 변수로 설정 (권장)**
```bash
# 현재 세션에서만 사용
export GEMINI_API_KEY="YOUR_GEMINI_API_KEY"

# 영구 설정 (.bashrc에 추가)
echo 'export GEMINI_API_KEY="YOUR_GEMINI_API_KEY"' >> ~/.bashrc
source ~/.bashrc
```

**방법 2: .env 파일 사용**
```bash
# 프로젝트 루트에 .gemini/.env 파일 생성
mkdir -p .gemini
echo 'GEMINI_API_KEY="YOUR_GEMINI_API_KEY"' >> .gemini/.env

# 또는 홈 디렉토리에 설정
mkdir -p ~/.gemini
echo 'GEMINI_API_KEY="YOUR_GEMINI_API_KEY"' >> ~/.gemini/.env
```

**방법 3: VS Code 디버깅 환경에서 설정**
`.vscode/launch.json`의 환경 변수에 직접 추가:
```json
{
  "env": {
    "GEMINI_SANDBOX": "false",
    "GEMINI_API_KEY": "YOUR_GEMINI_API_KEY"
  }
}
```

## 디버깅 설정

프로젝트에는 이미 다양한 디버깅 시나리오를 위한 VS Code launch 설정이 준비되어 있습니다.

### 사용 가능한 디버깅 구성

#### 1. **Launch CLI** - 메인 CLI 디버깅
- **용도**: CLI 애플리케이션의 기본 실행 및 디버깅
- **환경**: 샌드박스 비활성화 상태
- **실행**: `F5` 또는 디버그 패널에서 "Launch CLI" 선택

#### 2. **Launch E2E** - 통합 테스트 디버깅
- **용도**: 통합 테스트 실행 및 디버깅
- **특정 테스트**: `list_directory` 테스트가 기본으로 설정됨
- **출력**: verbose 모드로 실행되며 출력 유지

#### 3. **Launch Companion VS Code Extension** - VS Code 확장 디버깅
- **용도**: VS Code IDE 컴패니언 확장 개발 및 디버깅
- **자동 빌드**: 실행 전 자동으로 빌드됨
- **새 VS Code 인스턴스**: 확장이 설치된 새로운 VS Code 창이 열림

#### 4. **Attach** - 실행 중인 프로세스에 연결
- **용도**: 이미 실행 중인 Node.js 프로세스에 디버거 연결
- **포트**: 9229 (기본값)
- **샌드박스 지원**: 글로벌 설치된 환경에서도 소스 매핑 지원

#### 5. **Launch Program** - 개별 파일 실행
- **용도**: 현재 열린 파일을 직접 실행
- **사용법**: 실행하고 싶은 JavaScript/TypeScript 파일을 열고 실행

#### 6. **Debug Test File** - 개별 테스트 파일 디버깅
- **용도**: 특정 테스트 파일을 디버깅 모드로 실행
- **입력**: 실행 시 테스트 파일 경로를 입력받음
- **기본값**: LoadingIndicator 컴포넌트 테스트

## 디버깅 시나리오별 가이드

### 1. CLI 명령어 디버깅

**목적**: CLI 명령어 실행 과정에서 발생하는 문제를 디버깅

**단계**:
1. `Ctrl + Shift + D`로 디버그 패널 열기
2. "Launch CLI" 선택
3. `F5`로 디버깅 시작
4. CLI 프롬프트에서 명령어 입력 및 테스트
5. 브레이크포인트에서 변수 상태 확인

**팁**:
- `packages/cli/src/` 디렉토리의 파일들에 브레이크포인트 설정
- 통합 터미널에서 CLI와 상호작용 가능

### 2. Core 라이브러리 특정 기능 디버깅 ⭐

**목적**: packages/core의 핵심 기능들을 집중적으로 디버깅

#### 2.1 Gemini API 클라이언트 디버깅
**파일**: `packages/core/src/core/client.ts`, `packages/core/src/core/geminiChat.ts`

**단계**:
1. 해당 파일들에 브레이크포인트 설정
2. "Launch CLI" 실행
3. Gemini API 호출이 포함된 명령어 실행
4. 요청/응답 데이터 확인

**주요 체크포인트**:
- API 키 설정 상태
- 요청 파라미터 (모델명, 토큰 제한 등)
- 응답 처리 로직
- 에러 핸들링

#### 2.2 도구(Tools) 시스템 디버깅
**파일**: `packages/core/src/tools/` 디렉토리 내 파일들

**주요 도구들**:
- `read-file.ts`: 파일 읽기
- `write-file.ts`: 파일 쓰기
- `shell.ts`: 셸 명령 실행
- `web-search.ts`: 웹 검색
- `memoryTool.ts`: 메모리 관리
- `mcp-client.ts`: MCP 클라이언트

**디버깅 방법**:
```javascript
// 특정 도구 호출 시 브레이크포인트 설정
// 예: 파일 읽기 도구 디버깅
// packages/core/src/tools/read-file.ts 파일에서
export async function readFile(params: ReadFileParams): Promise<ReadFileResult> {
  // 여기에 브레이크포인트 설정
  console.log('Reading file:', params.path);
  // ...
}
```

#### 2.3 MCP (Model Context Protocol) 디버깅
**파일**: `packages/core/src/mcp/` 디렉토리

**단계**:
1. MCP 관련 파일에 브레이크포인트 설정
2. MCP 서버와의 통신이 필요한 명령어 실행
3. 프로토콜 메시지 교환 과정 추적

#### 2.4 프롬프트 및 콘텐츠 생성 디버깅
**파일**: `packages/core/src/core/contentGenerator.ts`, `packages/core/src/prompts/`

**단계**:
1. 프롬프트 생성 로직에 브레이크포인트 설정
2. 다양한 명령어로 프롬프트 변화 확인
3. 생성된 콘텐츠 품질 검증

### 3. 특정 기능 디버깅

**목적**: 파일 시스템, 메모리, 웹 검색 등 특정 기능 디버깅

**단계**:
1. 해당 기능 코드 파일 열기 (예: `packages/core/src/tools/`)
2. 관련 코드에 브레이크포인트 설정
3. "Launch CLI" 또는 "Launch E2E"로 디버깅 시작
4. 해당 기능을 트리거하는 명령어 실행

### 3. 통합 테스트 디버깅

**목적**: 특정 통합 테스트가 실패하는 원인 분석

**단계**:
1. "Launch E2E" 구성 선택
2. 필요시 `launch.json`에서 테스트 파일명 변경
3. `F5`로 디버깅 시작
4. 테스트 실행 과정 단계별 확인

**Core 라이브러리 관련 통합 테스트**:
- `integration-tests/file-system.test.js`: 파일 시스템 도구 테스트
- `integration-tests/save_memory.test.js`: 메모리 도구 테스트
- `integration-tests/run_shell_command.test.js`: 셸 명령 도구 테스트
- `integration-tests/google_web_search.test.js`: 웹 검색 도구 테스트

### 4. VS Code 확장 디버깅

**목적**: VS Code IDE 컴패니언 확장 개발 및 디버깅

**단계**:
1. "Launch Companion VS Code Extension" 선택
2. `F5`로 실행 (자동 빌드 후 새 VS Code 창 열림)
3. 새 창에서 확장 기능 테스트
4. 원본 창에서 디버깅 정보 확인

### 5. 개별 테스트 파일 디버깅

**목적**: 특정 단위 테스트 또는 컴포넌트 테스트 디버깅

**Core 라이브러리 테스트 파일들**:
- `packages/core/src/core/*.test.ts`: 핵심 로직 테스트
- `packages/core/src/tools/*.test.ts`: 개별 도구 테스트

**단계**:
1. "Debug Test File" 선택
2. `F5` 실행
3. 프롬프트에 테스트 파일 경로 입력 (예: `packages/core/src/tools/read-file.test.ts`)
4. 테스트 실행 및 디버깅

## Core 라이브러리 디버깅 베스트 프랙티스

### 1. 로깅 전략
```typescript
// packages/core/src/core/logger.ts 활용
import { logger } from '../core/logger.js';

export async function someFunction(params: any) {
  logger.debug('Function called with params:', params);

  try {
    const result = await processData(params);
    logger.info('Processing completed successfully');
    return result;
  } catch (error) {
    logger.error('Processing failed:', error);
    throw error;
  }
}
```

### 2. 도구별 디버깅 포인트

#### 파일 시스템 도구
```typescript
// packages/core/src/tools/read-file.ts
export async function readFile(params: ReadFileParams) {
  // 1. 파라미터 검증
  console.log('Reading file with params:', params);

  // 2. 파일 존재 여부 확인
  const exists = await fileExists(params.path);
  console.log('File exists:', exists);

  // 3. 파일 읽기 결과
  const content = await fs.readFile(params.path, 'utf-8');
  console.log('File content length:', content.length);

  return { content };
}
```

#### MCP 클라이언트 디버깅
```typescript
// packages/core/src/tools/mcp-client.ts
export class MCPClient {
  async callTool(name: string, params: any) {
    // 1. 요청 로깅
    console.log('MCP tool call:', { name, params });

    // 2. 응답 로깅
    const response = await this.transport.call(name, params);
    console.log('MCP response:', response);

    return response;
  }
}
```

### 3. 환경별 디버깅 설정

#### 개발 환경
```typescript
// DEBUG 환경변수로 상세 로깅 활성화
if (process.env.DEBUG) {
  logger.setLevel('debug');
}
```

#### 테스트 환경
```typescript
// 테스트에서 모킹된 의존성 확인
import { vi } from 'vitest';

beforeEach(() => {
  vi.clearAllMocks();
  console.log('Test setup completed');
});
```

## 유용한 디버깅 팁

### 1. 환경 변수 활용
```javascript
// DEBUG 모드에서 추가 로깅
if (process.env.DEBUG) {
  console.log('디버그 정보:', debugInfo);
}
```

### 2. 샌드박스 환경 제어
- `GEMINI_SANDBOX=false`: 샌드박스 비활성화로 직접 디버깅
- `GEMINI_SANDBOX=docker`: Docker 샌드박스 환경에서 테스트
- `GEMINI_SANDBOX=podman`: Podman 샌드박스 환경에서 테스트

### 3. 브레이크포인트 전략
- **조건부 브레이크포인트**: 특정 조건에서만 중단
- **로그포인트**: 코드 실행을 중단하지 않고 로그만 출력
- **히트 카운트**: 특정 횟수 실행 후 중단

### 4. 디버그 콘솔 활용
- 실행 중 변수 값 확인: `변수명`
- 표현식 평가: `계산식 또는 함수 호출`
- 객체 구조 탐색: `객체.속성` 또는 `객체['키']`

## 일반적인 문제 해결

### 1. 소스맵 문제
- TypeScript 파일에서 브레이크포인트가 동작하지 않는 경우
- 해결책: `npm run build` 실행 후 다시 디버깅 시작

### 2. 포트 충돌
- 다른 Node.js 프로세스가 9229 포트를 사용 중인 경우
- 해결책: `launch.json`에서 다른 포트 번호 사용

### 3. 샌드박스 환경 디버깅
- 샌드박스 내부에서 실행되는 코드 디버깅이 어려운 경우
- 해결책: `GEMINI_SANDBOX=false` 설정 후 디버깅

### 4. 의존성 관련 문제
- 모듈을 찾을 수 없는 경우
- 해결책: `npm install` 및 `npm run build` 재실행

## 고급 디버깅 기법

### 1. 멀티 프로세스 디버깅
- CLI가 자식 프로세스를 생성하는 경우
- `--inspect-brk` 플래그가 자식 프로세스에 전달되도록 설정

### 2. 원격 디버깅
- 샌드박스 환경 내부 프로세스 디버깅
- "Attach" 구성을 사용하여 원격 프로세스에 연결

### 3. 성능 프로파일링
- VS Code의 성능 프로파일러 사용
- CPU 사용량 및 메모리 사용 패턴 분석

## 개발 워크플로우 권장사항

### Core 라이브러리 개발 워크플로우

1. **코드 변경 후**: 항상 `npm run build` 실행 (특히 TypeScript 변경 시)
2. **새 도구 개발**:
   - `packages/core/src/tools/` 에 도구 구현
   - 해당 도구의 테스트 파일 작성
   - 통합 테스트로 전체 플로우 검증
3. **버그 수정**:
   - 재현 가능한 테스트 케이스 먼저 작성
   - Core 라이브러리의 해당 모듈에 브레이크포인트 설정
   - CLI를 통한 end-to-end 테스트
4. **통합 테스트**: 실제 사용 시나리오와 유사한 환경에서 테스트

### Core 라이브러리 특화 디버깅 체크리스트

#### Gemini API 관련
- [ ] API 키가 올바르게 설정되었는가?
- [ ] 모델 설정이 올바른가? (`client.ts` 확인)
- [ ] 토큰 제한이 적절하게 설정되었는가?
- [ ] 요청/응답 로그가 예상과 일치하는가?

#### 도구 시스템 관련
- [ ] 도구 등록이 올바르게 되었는가? (`tool-registry.ts` 확인)
- [ ] 도구 파라미터 검증이 올바른가?
- [ ] 도구 실행 결과가 예상과 일치하는가?
- [ ] 에러 핸들링이 적절한가?

#### MCP 관련
- [ ] MCP 서버 연결이 정상인가?
- [ ] 프로토콜 메시지가 올바르게 교환되는가?
- [ ] 도구 호출이 정상적으로 전달되는가?

#### 프롬프트 및 콘텐츠 생성
- [ ] 프롬프트가 올바르게 구성되는가?
- [ ] 컨텍스트가 적절하게 포함되는가?
- [ ] 생성된 콘텐츠가 요구사항을 만족하는가?

## 추가 자료

- **VS Code 디버깅 공식 문서**: https://code.visualstudio.com/docs/editor/debugging
- **Node.js 디버깅 가이드**: https://nodejs.org/en/docs/guides/debugging-getting-started/
- **프로젝트 특정 설정**: `.vscode/launch.json` 파일 참조

이 가이드를 통해 Gemini CLI 프로젝트의 다양한 컴포넌트를 효과적으로 디버깅할 수 있습니다. 각 디버깅 시나리오에 맞는 적절한 구성을 선택하여 개발 효율성을 높이세요.

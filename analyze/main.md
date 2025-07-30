# Gemini CLI Main 함수 분석

이 문서는 Gemini CLI의 진입점인 main 함수의 동작을 단계별로 분석하고, 실제 코드와 함께 설명합니다.

## 목차

1. [Main 함수 개요](#1-main-함수-개요)
2. [초기 설정 및 검증](#2-초기-설정-및-검증)
3. [메모리 관리 및 재시작](#3-메모리-관리-및-재시작)
4. [샌드박스 처리](#4-샌드박스-처리)
5. [UI 모드 결정](#5-ui-모드-결정)
6. [대화형 모드](#6-대화형-모드)
7. [비대화형 모드](#7-비대화형-모드)
8. [에러 처리](#8-에러-처리)
9. [전체 플로우 다이어그램](#9-전체-플로우-다이어그램)

## 1. Main 함수 개요

### 1.1 함수 시그니처 및 진입점

**파일:** `packages/cli/src/gemini.tsx`

```typescript
export async function main() {
  const workspaceRoot = process.cwd();
  const settings = loadSettings(workspaceRoot);

  // 체크포인트 정리
  await cleanupCheckpoints();

  // 설정 에러 확인
  if (settings.errors.length > 0) {
    for (const error of settings.errors) {
      let errorMessage = `Error in ${error.path}: ${error.message}`;
      if (!process.env.NO_COLOR) {
        errorMessage = `\x1b[31m${errorMessage}\x1b[0m`;
      }
      console.error(errorMessage);
      console.error(`Please fix ${error.path} and try again.`);
    }
    process.exit(1);
  }

  // ... 나머지 로직
}
```

### 1.2 주요 의존성 및 임포트

```typescript
import React from 'react';
import { render } from 'ink';                    // React 터미널 UI
import { AppWrapper } from './ui/App.js';        // 메인 UI 컴포넌트
import { loadCliConfig, parseArguments } from './config/config.js';
import { readStdin } from './utils/readStdin.js';
import { start_sandbox } from './utils/sandbox.js';
import { loadSettings } from './config/settings.js';
import { runNonInteractive } from './nonInteractiveCli.js';
import {
  Config,
  AuthType,
  sessionId,
  logUserPrompt
} from '@google/gemini-cli-core';
```

## 2. 초기 설정 및 검증

### 2.1 작업 디렉토리 및 설정 로드

```typescript
const workspaceRoot = process.cwd();
const settings = loadSettings(workspaceRoot);

// 체크포인트 정리 (이전 세션의 임시 파일들)
await cleanupCheckpoints();

// 설정 파일 에러 검증
if (settings.errors.length > 0) {
  for (const error of settings.errors) {
    let errorMessage = `Error in ${error.path}: ${error.message}`;
    if (!process.env.NO_COLOR) {
      errorMessage = `\x1b[31m${errorMessage}\x1b[0m`; // 빨간색 출력
    }
    console.error(errorMessage);
    console.error(`Please fix ${error.path} and try again.`);
  }
  process.exit(1);
}
```

### 2.2 명령행 인수 파싱 및 설정 로드

```typescript
const argv = await parseArguments();
const extensions = loadExtensions(workspaceRoot);
const config = await loadCliConfig(
  settings.merged,
  extensions,
  sessionId,
  argv,
);

// stdin 파이핑과 대화형 플래그 충돌 검사
if (argv.promptInteractive && !process.stdin.isTTY) {
  console.error(
    'Error: The --prompt-interactive flag is not supported when piping input from stdin.',
  );
  process.exit(1);
}

// 확장 기능 목록 출력 (--list-extensions)
if (config.getListExtensions()) {
  console.log('Installed extensions:');
  for (const extension of extensions) {
    console.log(`- ${extension.config.name}`);
  }
  process.exit(0);
}
```

### 2.3 기본 인증 방식 설정

```typescript
// Cloud Shell 환경에서 자동 인증 설정
if (!settings.merged.selectedAuthType) {
  if (process.env.CLOUD_SHELL === 'true') {
    settings.setValue(
      SettingScope.User,
      'selectedAuthType',
      AuthType.CLOUD_SHELL,
    );
  }
}

// 디버그 모드 설정
setMaxSizedBoxDebugging(config.getDebugMode());

// 설정 초기화
await config.initialize();
```

### 2.4 테마 설정 로드

```typescript
// 커스텀 테마 로드
themeManager.loadCustomThemes(settings.merged.customThemes);

if (settings.merged.theme) {
  if (!themeManager.setActiveTheme(settings.merged.theme)) {
    // 테마를 찾을 수 없는 경우 경고만 출력하고 계속
    console.warn(`Warning: Theme "${settings.merged.theme}" not found.`);
  }
}
```

## 3. 메모리 관리 및 재시작

### 3.1 Node.js 힙 메모리 계산

```typescript
function getNodeMemoryArgs(config: Config): string[] {
  const totalMemoryMB = os.totalmem() / (1024 * 1024);
  const heapStats = v8.getHeapStatistics();
  const currentMaxOldSpaceSizeMb = Math.floor(
    heapStats.heap_size_limit / 1024 / 1024,
  );

  // 전체 메모리의 50%를 타겟으로 설정
  const targetMaxOldSpaceSizeInMB = Math.floor(totalMemoryMB * 0.5);

  if (config.getDebugMode()) {
    console.debug(
      `Current heap size ${currentMaxOldSpaceSizeMb.toFixed(2)} MB`,
    );
  }

  // 재시작 방지 플래그 확인
  if (process.env.GEMINI_CLI_NO_RELAUNCH) {
    return [];
  }

  // 더 많은 메모리가 필요한 경우
  if (targetMaxOldSpaceSizeInMB > currentMaxOldSpaceSizeMb) {
    if (config.getDebugMode()) {
      console.debug(
        `Need to relaunch with more memory: ${targetMaxOldSpaceSizeInMB.toFixed(2)} MB`,
      );
    }
    return [`--max-old-space-size=${targetMaxOldSpaceSizeInMB}`];
  }

  return [];
}
```

### 3.2 프로세스 재시작 로직

```typescript
async function relaunchWithAdditionalArgs(additionalArgs: string[]) {
  const nodeArgs = [...additionalArgs, ...process.argv.slice(1)];
  const newEnv = { ...process.env, GEMINI_CLI_NO_RELAUNCH: 'true' };

  const child = spawn(process.execPath, nodeArgs, {
    stdio: 'inherit',
    env: newEnv,
  });

  await new Promise((resolve) => child.on('close', resolve));
  process.exit(0);
}
```

## 4. 샌드박스 처리

### 4.1 샌드박스 환경 확인 및 시작

```typescript
// 샌드박스 외부에서 실행 중인 경우
if (!process.env.SANDBOX) {
  const memoryArgs = settings.merged.autoConfigureMaxOldSpaceSize
    ? getNodeMemoryArgs(config)
    : [];
  const sandboxConfig = config.getSandbox();

  if (sandboxConfig) {
    // 샌드박스 시작 전 인증 검증
    if (settings.merged.selectedAuthType) {
      try {
        const err = validateAuthMethod(settings.merged.selectedAuthType);
        if (err) {
          throw new Error(err);
        }
        await config.refreshAuth(settings.merged.selectedAuthType);
      } catch (err) {
        console.error('Error authenticating:', err);
        process.exit(1);
      }
    }

    // 샌드박스 시작 (별도 프로세스)
    await start_sandbox(sandboxConfig, memoryArgs);
    process.exit(0);
  } else {
    // 샌드박스를 사용하지 않는 경우 메모리 설정만 적용
    if (memoryArgs.length > 0) {
      await relaunchWithAdditionalArgs(memoryArgs);
      process.exit(0);
    }
  }
}
```

### 4.2 OAuth 사전 처리

```typescript
// 브라우저 실행이 억제된 OAuth 인증의 경우 사전 처리
if (
  settings.merged.selectedAuthType === AuthType.LOGIN_WITH_GOOGLE &&
  config.isBrowserLaunchSuppressed()
) {
  // UI 렌더링 전에 OAuth 처리 (링크 복사 가능하도록)
  await getOauthClient(settings.merged.selectedAuthType, config);
}

// 실험적 ACP (Agent Communication Protocol) 모드
if (config.getExperimentalAcp()) {
  return runAcpPeer(config, settings);
}
```

## 5. UI 모드 결정

### 5.1 대화형/비대화형 모드 판단

```typescript
let input = config.getQuestion();
const startupWarnings = [
  ...(await getStartupWarnings()),
  ...(await getUserStartupWarnings(workspaceRoot)),
];

// 대화형 모드 조건:
// 1. --prompt-interactive 플래그가 설정됨
// 2. TTY 환경이고 초기 질문이 없음
const shouldBeInteractive =
  !!argv.promptInteractive || (process.stdin.isTTY && input?.length === 0);
```

### 5.2 입력 소스 결정

```typescript
// TTY가 아닌 경우 stdin에서 입력 읽기 (파이프된 입력)
if (!process.stdin.isTTY && !input) {
  input += await readStdin();
}

if (!input) {
  console.error('No input provided via stdin.');
  process.exit(1);
}
```

## 6. 대화형 모드

### 6.1 React UI 렌더링

```typescript
if (shouldBeInteractive) {
  const version = await getCliVersion();
  setWindowTitle(basename(workspaceRoot), settings);

  const instance = render(
    <React.StrictMode>
      <AppWrapper
        config={config}
        settings={settings}
        startupWarnings={startupWarnings}
        version={version}
      />
    </React.StrictMode>,
    { exitOnCtrlC: false }, // Ctrl+C 처리를 앱에서 직접 관리
  );

  // 정리 작업 등록
  registerCleanup(() => instance.unmount());
  return;
}
```

### 6.2 윈도우 타이틀 설정

```typescript
function setWindowTitle(title: string, settings: LoadedSettings) {
  if (!settings.merged.hideWindowTitle) {
    const windowTitle = (process.env.CLI_TITLE || `Gemini - ${title}`).replace(
      // 제어 문자 제거
      /[\x00-\x1F\x7F]/g,
      '',
    );

    // ANSI 이스케이프 시퀀스로 윈도우 타이틀 설정
    process.stdout.write(`\x1b]2;${windowTitle}\x07`);

    // 프로세스 종료 시 타이틀 초기화
    process.on('exit', () => {
      process.stdout.write(`\x1b]2;\x07`);
    });
  }
}
```

## 7. 비대화형 모드

### 7.1 사용자 프롬프트 로깅

```typescript
// 고유 프롬프트 ID 생성
const prompt_id = Math.random().toString(16).slice(2);

// 텔레메트리 이벤트 로깅
logUserPrompt(config, {
  'event.name': 'user_prompt',
  'event.timestamp': new Date().toISOString(),
  prompt: input,
  prompt_id,
  auth_type: config.getContentGeneratorConfig()?.authType,
  prompt_length: input.length,
});
```

### 7.2 비대화형 설정 구성

```typescript
async function loadNonInteractiveConfig(
  config: Config,
  extensions: Extension[],
  settings: LoadedSettings,
  argv: CliArgs,
) {
  let finalConfig = config;

  // YOLO 모드가 아닌 경우 위험한 도구들 제외
  if (config.getApprovalMode() !== ApprovalMode.YOLO) {
    const existingExcludeTools = settings.merged.excludeTools || [];
    const interactiveTools = [
      ShellTool.Name,    // 쉘 명령 실행
      EditTool.Name,     // 파일 편집
      WriteFileTool.Name,// 파일 작성
    ];

    const newExcludeTools = [
      ...new Set([...existingExcludeTools, ...interactiveTools]),
    ];

    const nonInteractiveSettings = {
      ...settings.merged,
      excludeTools: newExcludeTools,
    };

    finalConfig = await loadCliConfig(
      nonInteractiveSettings,
      extensions,
      config.getSessionId(),
      argv,
    );
    await finalConfig.initialize();
  }

  // 비대화형 인증 검증
  return await validateNonInteractiveAuth(
    settings.merged.selectedAuthType,
    finalConfig,
  );
}
```

### 7.3 비대화형 실행

```typescript
// 비대화형 모드 설정 로드
const nonInteractiveConfig = await loadNonInteractiveConfig(
  config,
  extensions,
  settings,
  argv,
);

// 비대화형 CLI 실행
await runNonInteractive(nonInteractiveConfig, input, prompt_id);
process.exit(0);
```

## 8. 에러 처리

### 8.1 전역 Promise Rejection 핸들러

```typescript
// 처리되지 않은 Promise rejection에 대한 전역 핸들러
process.on('unhandledRejection', (reason, _promise) => {
  console.error('=========================================');
  console.error('CRITICAL: Unhandled Promise Rejection!');
  console.error('=========================================');
  console.error('Reason:', reason);
  console.error('Stack trace may follow:');
  if (!(reason instanceof Error)) {
    console.error(reason);
  }
  // 심각한 에러의 경우 프로세스 종료
  process.exit(1);
});
```

### 8.2 정리 작업 등록

```typescript
import { cleanupCheckpoints, registerCleanup } from './utils/cleanup.js';

// 프로세스 종료 시 정리 작업
registerCleanup(() => {
  // UI 인스턴스 정리
  instance.unmount();

  // 임시 파일 정리
  cleanupCheckpoints();
});
```

## 9. 전체 플로우 다이어그램

```
main() 시작
    ↓
┌─────────────────────────┐
│ 1. 초기 설정            │
│ - 작업 디렉토리 확인     │
│ - 설정 파일 로드        │
│ - 체크포인트 정리       │
└─────────────────────────┘
    ↓
┌─────────────────────────┐
│ 2. 설정 검증 및 파싱     │
│ - 명령행 인수 파싱      │
│ - 확장 기능 로드        │
│ - Config 객체 생성      │
└─────────────────────────┘
    ↓
┌─────────────────────────┐
│ 3. 환경 설정            │
│ - 인증 방식 설정        │
│ - 테마 로드            │
│ - 디버그 모드 설정      │
└─────────────────────────┘
    ↓
┌─────────────────────────┐
│ 4. 메모리/샌드박스 처리  │
│ - 메모리 요구사항 확인   │
│ - 샌드박스 환경 확인    │
│ - 필요시 프로세스 재시작 │
└─────────────────────────┘
    ↓
┌─────────────────────────┐
│ 5. UI 모드 결정         │
│ - TTY 환경 확인         │
│ - 입력 소스 결정        │
│ - 대화형/비대화형 분기   │
└─────────────────────────┘
    ↓
┌─────────┐    ┌──────────┐
│대화형 모드│    │비대화형   │
│React UI │    │stdin 처리 │
│렌더링    │    │직접 실행  │
└─────────┘    └──────────┘
```

### 9.1 핵심 결정 포인트

1. **샌드박스 여부**: `!process.env.SANDBOX`
2. **대화형 여부**: `argv.promptInteractive || (process.stdin.isTTY && !input)`
3. **메모리 재시작**: `targetMemory > currentMemory && !GEMINI_CLI_NO_RELAUNCH`
4. **ACP 모드**: `config.getExperimentalAcp()`
5. **OAuth 브라우저**: `AuthType.LOGIN_WITH_GOOGLE && config.isBrowserLaunchSuppressed()`

### 9.2 프로세스 분기점

- **정상 종료**: `process.exit(0)`
- **에러 종료**: `process.exit(1)`
- **샌드박스 시작**: `start_sandbox()` → `process.exit(0)`
- **메모리 재시작**: `relaunchWithAdditionalArgs()` → `process.exit(0)`
- **대화형 모드**: React UI 렌더링 후 이벤트 루프 유지
- **비대화형 모드**: `runNonInteractive()` → `process.exit(0)`

### 9.3 설정 우선순위

1. **명령행 인수** (최고 우선순위)
2. **환경 변수**
3. **프로젝트 설정** (`.gemini/settings.json`)
4. **사용자 설정** (`~/.gemini/settings.json`)
5. **기본값** (최저 우선순위)

이 main 함수는 Gemini CLI의 모든 진입점을 관리하며, 다양한 실행 환경과 사용 사례에 대응할 수 있도록 복잡한 분기 로직을 포함하고 있습니다. 특히 메모리 관리, 샌드박스 처리, 그리고 대화형/비대화형 모드 전환이 핵심적인 기능입니다.

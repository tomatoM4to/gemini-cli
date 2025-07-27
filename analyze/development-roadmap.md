# 범용 AI 에이전트 개발 로드맵

## 📋 개요

Gemini CLI 분석을 통해 도출한 **범용 AI 에이전트** 개발 단계별 가이드입니다. 단일 AI 제공업체에서 시작하여 점진적으로 다중 제공업체, 비용 최적화, 고급 기능으로 확장하는 전략을 제시합니다.

## 🎯 개발 목표

- **다중 AI 제공업체 지원**: OpenAI, Anthropic, DeepSeek, Gemini, Ollama 등
- **비용 최적화**: 저비용 모델 우선 사용, 예산 관리
- **크로스 플랫폼**: Windows, macOS, Linux 지원
- **보안 우선**: 샌드박스 격리, 권한 제어
- **확장성**: 플러그인 시스템, 커스텀 도구

## 🚀 단계별 개발 계획

## Phase 1: 기초 인프라 구축 (1-2주)

### **1.1 프로젝트 초기화**
```bash
mkdir universal-ai-agent
cd universal-ai-agent
npm init -y
```

**모노레포 구조 설정:**
```json
{
  "name": "universal-ai-agent",
  "workspaces": ["packages/*"],
  "private": true,
  "type": "module",
  "engines": { "node": ">=20.0.0" }
}
```

**패키지 구조:**
```
packages/
├── core/          # 비즈니스 로직
├── cli/           # 터미널 인터페이스
└── providers/     # AI 제공업체들
    ├── openai/
    ├── anthropic/
    └── deepseek/
```

### **1.2 Core 패키지 기본 구조**
```typescript
// packages/core/src/index.ts
export interface AIProvider {
  name: string;
  sendMessage(prompt: string): Promise<string>;
  streamMessage(prompt: string): AsyncGenerator<string>;
  getCost(): number;
}

export interface Config {
  providers: ProviderConfig[];
  costLimit: number;
  primaryProvider: string;
  fallbackOrder: string[];
}

export class AIAgent {
  constructor(config: Config) {}
  async processPrompt(prompt: string): Promise<string> {}
}
```

### **1.3 개발 도구 설정**
```json
{
  "devDependencies": {
    "typescript": "^5.3.3",
    "vitest": "^3.1.1",
    "eslint": "^9.24.0",
    "prettier": "^3.5.3",
    "esbuild": "^0.25.0"
  }
}
```

## Phase 2: 첫 번째 AI 제공업체 구현 (1주)

### **2.1 OpenAI Provider 구현**
```typescript
// packages/providers/openai/src/index.ts
import { AIProvider } from '@universal-ai-agent/core';

export class OpenAIProvider implements AIProvider {
  name = 'openai';

  constructor(private apiKey: string) {}

  async sendMessage(prompt: string): Promise<string> {
    // OpenAI API 호출
  }

  async *streamMessage(prompt: string): AsyncGenerator<string> {
    // 스트림 응답
  }

  getCost(): number {
    // 비용 계산
  }
}
```

### **2.2 기본 CLI 구현**
```typescript
// packages/cli/src/index.ts
import { AIAgent } from '@universal-ai-agent/core';
import { OpenAIProvider } from '@universal-ai-agent/openai';

const agent = new AIAgent({
  providers: [new OpenAIProvider(process.env.OPENAI_API_KEY)],
  primaryProvider: 'openai'
});

const response = await agent.processPrompt(process.argv[2]);
console.log(response);
```

### **2.3 테스트 작성**
```typescript
// packages/core/src/__tests__/agent.test.ts
import { describe, it, expect } from 'vitest';
import { AIAgent } from '../index.js';

describe('AIAgent', () => {
  it('should process basic prompt', async () => {
    // 테스트 코드
  });
});
```

## Phase 3: 다중 제공업체 지원 (2주)

### **3.1 Anthropic Provider**
```typescript
// packages/providers/anthropic/src/index.ts
export class AnthropicProvider implements AIProvider {
  name = 'anthropic';
  // Claude API 구현
}
```

### **3.2 DeepSeek Provider**
```typescript
// packages/providers/deepseek/src/index.ts
export class DeepSeekProvider implements AIProvider {
  name = 'deepseek';
  // DeepSeek API 구현 (비용 최적화 우선)
}
```

### **3.3 Provider Registry 시스템**
```typescript
// packages/core/src/registry.ts
export class ProviderRegistry {
  private providers = new Map<string, AIProvider>();

  register(provider: AIProvider) {
    this.providers.set(provider.name, provider);
  }

  get(name: string): AIProvider | undefined {
    return this.providers.get(name);
  }

  getCheapest(): AIProvider {
    // 가장 저렴한 제공업체 반환
  }
}
```

## Phase 4: 비용 최적화 시스템 (1-2주)

### **4.1 비용 추적**
```typescript
// packages/core/src/cost-tracker.ts
export class CostTracker {
  private usage = new Map<string, number>();

  trackUsage(provider: string, cost: number) {
    const current = this.usage.get(provider) || 0;
    this.usage.set(provider, current + cost);
  }

  getTotalCost(): number {
    return Array.from(this.usage.values()).reduce((a, b) => a + b, 0);
  }

  isWithinBudget(limit: number): boolean {
    return this.getTotalCost() < limit;
  }
}
```

### **4.2 스마트 라우팅**
```typescript
// packages/core/src/router.ts
export class SmartRouter {
  selectProvider(prompt: string, config: Config): AIProvider {
    // 1. 예산 확인
    if (!this.isWithinBudget()) {
      return this.getCheapestProvider();
    }

    // 2. 작업 복잡도 분석
    const complexity = this.analyzeComplexity(prompt);

    // 3. 최적 제공업체 선택
    if (complexity === 'simple') {
      return this.getProvider('deepseek'); // 저비용
    } else {
      return this.getProvider(config.primaryProvider);
    }
  }
}
```

## Phase 5: 도구 시스템 구현 (2-3주)

### **5.1 기본 도구들**
```typescript
// packages/core/src/tools/shell.ts
export class ShellTool implements Tool {
  name = 'shell';

  async execute(command: string): Promise<string> {
    // 명령어 실행 (샌드박스 내)
  }
}

// packages/core/src/tools/file.ts
export class FileTool implements Tool {
  name = 'file';

  async read(path: string): Promise<string> {}
  async write(path: string, content: string): Promise<void> {}
}
```

### **5.2 도구 레지스트리**
```typescript
// packages/core/src/tool-registry.ts
export class ToolRegistry {
  private tools = new Map<string, Tool>();

  register(tool: Tool) {
    this.tools.set(tool.name, tool);
  }

  async execute(name: string, params: any): Promise<any> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Tool ${name} not found`);

    return await tool.execute(params);
  }
}
```

## Phase 6: 샌드박스 보안 시스템 (2-3주)

### **6.1 기본 샌드박스**
```typescript
// packages/core/src/sandbox/index.ts
export class Sandbox {
  async execute(command: string): Promise<string> {
    // 플랫폼별 샌드박스 실행
    if (process.platform === 'darwin') {
      return this.executeMacOS(command);
    } else {
      return this.executeDocker(command);
    }
  }

  private async executeMacOS(command: string): Promise<string> {
    // sandbox-exec 사용
  }

  private async executeDocker(command: string): Promise<string> {
    // Docker 컨테이너 사용
  }
}
```

### **6.2 권한 관리**
```typescript
// packages/core/src/permissions.ts
export class PermissionManager {
  checkFileAccess(path: string): boolean {
    // 파일 접근 권한 확인
    return path.startsWith(process.cwd());
  }

  checkNetworkAccess(url: string): boolean {
    // 네트워크 접근 권한 확인
  }
}
```

## Phase 7: 고급 UI 개발 (2주)

### **7.1 React + Ink CLI**
```typescript
// packages/cli/src/ui/App.tsx
import React from 'react';
import { Box, Text } from 'ink';

export function App() {
  return (
    <Box flexDirection="column">
      <Text>Universal AI Agent</Text>
      {/* 대화형 인터페이스 */}
    </Box>
  );
}
```

### **7.2 스트림 처리**
```typescript
// packages/cli/src/hooks/useAIStream.ts
export function useAIStream(prompt: string) {
  const [response, setResponse] = useState('');

  useEffect(() => {
    const stream = agent.streamMessage(prompt);
    // 실시간 응답 표시
  }, [prompt]);

  return response;
}
```

## Phase 8: 설정 및 확장성 (1-2주)

### **8.1 설정 시스템**
```typescript
// packages/core/src/config.ts
export interface Config {
  providers: {
    openai: { enabled: boolean; apiKey: string; };
    anthropic: { enabled: boolean; apiKey: string; };
    deepseek: { enabled: boolean; apiKey: string; };
  };
  costOptimization: {
    primary: string;
    fallback: string[];
    budgetLimit: number;
  };
  sandbox: {
    enabled: boolean;
    profile: string;
  };
}
```

### **8.2 플러그인 시스템**
```typescript
// packages/core/src/plugins.ts
export interface Plugin {
  name: string;
  initialize(agent: AIAgent): void;
}

export class PluginManager {
  private plugins: Plugin[] = [];

  register(plugin: Plugin) {
    this.plugins.push(plugin);
  }
}
```

## Phase 9: 테스트 및 최적화 (1-2주)

### **9.1 통합 테스트**
```bash
# 다양한 시나리오 테스트
npm run test:integration
npm run test:e2e
npm run test:providers
```

### **9.2 성능 최적화**
- 응답 시간 최적화
- 메모리 사용량 개선
- 캐싱 시스템 구현

### **9.3 에러 처리**
```typescript
// packages/core/src/error-handler.ts
export class ErrorHandler {
  handleProviderError(error: Error, provider: string) {
    // 제공업체 오류 시 fallback 처리
  }

  handleCostLimitError(cost: number, limit: number) {
    // 예산 초과 시 처리
  }
}
```

## Phase 10: 배포 및 문서화 (1주)

### **10.1 빌드 시스템**
```json
{
  "scripts": {
    "build": "node scripts/build.js",
    "bundle": "node esbuild.config.js",
    "test": "npm run test --workspaces",
    "release": "npm run build && npm run bundle"
  }
}
```

### **10.2 문서화**
- API 문서
- 사용자 가이드
- 개발자 가이드
- 설정 예시

## 📊 우선순위 매트릭스

| Phase | 중요도 | 난이도 | 시간 | 의존성 |
|-------|--------|--------|------|--------|
| 1. 기초 인프라 | 🔴 높음 | 🟡 중간 | 1-2주 | 없음 |
| 2. 첫 Provider | 🔴 높음 | 🟢 낮음 | 1주 | Phase 1 |
| 3. 다중 Provider | 🔴 높음 | 🟡 중간 | 2주 | Phase 2 |
| 4. 비용 최적화 | 🟠 중간 | 🟡 중간 | 1-2주 | Phase 3 |
| 5. 도구 시스템 | 🟠 중간 | 🔴 높음 | 2-3주 | Phase 3 |
| 6. 샌드박스 | 🔴 높음 | 🔴 높음 | 2-3주 | Phase 5 |
| 7. 고급 UI | 🟡 낮음 | 🟡 중간 | 2주 | Phase 3 |
| 8. 확장성 | 🟠 중간 | 🟡 중간 | 1-2주 | Phase 7 |
| 9. 테스트 | 🔴 높음 | 🟡 중간 | 1-2주 | 모든 Phase |
| 10. 배포 | 🟠 중간 | 🟢 낮음 | 1주 | Phase 9 |

## 🎯 핵심 성공 요소

### **1. MVP 접근법**
- Phase 1-3으로 기본 동작하는 AI 에이전트 완성
- 빠른 피드백 루프로 개선

### **2. 비용 최적화 우선**
- DeepSeek 등 저비용 모델을 기본으로 설정
- 실시간 비용 모니터링

### **3. 보안 우선 설계**
- 샌드박스를 선택사항이 아닌 필수로 구현
- 권한 최소화 원칙

### **4. 확장성 고려**
- 모든 컴포넌트를 인터페이스 기반으로 설계
- 플러그인 시스템으로 확장 가능

## 🚧 주요 도전 과제

1. **API 통합 복잡성**: 각 제공업체별 다른 API 스펙
2. **비용 예측 정확도**: 실시간 사용량 추적의 어려움
3. **샌드박스 호환성**: 플랫폼별 다른 격리 방식
4. **성능 최적화**: 다중 제공업체 간 전환 지연
5. **에러 복구**: 제공업체 장애 시 graceful fallback

## 🔮 향후 확장 방향

1. **추가 제공업체**: Ollama (로컬), Cohere, Perplexity
2. **고급 기능**: 컨텍스트 관리, RAG 시스템
3. **IDE 통합**: VS Code, JetBrains, Neovim 플러그인
4. **웹 인터페이스**: React 기반 대시보드
5. **모바일 앱**: React Native 기반

---

**작성일**: 2025-01-27
**목적**: 범용 AI 에이전트 단계별 개발 가이드 ✅
**예상 총 개발 기간**: 3-4개월 (1인 개발 기준)

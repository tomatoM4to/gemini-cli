# Gemini CLI API 호출 과정 분석

이 문서는 Gemini CLI 프로젝트가 LLM API를 호출하는 전체 과정을 단계별로 분석하고, 실제 코드와 함께 설명합니다.

## 목차

1. [인증 및 초기화 단계](#1-인증-및-초기화-단계)
2. [클라이언트 초기화 단계](#2-클라이언트-초기화-단계)
3. [사용자 입력 처리](#3-사용자-입력-처리)
4. [API 요청 준비](#4-api-요청-준비)
5. [실제 API 호출](#5-실제-api-호출)
6. [응답 처리](#6-응답-처리)
7. [도구 호출 처리](#7-도구-호출-처리)
8. [텔레메트리 및 로깅](#8-텔레메트리-및-로깅)
9. [전체 플로우 요약](#9-전체-플로우-요약)

## 1. 인증 및 초기화 단계

### 1.1 인증 방식 선택 및 검증

**왜 필요한가?**
- **다양한 사용 환경 지원**: 개발자마다 다른 인증 환경 (개인 API 키, 회사 OAuth, 클라우드 환경)
- **보안 강화**: 각 환경에 맞는 최적의 보안 방식 제공
- **사용자 경험**: 복잡한 인증 설정 없이 바로 사용 가능
- **엔터프라이즈 지원**: 회사 정책에 맞는 인증 방식 선택 가능

Gemini CLI는 4가지 인증 방식을 지원합니다:

- `LOGIN_WITH_GOOGLE`: Google OAuth 로그인 (Gemini Code Assist)
- `USE_GEMINI`: Gemini API 키 사용
- `USE_VERTEX_AI`: Vertex AI API 키 사용
- `CLOUD_SHELL`: Cloud Shell 환경

**파일:** `packages/cli/src/config/auth.ts`

```typescript
export const validateAuthMethod = (authMethod: string): string | null => {
  loadEnvironment();
  if (
    authMethod === AuthType.LOGIN_WITH_GOOGLE ||
    authMethod === AuthType.CLOUD_SHELL
  ) {
    return null;
  }

  if (authMethod === AuthType.USE_GEMINI) {
    if (!process.env.GEMINI_API_KEY) {
      return 'GEMINI_API_KEY environment variable not found. Add that to your environment and try again (no reload needed if using .env)!';
    }
    return null;
  }

  if (authMethod === AuthType.USE_VERTEX_AI) {
    const hasVertexProjectLocationConfig =
      !!process.env.GOOGLE_CLOUD_PROJECT && !!process.env.GOOGLE_CLOUD_LOCATION;
    const hasGoogleApiKey = !!process.env.GOOGLE_API_KEY;
    if (!hasVertexProjectLocationConfig && !hasGoogleApiKey) {
      return (
        'When using Vertex AI, you must specify either:\n' +
        '• GOOGLE_CLOUD_PROJECT and GOOGLE_CLOUD_LOCATION environment variables.\n' +
        '• GOOGLE_API_KEY environment variable (if using express mode).\n' +
        'Update your environment and try again (no reload needed if using .env)!'
      );
    }
    return null;
  }

  return 'Invalid auth method selected.';
};
```

### 1.2 ContentGenerator 생성

**왜 필요한가?**
- **추상화 레이어**: 서로 다른 API 엔드포인트를 동일한 인터페이스로 통합
- **유연성**: 인증 방식 변경 시 상위 로직은 그대로 유지
- **확장성**: 새로운 LLM 서비스 추가 시 ContentGenerator 인터페이스만 구현하면 됨
- **설정 중앙화**: 모든 API 관련 설정을 한 곳에서 관리

인증 방식에 따라 적절한 ContentGenerator를 생성합니다.

**파일:** `packages/core/src/core/contentGenerator.ts`

```typescript
export function createContentGeneratorConfig(
  config: Config,
  authType: AuthType | undefined,
): ContentGeneratorConfig {
  const geminiApiKey = process.env.GEMINI_API_KEY || undefined;
  const googleApiKey = process.env.GOOGLE_API_KEY || undefined;
  const googleCloudProject = process.env.GOOGLE_CLOUD_PROJECT || undefined;
  const googleCloudLocation = process.env.GOOGLE_CLOUD_LOCATION || undefined;

  // Use runtime model from config if available; otherwise, fall back to parameter or default
  const effectiveModel = config.getModel() || DEFAULT_GEMINI_MODEL;

  const contentGeneratorConfig: ContentGeneratorConfig = {
    model: effectiveModel,
    authType,
    proxy: config?.getProxy(),
  };

  // OAuth 인증의 경우 추가 설정 불필요
  if (
    authType === AuthType.LOGIN_WITH_GOOGLE ||
    authType === AuthType.CLOUD_SHELL
  ) {
    return contentGeneratorConfig;
  }

  // Gemini API 키 설정
  if (authType === AuthType.USE_GEMINI && geminiApiKey) {
    contentGeneratorConfig.apiKey = geminiApiKey;
    contentGeneratorConfig.vertexai = false;
    getEffectiveModel(
      contentGeneratorConfig.apiKey,
      contentGeneratorConfig.model,
      contentGeneratorConfig.proxy,
    );
    return contentGeneratorConfig;
  }

  // Vertex AI 설정
  if (
    authType === AuthType.USE_VERTEX_AI &&
    (googleApiKey || (googleCloudProject && googleCloudLocation))
  ) {
    contentGeneratorConfig.apiKey = googleApiKey;
    contentGeneratorConfig.vertexai = true;
    return contentGeneratorConfig;
  }

  return contentGeneratorConfig;
}

export async function createContentGenerator(
  config: ContentGeneratorConfig,
  gcConfig: Config,
  sessionId?: string,
): Promise<ContentGenerator> {
  const version = process.env.CLI_VERSION || process.version;
  const httpOptions = {
    headers: {
      'User-Agent': `GeminiCLI/${version} (${process.platform}; ${process.arch})`,
    },
  };

  // OAuth 인증: CodeAssistServer 생성
  if (
    config.authType === AuthType.LOGIN_WITH_GOOGLE ||
    config.authType === AuthType.CLOUD_SHELL
  ) {
    return createCodeAssistContentGenerator(
      httpOptions,
      config.authType,
      gcConfig,
      sessionId,
    );
  }

  // API 키 인증: GoogleGenAI 인스턴스 생성
  if (
    config.authType === AuthType.USE_GEMINI ||
    config.authType === AuthType.USE_VERTEX_AI
  ) {
    const googleGenAI = new GoogleGenAI({
      apiKey: config.apiKey === '' ? undefined : config.apiKey,
      vertexai: config.vertexai,
      httpOptions,
    });

    return googleGenAI.models;
  }

  throw new Error(
    `Error creating contentGenerator: Unsupported authType: ${config.authType}`,
  );
}
```

### 1.3 OAuth 인증용 CodeAssist ContentGenerator

**왜 필요한가?**
- **엔터프라이즈 지원**: 회사에서 관리하는 OAuth 토큰으로 안전한 액세스
- **고급 기능**: Google Code Assist의 고급 기능 (더 큰 컨텍스트, Pro 모델 등)
- **자동 토큰 관리**: 토큰 갱신, 만료 처리 자동화
- **프로젝트 연동**: Google Cloud 프로젝트와 연결된 기능 사용

**파일:** `packages/core/src/code_assist/codeAssist.ts`

```typescript
export async function createCodeAssistContentGenerator(
  httpOptions: HttpOptions,
  authType: AuthType,
  config: Config,
  sessionId?: string,
): Promise<ContentGenerator> {
  if (
    authType === AuthType.LOGIN_WITH_GOOGLE ||
    authType === AuthType.CLOUD_SHELL
  ) {
    // OAuth 클라이언트 생성
    const authClient = await getOauthClient(authType, config);
    // 사용자 설정 및 프로젝트 정보 확인
    const userData = await setupUser(authClient);
    // CodeAssistServer 인스턴스 반환
    return new CodeAssistServer(
      authClient,
      userData.projectId,
      httpOptions,
      sessionId,
      userData.userTier,
    );
  }

  throw new Error(`Unsupported authType: ${authType}`);
}
```

## 2. 클라이언트 초기화 단계

### 2.1 GeminiClient 초기화

**왜 필요한가?**
- **상태 관리**: 대화 세션, 설정, 도구들의 중앙 관리
- **프록시 지원**: 기업 환경의 네트워크 제한 대응
- **세션 격리**: 여러 대화 세션 간 독립성 보장
- **루프 감지**: 무한 도구 호출 방지

**파일:** `packages/core/src/core/client.ts`

```typescript
export class GeminiClient {
  constructor(private config: Config) {
    // 프록시 설정
    if (config.getProxy()) {
      setGlobalDispatcher(new ProxyAgent(config.getProxy() as string));
    }

    this.embeddingModel = config.getEmbeddingModel();
    this.loopDetector = new LoopDetectionService(config);
  }

  async initialize(contentGeneratorConfig: ContentGeneratorConfig) {
    // ContentGenerator 생성
    this.contentGenerator = await createContentGenerator(
      contentGeneratorConfig,
      this.config,
      this.config.getSessionId(),
    );
    // Chat 세션 시작
    this.chat = await this.startChat();
  }

  getContentGenerator(): ContentGenerator {
    if (!this.contentGenerator) {
      throw new Error('Content generator not initialized');
    }
    return this.contentGenerator;
  }
}
```

### 2.2 Chat 세션 시작

**왜 필요한가?**
- **컨텍스트 연속성**: 대화 히스토리를 통한 문맥 유지
- **도구 등록**: 모델이 사용할 수 있는 함수들 사전 정의
- **시스템 프롬프트**: 모델의 동작 방식 및 역할 정의
- **Thinking 지원**: 지원하는 모델에서 사고 과정 표시

**파일:** `packages/core/src/core/client.ts`

```typescript
async startChat(extraHistory?: Content[]): Promise<GeminiChat> {
  const tools = await this.config.getTools();
  const history: Content[] = [
    {
      role: 'user',
      parts: [{ text: 'Hello' }],
    },
    {
      role: 'model',
      parts: [{ text: 'Got it. Thanks for the context!' }],
    },
    ...(extraHistory ?? []),
  ];

  try {
    const userMemory = this.config.getUserMemory();
    const systemInstruction = getCoreSystemPrompt(userMemory);

    // 모델에 따라 thinking 기능 지원 여부 확인
    const generateContentConfigWithThinking = isThinkingSupported(
      this.config.getModel(),
    )
      ? {
          ...this.generateContentConfig,
          thinkingConfig: {
            includeThoughts: true,
          },
        }
      : this.generateContentConfig;

    return new GeminiChat(
      this.config,
      this.getContentGenerator(),
      {
        systemInstruction,
        ...generateContentConfigWithThinking,
        tools,
      },
      history,
    );
  } catch (error) {
    await reportError(
      error,
      'Error initializing Gemini chat session.',
      history,
      'startChat',
    );
    throw new Error(`Failed to initialize chat: ${getErrorMessage(error)}`);
  }
}
```

## 3. 사용자 입력 처리

### 3.1 대화형 모드 시작

**왜 필요한가?**
- **사용자 경험**: TTY 환경에서 실시간 대화형 인터페이스 제공
- **유연한 실행**: 대화형/비대화형 모드 자동 감지
- **React UI**: 풍부한 UI 컴포넌트로 더 나은 사용자 경험
- **메모리 관리**: 장시간 실행 시 메모리 최적화

**파일:** `packages/cli/src/gemini.tsx`

```typescript
export async function main() {
  const workspaceRoot = process.cwd();
  const settings = loadSettings(workspaceRoot);
  const argv = await parseArguments();
  const extensions = loadExtensions(workspaceRoot);
  const config = await loadCliConfig(
    settings.merged,
    extensions,
    sessionId,
    argv,
  );

  // 설정에 따라 UI 렌더링 또는 비대화형 실행
  const shouldBeInteractive =
    !!argv.promptInteractive || (process.stdin.isTTY && input?.length === 0);

  if (shouldBeInteractive) {
    // React UI 렌더링
    const instance = render(
      <React.StrictMode>
        <AppWrapper
          config={config}
          settings={settings}
          startupWarnings={startupWarnings}
          version={version}
        />
      </React.StrictMode>,
      { exitOnCtrlC: false },
    );
  } else {
    // 비대화형 모드 실행
    await runNonInteractive(nonInteractiveConfig, input, prompt_id);
  }
}
```

### 3.2 비대화형 모드 실행

**왜 필요한가?**
- **스크립트 통합**: CI/CD 파이프라인, 배치 작업에서 사용
- **턴 제한**: 무한 루프 방지 및 비용 제어
- **스트리밍 출력**: 실시간 응답 표시로 사용자 경험 향상
- **자동 도구 실행**: 사용자 개입 없이 연속적인 작업 수행

**파일:** `packages/cli/src/nonInteractiveCli.ts`

```typescript
export async function runNonInteractive(
  config: Config,
  input: string,
  prompt_id: string,
): Promise<void> {
  await config.initialize();

  const geminiClient = config.getGeminiClient();
  const toolRegistry: ToolRegistry = await config.getToolRegistry();
  const chat = await geminiClient.getChat();

  let currentMessages: Content[] = [{ role: 'user', parts: [{ text: input }] }];
  let turnCount = 0;

  while (true) {
    turnCount++;
    if (
      config.getMaxSessionTurns() > 0 &&
      turnCount > config.getMaxSessionTurns()
    ) {
      console.error('\n Reached max session turns for this session.');
      return;
    }

    const functionCalls: FunctionCall[] = [];

    // 메시지 스트림 시작
    const responseStream = await chat.sendMessageStream(
      {
        message: currentMessages[0]?.parts || [],
        config: {
          abortSignal: abortController.signal,
          tools: [
            { functionDeclarations: toolRegistry.getFunctionDeclarations() },
          ],
        },
      },
      prompt_id,
    );

    // 스트리밍 응답 처리
    for await (const resp of responseStream) {
      const textPart = getResponseText(resp);
      if (textPart) {
        process.stdout.write(textPart);
      }
      if (resp.functionCalls) {
        functionCalls.push(...resp.functionCalls);
      }
    }

    // 도구 호출이 있으면 실행
    if (functionCalls.length > 0) {
      // 도구 실행 로직...
    } else {
      break; // 더 이상 도구 호출이 없으면 종료
    }
  }
}
```

## 4. API 요청 준비

### 4.1 메시지 전송 준비

**왜 필요한가?**
- **로깅**: 디버깅 및 사용 분석을 위한 API 요청 기록
- **모델 선택**: 동적 모델 변경 및 폴백 지원
- **할당량 관리**: 사용량 초과 시 적절한 모델로 전환
- **재시도 준비**: 실패 시 자동 재시도를 위한 설정

**파일:** `packages/core/src/core/geminiChat.ts`

```typescript
async sendMessage(
  params: SendMessageParameters,
  prompt_id: string,
): Promise<GenerateContentResponse> {
  await this.sendPromise;

  // 사용자 컨텐츠 생성
  const userContent = createUserContent(params.message);
  // 기존 히스토리와 새 입력 결합
  const requestContents = this.getHistory(true).concat(userContent);

  // API 요청 로깅
  this._logApiRequest(requestContents, this.config.getModel(), prompt_id);

  const startTime = Date.now();
  let response: GenerateContentResponse;

  try {
    const apiCall = () => {
      const modelToUse = this.config.getModel() || DEFAULT_GEMINI_FLASH_MODEL;

      // 할당량 오류 후 Flash 모델 호출 방지
      if (
        this.config.getQuotaErrorOccurred() &&
        modelToUse === DEFAULT_GEMINI_FLASH_MODEL
      ) {
        throw new Error(
          'Please submit a new query to continue with the Flash model.',
        );
      }

      return this.contentGenerator.generateContent({
        model: modelToUse,
        contents: requestContents,
        config: { ...this.generationConfig, ...params.config },
      });
    };

    // 재시도 로직과 함께 API 호출
    response = await retryWithBackoff(apiCall, {
      shouldRetry: (error: Error) => {
        if (error && error.message) {
          if (error.message.includes('429')) return true;
          if (error.message.match(/5\d{2}/)) return true;
        }
        return false;
      },
      onPersistent429: async (authType?: string, error?: unknown) =>
        await this.handleFlashFallback(authType, error),
      authType: this.config.getContentGeneratorConfig()?.authType,
    });

    // 응답 로깅
    const durationMs = Date.now() - startTime;
    await this._logApiResponse(
      durationMs,
      prompt_id,
      response.usageMetadata,
      JSON.stringify(response),
    );

    return response;
  } catch (error) {
    const durationMs = Date.now() - startTime;
    this._logApiError(durationMs, error, prompt_id);
    throw error;
  }
}
```

## 5. 실제 API 호출

### 5.1 OAuth 방식 - CodeAssistServer

**왜 필요한가?**
- **스트리밍 지원**: SSE(Server-Sent Events)로 실시간 응답 표시
- **인증 헤더**: OAuth 토큰을 통한 안전한 API 액세스
- **에러 파싱**: 스트리밍 환경에서의 정확한 에러 처리
- **프로젝트 연동**: Google Cloud 프로젝트별 설정 및 권한 관리

**파일:** `packages/core/src/code_assist/server.ts`

```typescript
export class CodeAssistServer implements ContentGenerator {
  // 일반 요청
  async generateContent(
    req: GenerateContentParameters,
  ): Promise<GenerateContentResponse> {
    const resp = await this.requestPost<CaGenerateContentResponse>(
      'generateContent',
      toGenerateContentRequest(req, this.projectId, this.sessionId),
      req.config?.abortSignal,
    );
    return fromGenerateContentResponse(resp);
  }

  // 스트리밍 요청
  async generateContentStream(
    req: GenerateContentParameters,
  ): Promise<AsyncGenerator<GenerateContentResponse>> {
    const resps = await this.requestStreamingPost<CaGenerateContentResponse>(
      'streamGenerateContent',
      toGenerateContentRequest(req, this.projectId, this.sessionId),
      req.config?.abortSignal,
    );
    return (async function* (): AsyncGenerator<GenerateContentResponse> {
      for await (const resp of resps) {
        yield fromGenerateContentResponse(resp);
      }
    })();
  }

  // HTTP POST 요청
  async requestPost<T>(
    method: string,
    req: object,
    signal?: AbortSignal,
  ): Promise<T> {
    const res = await this.client.request({
      url: this.getMethodUrl(method),
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...this.httpOptions.headers,
      },
      responseType: 'json',
      body: JSON.stringify(req),
      signal,
    });
    return res.data as T;
  }

  // 스트리밍 POST 요청
  async requestStreamingPost<T>(
    method: string,
    req: object,
    signal?: AbortSignal,
  ): Promise<AsyncGenerator<T>> {
    const res = await this.client.request({
      url: this.getMethodUrl(method),
      method: 'POST',
      params: {
        alt: 'sse', // Server-Sent Events
      },
      headers: {
        'Content-Type': 'application/json',
        ...this.httpOptions.headers,
      },
      responseType: 'stream',
      body: JSON.stringify(req),
      signal,
    });

    return (async function* (): AsyncGenerator<T> {
      const rl = readline.createInterface({
        input: res.data as NodeJS.ReadableStream,
        crlfDelay: Infinity,
      });

      let bufferedLines: string[] = [];
      for await (const line of rl) {
        if (line === '') {
          if (bufferedLines.length === 0) {
            continue;
          }
          yield JSON.parse(bufferedLines.join('\n')) as T;
          bufferedLines = [];
        } else if (line.startsWith('data: ')) {
          bufferedLines.push(line.slice(6).trim());
        } else {
          throw new Error(`Unexpected line format in response: ${line}`);
        }
      }
    })();
  }

  getMethodUrl(method: string): string {
    const endpoint = process.env.CODE_ASSIST_ENDPOINT ?? CODE_ASSIST_ENDPOINT;
    return `${endpoint}/${CODE_ASSIST_API_VERSION}:${method}`;
  }
}
```

**실제 OAuth HTTP 요청 예시:**
```
POST https://cloudcode-pa.googleapis.com/v1internal:generateContent
Content-Type: application/json
Authorization: Bearer oauth_access_token_here
User-Agent: GeminiCLI/0.1.13 (win32; x64)

{
  "model": "gemini-2.5-flash",
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "사용자 질문이 여기에" }]
    }
  ],
  "projectId": "your-project-id",
  "sessionId": "session-uuid-here",
  "systemInstruction": {
    "parts": [{ "text": "시스템 프롬프트..." }]
  },
  "tools": [
    {
      "functionDeclarations": [
        {
          "name": "writeFile",
          "description": "파일을 작성하는 도구",
          "parameters": { /* 스키마 */ }
        }
      ]
    }
  ]
}
```

### 5.2 API 키 방식 - GoogleGenAI

**왜 필요한가?**
- **간편한 설정**: API 키만으로 빠른 시작 가능
- **직접 액세스**: Google의 공식 SDK를 통한 안정적인 연결
- **비용 투명성**: 개인 API 키로 사용량 직접 관리
- **개발 환경**: 프로토타이핑 및 개인 프로젝트에 적합

API 키를 사용할 때는 `@google/genai` 패키지가 내부적으로 다음과 같은 HTTP 요청을 실행합니다:

**실제 API 키 HTTP 요청 예시:**
```
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent
Content-Type: application/json
x-goog-api-key: your_api_key_here
User-Agent: GeminiCLI/0.1.13 (win32; x64)

{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "사용자 질문" }]
    }
  ],
  "generationConfig": {
    "temperature": 0.7
  },
  "tools": [
    {
      "functionDeclarations": [/* 도구 정의들 */]
    }
  ]
}
```

### 5.3 재시도 로직

**왜 필요한가?**
- **안정성**: 일시적인 네트워크 오류나 서버 과부하 상황 대응
- **할당량 관리**: 사용량 초과 시 자동 모델 전환으로 서비스 연속성
- **비용 최적화**: Pro 모델에서 Flash 모델로 폴백하여 비용 절약
- **사용자 경험**: 수동 재시도 없이 자동으로 문제 해결

**파일:** `packages/core/src/utils/retry.ts`

```typescript
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options?: Partial<RetryOptions>,
): Promise<T> {
  const {
    maxAttempts,
    initialDelayMs,
    maxDelayMs,
    onPersistent429,
    authType,
    shouldRetry,
  } = {
    ...DEFAULT_RETRY_OPTIONS,
    ...options,
  };

  let attempt = 0;
  let currentDelay = initialDelayMs;
  let consecutive429Count = 0;

  while (attempt < maxAttempts) {
    attempt++;
    try {
      return await fn();
    } catch (error) {
      const errorStatus = getErrorStatus(error);

      // Pro 할당량 초과 에러 - OAuth 사용자 즉시 폴백
      if (
        errorStatus === 429 &&
        authType === AuthType.LOGIN_WITH_GOOGLE &&
        isProQuotaExceededError(error) &&
        onPersistent429
      ) {
        try {
          const fallbackModel = await onPersistent429(authType, error);
          if (fallbackModel !== false && fallbackModel !== null) {
            // 재시도 카운터 리셋하고 새 모델로 시도
            attempt = 0;
            consecutive429Count = 0;
            currentDelay = initialDelayMs;
            continue;
          } else {
            throw error;
          }
        } catch (fallbackError) {
          console.warn('Fallback to Flash model failed:', fallbackError);
        }
      }

      // 연속된 429 에러 추적
      if (errorStatus === 429) {
        consecutive429Count++;
      } else {
        consecutive429Count = 0;
      }

      // 지속적인 429 에러에 대한 폴백
      if (
        consecutive429Count >= 2 &&
        onPersistent429 &&
        authType === AuthType.LOGIN_WITH_GOOGLE
      ) {
        try {
          const fallbackModel = await onPersistent429(authType, error);
          if (fallbackModel !== false && fallbackModel !== null) {
            attempt = 0;
            consecutive429Count = 0;
            currentDelay = initialDelayMs;
            continue;
          } else {
            throw error;
          }
        } catch (fallbackError) {
          console.warn('Fallback to Flash model failed:', fallbackError);
        }
      }

      // 재시도 한도 초과 또는 재시도 불가능한 에러
      if (attempt >= maxAttempts || !shouldRetry(error as Error)) {
        throw error;
      }

      // Retry-After 헤더 확인 및 지수 백오프
      const { delayDurationMs } = getDelayDurationAndStatus(error);
      if (delayDurationMs > 0) {
        await delay(delayDurationMs);
        currentDelay = initialDelayMs;
      } else {
        const jitter = currentDelay * 0.3 * (Math.random() * 2 - 1);
        const delayWithJitter = Math.max(0, currentDelay + jitter);
        await delay(delayWithJitter);
        currentDelay = Math.min(maxDelayMs, currentDelay * 2);
      }
    }
  }

  throw new Error('Retry attempts exhausted');
}
```

## 6. 응답 처리

### 6.1 스트리밍 응답 처리

**왜 필요한가?**
- **실시간성**: 긴 응답도 즉시 표시 시작으로 체감 속도 향상
- **취소 가능**: 사용자가 중간에 응답을 취소할 수 있음
- **메모리 효율**: 전체 응답을 메모리에 저장하지 않고 청크 단위 처리
- **에러 복구**: 부분 응답이라도 사용자에게 유용한 정보 제공

**파일:** `packages/core/src/core/geminiChat.ts`

```typescript
async sendMessageStream(
  params: SendMessageParameters,
  prompt_id: string,
): Promise<AsyncGenerator<GenerateContentResponse>> {
  await this.sendPromise;
  const userContent = createUserContent(params.message);
  const requestContents = this.getHistory(true).concat(userContent);

  const startTime = Date.now();

  try {
    const apiCall = () => {
      const modelToUse = this.config.getModel();

      return this.contentGenerator.generateContentStream({
        model: modelToUse,
        contents: requestContents,
        config: { ...this.generationConfig, ...params.config },
      });
    };

    const streamResponse = await retryWithBackoff(apiCall, {
      shouldRetry: (error: Error) => {
        if (error && error.message) {
          if (error.message.includes('429')) return true;
          if (error.message.match(/5\d{2}/)) return true;
        }
        return false;
      },
      onPersistent429: async (authType?: string, error?: unknown) =>
        await this.handleFlashFallback(authType, error),
      authType: this.config.getContentGeneratorConfig()?.authType,
    });

    const result = this.processStreamResponse(
      streamResponse,
      userContent,
      startTime,
      prompt_id,
    );
    return result;
  } catch (error) {
    const durationMs = Date.now() - startTime;
    this._logApiError(durationMs, error, prompt_id);
    throw error;
  }
}

private async *processStreamResponse(
  streamResponse: AsyncGenerator<GenerateContentResponse>,
  inputContent: Content,
  startTime: number,
  prompt_id: string,
) {
  const outputContent: Content[] = [];
  const chunks: GenerateContentResponse[] = [];
  let errorOccurred = false;

  try {
    for await (const chunk of streamResponse) {
      if (isValidResponse(chunk)) {
        chunks.push(chunk);
        const content = chunk.candidates?.[0]?.content;
        if (content !== undefined) {
          // 생각(thinking) 부분 처리
          if (this.isThoughtContent(content)) {
            yield chunk;
            continue;
          }
          outputContent.push(content);
        }
      }
      yield chunk;
    }
  } catch (error) {
    errorOccurred = true;
    const durationMs = Date.now() - startTime;
    this._logApiError(durationMs, error, prompt_id);
    throw error;
  }

  if (!errorOccurred) {
    const durationMs = Date.now() - startTime;
    await this._logApiResponse(
      durationMs,
      prompt_id,
      this.getFinalUsageMetadata(chunks),
      JSON.stringify(chunks),
    );
  }

  // 히스토리 업데이트
  this.recordHistory(inputContent, outputContent);
}
```

### 6.2 히스토리 관리

**왜 필요한가?**
- **컨텍스트 유지**: 이전 대화 내용을 기억하여 연속성 있는 대화
- **중복 제거**: 인접한 동일 역할의 메시지 통합으로 토큰 절약
- **Thinking 필터링**: 사고 과정은 표시하되 히스토리에는 저장하지 않음
- **함수 호출 기록**: 도구 실행 결과를 포함한 완전한 대화 흐름 보존

**파일:** `packages/core/src/core/geminiChat.ts`

```typescript
private recordHistory(
  userInput: Content,
  modelOutput: Content[],
  automaticFunctionCallingHistory?: Content[],
) {
  const nonThoughtModelOutput = modelOutput.filter(
    (content) => !this.isThoughtContent(content),
  );

  let outputContents: Content[] = [];
  if (
    nonThoughtModelOutput.length > 0 &&
    nonThoughtModelOutput.every((content) => content.role !== undefined)
  ) {
    outputContents = nonThoughtModelOutput;
  } else if (nonThoughtModelOutput.length === 0 && modelOutput.length > 0) {
    // 모델이 생각만 반환한 경우 - 빈 응답 추가 안함
  } else {
    // Function response가 아닌 경우 빈 모델 응답 추가
    if (!isFunctionResponse(userInput)) {
      outputContents.push({
        role: 'model',
        parts: [],
      } as Content);
    }
  }

  // 자동 함수 호출 히스토리 처리
  if (
    automaticFunctionCallingHistory &&
    automaticFunctionCallingHistory.length > 0
  ) {
    this.history.push(
      ...extractCuratedHistory(automaticFunctionCallingHistory),
    );
  } else {
    this.history.push(userInput);
  }

  // 인접한 모델 역할 통합
  const consolidatedOutputContents: Content[] = [];
  for (const content of outputContents) {
    if (this.isThoughtContent(content)) {
      continue;
    }
    const lastContent =
      consolidatedOutputContents[consolidatedOutputContents.length - 1];
    if (this.isTextContent(lastContent) && this.isTextContent(content)) {
      // 텍스트 컨텐츠 병합
      lastContent.parts[0].text += content.parts[0].text || '';
      if (content.parts.length > 1) {
        lastContent.parts.push(...content.parts.slice(1));
      }
    } else {
      consolidatedOutputContents.push(content);
    }
  }

  if (consolidatedOutputContents.length > 0) {
    this.history.push(...consolidatedOutputContents);
  }
}
```

## 7. 도구 호출 처리

### 7.1 Turn 클래스에서의 도구 실행

**왜 필요한가?**
- **기능 확장**: LLM이 파일 시스템, 웹 검색 등 외부 도구 사용 가능
- **실시간 피드백**: 도구 실행 중에도 사용자에게 진행 상황 표시
- **안전한 실행**: 도구 호출을 격리하여 시스템 보안 유지
- **취소 지원**: 장시간 실행되는 도구도 중간에 취소 가능

**파일:** `packages/core/src/core/turn.ts`

```typescript
export class Turn {
  async *run(
    req: PartListUnion,
    signal: AbortSignal,
  ): AsyncGenerator<ServerGeminiStreamEvent> {
    try {
      const responseStream = await this.chat.sendMessageStream(
        {
          message: req,
          config: {
            abortSignal: signal,
          },
        },
        this.prompt_id,
      );

      for await (const resp of responseStream) {
        if (signal?.aborted) {
          yield { type: GeminiEventType.UserCancelled };
          return;
        }
        this.debugResponses.push(resp);

        // 생각(thought) 부분 처리
        const thoughtPart = resp.candidates?.[0]?.content?.parts?.[0];
        if (thoughtPart?.thought) {
          const rawText = thoughtPart.text ?? '';
          const subjectStringMatches = rawText.match(/\*\*(.*?)\*\*/s);
          const subject = subjectStringMatches
            ? subjectStringMatches[1].trim()
            : '';
          const description = rawText.replace(/\*\*(.*?)\*\*/s, '').trim();
          const thought: ThoughtSummary = {
            subject,
            description,
          };

          yield {
            type: GeminiEventType.Thought,
            value: thought,
          };
          continue;
        }

        // 일반 텍스트 응답 처리
        const text = getResponseText(resp);
        if (text) {
          yield { type: GeminiEventType.Content, value: text };
        }

        // 함수 호출 처리
        const functionCalls = resp.functionCalls ?? [];
        for (const fnCall of functionCalls) {
          const event = this.handlePendingFunctionCall(fnCall);
          if (event) {
            yield event;
          }
        }

        // 응답 완료 확인
        const finishReason = resp.candidates?.[0]?.finishReason;
        if (finishReason) {
          yield {
            type: GeminiEventType.Finished,
            value: finishReason as FinishReason,
          };
        }
      }
    } catch (e) {
      // 에러 처리...
      const structuredError: StructuredError = {
        message: getErrorMessage(error),
        status,
      };
      yield { type: GeminiEventType.Error, value: { error: structuredError } };
      return;
    }
  }

  private handlePendingFunctionCall(
    fnCall: FunctionCall,
  ): ServerGeminiStreamEvent | null {
    const callId =
      fnCall.id ??
      `${fnCall.name}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
    const name = fnCall.name || 'undefined_tool_name';
    const args = (fnCall.args || {}) as Record<string, unknown>;

    const toolCallRequest: ToolCallRequestInfo = {
      callId,
      name,
      args,
      isClientInitiated: false,
      prompt_id: this.prompt_id,
    };

    this.pendingToolCalls.push(toolCallRequest);

    return {
      type: GeminiEventType.ToolCallRequest,
      value: toolCallRequest,
    };
  }
}
```

### 7.2 비대화형 모드에서의 도구 실행

**왜 필요한가?**
- **자동화**: 사용자 개입 없이 연속적인 작업 체인 실행
- **배치 처리**: 스크립트나 CI/CD에서 도구를 활용한 복합 작업
- **결과 연결**: 이전 도구의 결과를 다음 LLM 호출에 자동 반영
- **에러 전파**: 도구 실행 실패 시 적절한 에러 메시지 전달

**파일:** `packages/cli/src/nonInteractiveCli.ts`

```typescript
// 함수 호출이 있는 경우 도구 실행
if (functionCalls.length > 0) {
  const toolResponseParts: Part[] = [];

  for (const fc of functionCalls) {
    const callId = fc.id ?? `${fc.name}-${Date.now()}`;
    const requestInfo: ToolCallRequestInfo = {
      callId,
      name: fc.name as string,
      args: (fc.args ?? {}) as Record<string, unknown>,
      isClientInitiated: false,
      prompt_id,
    };

    // 도구 실행
    const toolResponse = await executeToolCall(
      config,
      requestInfo,
      toolRegistry,
      abortController.signal,
    );

    // 도구 응답을 파트로 변환
    toolResponseParts.push({
      functionResponse: {
        name: fc.name as string,
        response: toolResponse.llmContent,
      },
    });
  }

  // 도구 응답을 다음 메시지로 설정
  currentMessages = [{ role: 'model', parts: toolResponseParts }];
} else {
  break; // 더 이상 도구 호출이 없으면 종료
}
```

## 8. 텔레메트리 및 로깅

### 8.1 API 요청/응답 로깅

**왜 필요한가?**
- **디버깅**: API 호출 문제 발생 시 상세한 로그로 원인 파악
- **사용 분석**: 어떤 모델과 기능이 많이 사용되는지 통계 수집
- **성능 모니터링**: 응답 시간, 토큰 사용량 등 성능 지표 추적
- **비용 관리**: API 사용량 기반의 비용 예측 및 최적화

**파일:** `packages/core/src/core/geminiChat.ts`

```typescript
private async _logApiRequest(
  contents: Content[],
  model: string,
  prompt_id: string,
): Promise<void> {
  const requestText = this._getRequestTextFromContents(contents);
  logApiRequest(
    this.config,
    new ApiRequestEvent(model, prompt_id, requestText),
  );
}

private async _logApiResponse(
  durationMs: number,
  prompt_id: string,
  usageMetadata?: GenerateContentResponseUsageMetadata,
  responseText?: string,
): Promise<void> {
  logApiResponse(
    this.config,
    new ApiResponseEvent(
      this.config.getModel(),
      durationMs,
      prompt_id,
      this.config.getContentGeneratorConfig()?.authType,
      usageMetadata,
      responseText,
    ),
  );
}

private _logApiError(
  durationMs: number,
  error: unknown,
  prompt_id: string,
): void {
  const errorMessage = error instanceof Error ? error.message : String(error);
  const errorType = error instanceof Error ? error.name : 'unknown';

  logApiError(
    this.config,
    new ApiErrorEvent(
      this.config.getModel(),
      errorMessage,
      durationMs,
      prompt_id,
      this.config.getContentGeneratorConfig()?.authType,
      errorType,
    ),
  );
}
```

### 8.2 텔레메트리 이벤트 타입

**왜 필요한가?**
- **구조화된 데이터**: 일관된 형식으로 분석 도구에서 쉽게 처리
- **타임스탬프**: 시간 기반 분석 및 성능 추적
- **분류**: 요청/응답/에러별로 다른 분석 및 알림 정책 적용
- **메타데이터**: 모델, 인증 방식 등 컨텍스트 정보 보존

**파일:** `packages/core/src/telemetry/types.ts`

```typescript
export class ApiRequestEvent {
  'event.name': 'api_request';
  'event.timestamp': string; // ISO 8601
  model: string;
  prompt_id: string;
  request_text: string;

  constructor(model: string, prompt_id: string, request_text: string) {
    this['event.name'] = 'api_request';
    this['event.timestamp'] = new Date().toISOString();
    this.model = model;
    this.prompt_id = prompt_id;
    this.request_text = request_text;
  }
}

export class ApiResponseEvent {
  'event.name': 'api_response';
  'event.timestamp': string;
  model: string;
  duration_ms: number;
  prompt_id: string;
  auth_type?: string;
  usage_metadata?: GenerateContentResponseUsageMetadata;
  response_text?: string;

  constructor(
    model: string,
    duration_ms: number,
    prompt_id: string,
    auth_type?: string,
    usage_metadata?: GenerateContentResponseUsageMetadata,
    response_text?: string,
  ) {
    this['event.name'] = 'api_response';
    this['event.timestamp'] = new Date().toISOString();
    this.model = model;
    this.duration_ms = duration_ms;
    this.prompt_id = prompt_id;
    this.auth_type = auth_type;
    this.usage_metadata = usage_metadata;
    this.response_text = response_text;
  }
}

export class ApiErrorEvent {
  'event.name': 'api_error';
  'event.timestamp': string;
  model: string;
  error_message: string;
  duration_ms: number;
  prompt_id: string;
  auth_type?: string;
  error_type: string;

  constructor(
    model: string,
    error_message: string,
    duration_ms: number,
    prompt_id: string,
    auth_type?: string,
    error_type: string,
  ) {
    this['event.name'] = 'api_error';
    this['event.timestamp'] = new Date().toISOString();
    this.model = model;
    this.error_message = error_message;
    this.duration_ms = duration_ms;
    this.prompt_id = prompt_id;
    this.auth_type = auth_type;
    this.error_type = error_type;
  }
}
```

## 9. 전체 플로우 요약

### 9.1 핵심 실행 경로

```
main()
  ↓
loadSettings() → parseArguments() → loadCliConfig()
  ↓
config.initialize() → createContentGenerator() → GeminiClient.initialize()
  ↓
shouldBeInteractive ? render(<AppWrapper>) : runNonInteractive()
  ↓
사용자 입력 수집 (React UI 또는 stdin)
  ↓
chat.sendMessageStream() → geminiChat.sendMessageStream()
  ↓
retryWithBackoff(contentGenerator.generateContentStream())
  ↓
OAuth: CodeAssistServer.requestStreamingPost()
API Key: GoogleGenAI.models.generateContentStream()
  ↓
processStreamResponse() → recordHistory()
  ↓
도구 호출 감지 → executeToolCall() → 결과를 다시 모델에 전송
  ↓
최종 응답 표시 및 히스토리 업데이트
```

### 9.2 주요 컴포넌트 역할

- **GeminiClient**: 메인 API 클라이언트, 설정 관리 및 전체 플로우 조정
- **GeminiChat**: 대화 세션 관리, 히스토리 유지, 스트리밍 처리
- **ContentGenerator**: 인터페이스, 실제 API 호출 추상화
  - **CodeAssistServer**: OAuth 인증용 구현체
  - **GoogleGenAI.models**: API 키 인증용 구현체
- **Turn**: 단일 턴의 대화 관리, 도구 호출 처리
- **ToolRegistry**: 사용 가능한 도구 관리 및 실행

### 9.3 에러 처리 및 폴백

- **재시도 로직**: 429, 5xx 에러에 대한 지수 백오프
- **모델 폴백**: OAuth 사용자의 할당량 초과시 Flash 모델로 자동 전환
- **취소 처리**: AbortController를 통한 요청 취소
- **텔레메트리**: 모든 API 호출에 대한 로깅 및 메트릭 수집

### 9.4 보안 고려사항

- **API 키 관리**: 환경 변수를 통한 안전한 저장
- **OAuth 토큰**: 로컬 캐시에 암호화 저장
- **프록시 지원**: 기업 환경에서의 네트워크 제한 대응
- **샌드박스**: 도구 실행을 위한 격리된 환경 제공

이 전체 플로우는 확장 가능하고 견고한 아키텍처를 통해 다양한 인증 방식과 사용 사례를 지원하며, 에러 상황에서도 안정적으로 동작하도록 설계되었습니다.

## 10. 구현 단계별 가이드

### 10.1 1단계: 기본 인프라 구축

**우선순위가 높은 이유**: 모든 상위 기능의 기반이 되는 핵심 컴포넌트

1. **ContentGenerator 인터페이스 정의**
   ```typescript
   interface ContentGenerator {
     generateContent(req: GenerateContentParameters): Promise<GenerateContentResponse>;
     generateContentStream(req: GenerateContentParameters): AsyncGenerator<GenerateContentResponse>;
   }
   ```

2. **기본 설정 시스템 구현**
   - 환경 변수 로딩
   - 설정 파일 파싱
   - 기본값 정의

3. **에러 처리 시스템 구현**
   - 구조화된 에러 타입 정의
   - 에러 로깅 유틸리티
   - 기본 재시도 로직

### 10.2 2단계: 단일 인증 방식 구현

**우선순위가 높은 이유**: 가장 간단한 방식부터 시작하여 기본 플로우 검증

1. **API 키 방식 먼저 구현** (OAuth보다 단순)
   - GoogleGenAI ContentGenerator 구현
   - 환경 변수에서 API 키 읽기
   - 기본 HTTP 요청/응답 처리

2. **기본 GeminiClient 클래스**
   ```typescript
   class GeminiClient {
     constructor(config: Config)
     initialize(contentGeneratorConfig: ContentGeneratorConfig)
     sendMessage(message: string): Promise<string>
   }
   ```

3. **단순한 CLI 인터페이스**
   - 명령행 인자 파싱
   - 단일 질문-답변 처리
   - 기본 출력 형식

### 10.3 3단계: 스트리밍 기능 추가

**우선순위가 높은 이유**: 사용자 경험에 큰 영향을 미치는 핵심 기능

1. **스트리밍 응답 처리**
   ```typescript
   async *generateContentStream(): AsyncGenerator<GenerateContentResponse> {
     // 청크 단위 응답 처리
   }
   ```

2. **실시간 출력 시스템**
   - 터미널에 실시간 텍스트 출력
   - 취소 신호 처리 (AbortController)
   - 진행 상황 표시

3. **기본 에러 복구**
   - 스트리밍 중 연결 끊김 처리
   - 부분 응답 저장 및 복구

### 10.4 4단계: 대화 히스토리 시스템

**우선순위가 높은 이유**: 실용적인 대화형 AI를 위한 필수 기능

1. **GeminiChat 클래스 구현**
   ```typescript
   class GeminiChat {
     private history: Content[] = [];
     sendMessage(message: string): Promise<GenerateContentResponse>
     sendMessageStream(message: string): AsyncGenerator<GenerateContentResponse>
   }
   ```

2. **히스토리 관리 로직**
   - 사용자/모델 메시지 저장
   - 컨텍스트 윈도우 관리
   - 중복 메시지 통합

3. **시스템 프롬프트 지원**
   - 역할 정의 및 지시사항
   - 모델별 최적화된 프롬프트

### 10.5 5단계: 도구 호출 시스템

**우선순위가 중간인 이유**: 고급 기능이지만 실용성을 크게 높임

1. **ToolRegistry 구현**
   ```typescript
   class ToolRegistry {
     registerTool(tool: Tool): void
     getFunctionDeclarations(): FunctionDeclaration[]
     executeTool(name: string, args: object): Promise<ToolResult>
   }
   ```

2. **기본 도구들 구현**
   - 파일 읽기/쓰기 도구
   - 디렉토리 나열 도구
   - 간단한 검색 도구

3. **도구 실행 엔진**
   - 안전한 함수 호출
   - 결과 검증 및 포맷팅
   - 에러 처리 및 복구

### 10.6 6단계: OAuth 인증 추가

**우선순위가 중간인 이유**: 엔터프라이즈 사용자를 위한 중요하지만 복잡한 기능

1. **OAuth 클라이언트 구현**
   - Google OAuth 2.0 플로우
   - 토큰 저장 및 갱신
   - 권한 스코프 관리

2. **CodeAssistServer ContentGenerator**
   ```typescript
   class CodeAssistServer implements ContentGenerator {
     constructor(authClient: OAuth2Client, projectId: string)
     requestPost<T>(method: string, req: object): Promise<T>
     requestStreamingPost<T>(method: string, req: object): AsyncGenerator<T>
   }
   ```

3. **인증 플로우 통합**
   - 브라우저 기반 인증
   - 토큰 캐시 관리
   - 만료 시 자동 갱신

### 10.7 7단계: 고급 에러 처리 및 재시도

**우선순위가 중간인 이유**: 안정성 향상을 위한 중요한 기능

1. **재시도 로직 고도화**
   ```typescript
   async function retryWithBackoff<T>(
     fn: () => Promise<T>,
     options: RetryOptions
   ): Promise<T>
   ```

2. **모델 폴백 시스템**
   - 할당량 초과 시 Flash 모델로 전환
   - 모델별 기능 호환성 검사
   - 비용 최적화 로직

3. **네트워크 에러 처리**
   - 연결 타임아웃 관리
   - 프록시 지원
   - DNS 해결 실패 처리

### 10.8 8단계: 사용자 인터페이스 개선

**우선순위가 낮은 이유**: 기능적으로는 완성되었으나 사용자 경험 향상

1. **React 기반 터미널 UI**
   - Ink 라이브러리 활용
   - 대화형 컴포넌트
   - 키보드 단축키 지원

2. **테마 및 커스터마이제이션**
   - 색상 테마 지원
   - 출력 형식 옵션
   - 폰트 및 레이아웃 설정

3. **진행 상황 표시**
   - 로딩 애니메이션
   - 진행률 바
   - 상태 메시지

### 10.9 9단계: 텔레메트리 및 로깅

**우선순위가 낮은 이유**: 운영 및 디버깅을 위한 부가 기능

1. **로깅 시스템 구축**
   ```typescript
   class TelemetryLogger {
     logApiRequest(event: ApiRequestEvent): void
     logApiResponse(event: ApiResponseEvent): void
     logApiError(event: ApiErrorEvent): void
   }
   ```

2. **메트릭 수집**
   - 사용량 통계
   - 성능 지표
   - 에러 발생률

3. **로그 저장 및 분석**
   - 로컬 파일 저장
   - 클라우드 전송 (선택적)
   - 대시보드 연동

### 10.10 10단계: 최적화 및 고급 기능

**우선순위가 가장 낮은 이유**: 기본 기능이 완성된 후 성능 및 고급 사용자를 위한 기능

1. **성능 최적화**
   - 응답 캐싱
   - 토큰 사용량 최적화
   - 메모리 사용량 관리

2. **고급 설정 옵션**
   - 커스텀 모델 파라미터
   - 고급 프롬프트 엔지니어링
   - 플러그인 시스템

3. **통합 및 확장성**
   - VS Code 확장
   - API 서버 모드
   - 다른 도구와의 연동

### 구현 팁

**핵심 원칙:**
1. **점진적 개발**: 각 단계는 독립적으로 테스트 가능해야 함
2. **테스트 우선**: 각 컴포넌트는 단위 테스트와 함께 개발
3. **설정 가능**: 하드코딩 대신 설정으로 동작 변경 가능
4. **에러 처리**: 모든 외부 호출에 적절한 에러 처리 포함

**개발 순서의 이유:**
- 1-3단계: 기본 동작하는 MVP 완성
- 4-5단계: 실용적인 기능 추가로 사용자 가치 제공
- 6-7단계: 엔터프라이즈 및 안정성 요구사항 충족
- 8-10단계: 사용자 경험 및 운영 효율성 향상

이런 단계별 접근을 통해 각 단계마다 동작하는 제품을 확보하면서, 점진적으로 기능을 확장할 수 있습니다.

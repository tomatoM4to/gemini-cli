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

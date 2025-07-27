# Gemini CLI 샌드박스 시스템 분석

## 📋 개요

Gemini CLI의 샌드박스 시스템은 AI가 실행하는 위험한 도구들을 안전하게 격리하는 핵심 보안 메커니즘입니다. 플랫폼별로 최적화된 격리 기술을 사용하여 호스트 시스템을 보호합니다.

## 🏗️ 아키텍처 개요

```
┌─────────────────────────────────────────┐
│               Gemini CLI                │
│            (sandbox.ts)                 │
└─────────────┬───────────────────────────┘
              │
    ┌─────────▼─────────┐
    │ 플랫폼 감지        │
    │ os.platform()     │
    └─────────┬─────────┘
              │
    ┌─────────▼─────────┬─────────────────┬─────────────────┐
    │      macOS        │     Linux       │     Windows     │
    │  sandbox-exec     │ Docker/Podman   │ Docker/Podman   │
    │   (Seatbelt)      │   (UID/GID)     │  (경로 변환)     │
    └─────────┬─────────┴─────────────────┴─────────────────┘
              │
    ┌─────────▼─────────┐
    │   격리된 환경에서   │
    │   위험한 AI 도구   │
    │      안전 실행     │
    └───────────────────┘
```

## 🖥️ 플랫폼별 구현

### 1. **macOS - sandbox-exec (Seatbelt)**

**특징:**
- 시스템 레벨 경량 격리
- Apple의 내장 Seatbelt 프레임워크 사용
- 프로파일 기반 권한 제어

**구현:**
```typescript
if (config.command === 'sandbox-exec') {
  const profile = process.env.SEATBELT_PROFILE || 'permissive-open';
  const args = [
    '-D', `TARGET_DIR=${fs.realpathSync(process.cwd())}`,
    '-D', `TMP_DIR=${fs.realpathSync(os.tmpdir())}`,
    '-D', `HOME_DIR=${fs.realpathSync(os.homedir())}`,
    '-D', `CACHE_DIR=${darwin_cache_dir}`,
    '-f', profileFile,
    'sh', '-c', command
  ];
}
```

**보안 프로파일:**
- `permissive-open`: 기본 허용, 특정 쓰기만 차단
- `permissive-closed`: 기본 허용, 네트워크 제한
- `restrictive-open`: 기본 거부, 필요한 것만 허용
- `restrictive-closed`: 최고 보안, 최소 권한

### 2. **Linux - Docker/Podman + UID/GID 매핑**

**특징:**
- 완전한 컨테이너 격리
- 파일 권한 문제 해결을 위한 UID/GID 매핑
- Debian/Ubuntu 자동 감지

**구현:**
```typescript
// Debian/Ubuntu 자동 감지
async function shouldUseCurrentUserInSandbox(): Promise<boolean> {
  if (os.platform() === 'linux') {
    const osReleaseContent = await readFile('/etc/os-release', 'utf8');
    if (osReleaseContent.includes('ID=debian') ||
        osReleaseContent.includes('ID=ubuntu')) {
      return true;
    }
  }
  return false;
}

// 사용자 생성 및 권한 처리
const setupUserCommands = [
  `groupadd -f -g ${gid} ${username}`,
  `useradd -o -u ${uid} -g ${gid} -d ${homeDir} -s /bin/bash ${username}`
].join(' && ');
```

### 3. **Windows - Docker + 경로 변환**

**특징:**
- Docker Desktop 기반 격리
- Windows 경로를 Unix 스타일로 변환
- 볼륨 마운트 최적화

**구현:**
```typescript
function getContainerPath(hostPath: string): string {
  if (os.platform() !== 'win32') {
    return hostPath;
  }
  // C:\path\to\file -> /c/path/to/file
  const withForwardSlashes = hostPath.replace(/\\/g, '/');
  const match = withForwardSlashes.match(/^([A-Z]):\/(.*)/i);
  if (match) {
    return `/${match[1].toLowerCase()}/${match[2]}`;
  }
  return hostPath;
}
```

## 🛡️ 보안 계층

### **파일 시스템 보호**

**허용된 쓰기 경로:**
```
TARGET_DIR     - 현재 작업 디렉토리
TMP_DIR        - 임시 파일 디렉토리
~/.gemini      - 설정 파일
~/.npm         - npm 캐시
~/.cache       - 일반 캐시
/dev/std*      - 표준 입출력
```

**차단된 영역:**
```
~/Documents    - 사용자 문서
~/Desktop      - 바탕화면
/etc           - 시스템 설정
/usr           - 시스템 바이너리
/bin           - 실행 파일
```

### **네트워크 격리**

**프록시 지원:**
```typescript
// 격리된 네트워크 + 프록시 컨테이너
const SANDBOX_NETWORK_NAME = 'gemini-cli-sandbox';
const SANDBOX_PROXY_NAME = 'gemini-cli-sandbox-proxy';

// 내부 네트워크 생성
execSync(`${command} network create --internal ${SANDBOX_NETWORK_NAME}`);

// 프록시를 통한 외부 접근
if (proxyCommand) {
  const proxyContainer = `${command} run --network ${SANDBOX_PROXY_NAME} -p 8877:8877`;
}
```

## ⚙️ 환경 구성

### **볼륨 마운트 전략**

```typescript
// 작업 디렉토리
args.push('--volume', `${workdir}:${containerWorkdir}`);

// 사용자 설정
args.push('--volume', `${userSettingsDir}:${containerSettingsDir}`);

// 임시 디렉토리
args.push('--volume', `${os.tmpdir()}:${getContainerPath(os.tmpdir())}`);

// Google Cloud 설정 (읽기 전용)
if (fs.existsSync(gcloudConfigDir)) {
  args.push('--volume', `${gcloudConfigDir}:${containerPath}:ro`);
}
```

### **환경 변수 복사**

```typescript
// API 키들
GEMINI_API_KEY, GOOGLE_API_KEY

// Google Cloud 설정
GOOGLE_GENAI_USE_VERTEXAI, GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_LOCATION

// 터미널 설정
TERM, COLORTERM

// 개발 환경
NODE_OPTIONS, VIRTUAL_ENV (작업 디렉토리 내부만)

// 프록시 설정
HTTPS_PROXY, HTTP_PROXY, NO_PROXY
```

## 🔧 고급 기능

### **커스텀 샌드박스**

**프로젝트별 Dockerfile:**
```dockerfile
# .gemini/sandbox.Dockerfile
FROM gemini-cli-sandbox:latest
RUN apt-get update && apt-get install -y custom-tools
COPY custom-scripts/ /usr/local/bin/
```

**프로젝트별 bashrc:**
```bash
# .gemini/sandbox.bashrc
export CUSTOM_VAR="project-specific"
source /usr/local/bin/setup-env.sh
```

### **개발자 도구**

**디버깅 포트 노출:**
```typescript
if (process.env.DEBUG) {
  const debugPort = process.env.DEBUG_PORT || '9229';
  args.push('--publish', `${debugPort}:${debugPort}`);
}
```

**빌드 시스템 통합:**
```typescript
if (process.env.BUILD_SANDBOX) {
  execSync(`cd ${gcRoot} && node scripts/build_sandbox.js -s ${buildArgs}`);
}
```

## 📊 보안 레벨 비교

| 모드 | 파일 제한 | 네트워크 | 프로세스 | 사용 사례 |
|------|-----------|----------|----------|-----------|
| **permissive-open** | 🟡 부분 제한 | 🟢 전체 허용 | 🟢 전체 허용 | 일반 개발 |
| **permissive-closed** | 🟡 부분 제한 | 🟡 제한적 | 🟢 전체 허용 | 네트워크 격리 |
| **restrictive-open** | 🔴 강력 제한 | 🟢 전체 허용 | 🟡 제한적 | 보안 우선 |
| **restrictive-closed** | 🔴 강력 제한 | 🔴 강력 제한 | 🟡 제한적 | 최고 보안 |

## 🚀 실제 사용 예시

### **기본 사용:**
```bash
# 자동 플랫폼 감지
gemini "프로젝트 README를 업데이트해줘"

# macOS에서 restrictive 모드
SEATBELT_PROFILE=restrictive-closed gemini "안전하게 파일을 수정해줘"

# Linux에서 강제 UID/GID 사용
SANDBOX_SET_UID_GID=1 gemini "권한 문제 없이 파일 작업해줘"
```

### **고급 설정:**
```bash
# 커스텀 마운트
SANDBOX_MOUNTS="/host/data:/container/data:ro" gemini "데이터 분석해줘"

# 포트 노출
SANDBOX_PORTS="8080,3000" gemini "웹 서버를 시작해줘"

# 프록시 사용
GEMINI_SANDBOX_PROXY_COMMAND="mitmproxy" gemini "네트워크 요청 모니터링하며 작업해줘"
```

## 🎯 범용 AI 에이전트 적용 포인트

### **1. 멀티 플랫폼 지원**
- 단일 코드베이스로 모든 OS 지원
- 플랫폼별 최적화된 격리 기술 활용
- 자동 환경 감지 및 설정

### **2. 확장 가능한 보안 정책**
- 프로파일 기반 권한 관리
- 프로젝트별 커스터마이징
- 단계적 보안 레벨 제공

### **3. 개발자 친화적 설계**
- 투명한 격리 (사용자가 거의 인지하지 못함)
- 디버깅 및 개발 도구 지원
- 설정 가능한 환경 변수

### **4. 운영 환경 대응**
- CI/CD 통합 지원
- 컨테이너 런타임 선택 (Docker/Podman)
- 네트워크 정책 제어

## 🔮 향후 개선 방향

1. **더 세밀한 권한 제어**: 도구별 개별 권한 설정
2. **실시간 모니터링**: 샌드박스 내 활동 로깅
3. **성능 최적화**: 컨테이너 시작 시간 단축
4. **추가 플랫폼 지원**: BSD, Solaris 등

---

**작성일**: 2025-01-27
**목적**: 범용 AI 에이전트 개발을 위한 샌드박스 시스템 이해 ✅

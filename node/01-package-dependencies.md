# Package Dependencies & Lock File

npm 패키지 의존성 타입과 package-lock.json의 역할 정리.

## Dependency Types

### dependencies

프로덕션 런타임에 필요한 패키지. 앱이 실행될 때 반드시 있어야 한다.

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "axios": "^1.6.0"
  }
}
```

- `npm install react` 하면 여기에 추가
- 이 패키지를 `npm install`하는 소비자도 함께 설치하게 됨

### devDependencies

개발/빌드 시에만 필요한 패키지. 프로덕션 번들에는 포함되지 않는다.

```json
{
  "devDependencies": {
    "typescript": "^5.3.0",
    "jest": "^29.7.0",
    "eslint": "^8.56.0",
    "@types/react": "^18.2.0"
  }
}
```

- `npm install -D typescript` 하면 여기에 추가
- 라이브러리를 배포할 때, 소비자가 `npm install your-lib` 하면 devDependencies는 설치되지 않음
- 빌드 도구, 테스트 프레임워크, 타입 정의, 린터 등이 해당

### peerDependencies

"나는 이 패키지가 필요한데, 직접 설치하지 않을 테니 호스트(소비자)가 설치해줘"라는 의미.

```json
{
  "name": "my-react-component-lib",
  "peerDependencies": {
    "react": "^17.0.0 || ^18.0.0",
    "react-dom": "^17.0.0 || ^18.0.0"
  }
}
```

주로 라이브러리/플러그인에서 사용한다. 핵심 이유는 싱글턴 보장:

```
# peerDependencies 없이 dependencies로 react를 넣으면:
node_modules/
├── react@18.2.0              ← 앱의 react
└── my-lib/
    └── node_modules/
        └── react@18.3.0     ← 라이브러리의 react (중복!)

# React가 2개 → Context, hooks 등이 깨짐
# peerDependencies로 선언하면 앱의 react 하나만 공유
```

npm 버전별 동작 차이:

| npm 버전 | peerDependencies 동작 |
|----------|----------------------|
| npm 3~6 | 경고만 출력, 자동 설치 안 함 |
| npm 7+ | 자동 설치 시도, 충돌 시 에러 (`--legacy-peer-deps`로 우회) |

### peerDependenciesMeta

peer dependency를 선택적으로 만들 수 있다.

```json
{
  "peerDependencies": {
    "react": "^18.0.0",
    "react-native": ">=0.70.0"
  },
  "peerDependenciesMeta": {
    "react-native": {
      "optional": true
    }
  }
}
```

### optionalDependencies

설치 실패해도 `npm install` 전체가 실패하지 않는 패키지. 플랫폼별 네이티브 바인딩 등에 사용.

```json
{
  "optionalDependencies": {
    "fsevents": "^2.3.0"
  }
}
```

`fsevents`는 macOS 전용이라 Linux에서는 설치 실패하지만, 없어도 동작하도록 fallback이 있다.

## package-lock.json

### 왜 필요한가

`package.json`의 버전 범위(`^18.2.0`)는 설치 시점에 따라 다른 버전이 설치될 수 있다.

```json
// package.json
"react": "^18.2.0"  // 18.2.0 ~ 18.x.x 중 최신

// 2024년 1월 설치 → react@18.2.0
// 2024년 6월 설치 → react@18.3.1
// 같은 package.json인데 다른 결과 → "works on my machine" 문제
```

`package-lock.json`은 정확한 버전, 다운로드 URL, integrity hash까지 고정한다.

### 구조

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "lockfileVersion": 3,
  "packages": {
    "": {
      "name": "my-app",
      "dependencies": {
        "axios": "^1.6.0"
      }
    },
    "node_modules/axios": {
      "version": "1.6.7",
      "resolved": "https://registry.npmjs.org/axios/-/axios-1.6.7.tgz",
      "integrity": "sha512-/hDJGff6/c7u0hDkvkGxR/oj6BVSqNTTf...",
      "dependencies": {
        "follow-redirects": "^1.15.4"
      }
    }
  }
}
```

| 필드 | 역할 |
|------|------|
| `version` | 실제 설치된 정확한 버전 |
| `resolved` | 다운로드 URL |
| `integrity` | SHA-512 해시 (변조 방지) |
| `dependencies` | 이 패키지의 하위 의존성 |

### npm install vs npm ci

| | `npm install` | `npm ci` |
|---|---|---|
| lock 파일 | 없으면 생성, 있으면 업데이트 가능 | 반드시 있어야 함 |
| node_modules | 기존 것에 추가/업데이트 | 삭제 후 처음부터 설치 |
| 버전 결정 | package.json 범위 내 최신 | lock 파일 그대로 |
| 용도 | 개발 중 | CI/CD, 프로덕션 빌드 |
| 속도 | 느림 (의존성 해석) | 빠름 (해석 불필요) |

CI/CD에서는 반드시 `npm ci`를 써야 한다. `npm install`은 lock 파일을 변경할 수 있어서 재현 불가능한 빌드가 될 수 있다.

### lock 파일 커밋 여부

| 프로젝트 타입 | 커밋? | 이유 |
|--------------|-------|------|
| 앱 (SPA, 서버) | O | 모든 환경에서 동일한 의존성 보장 |
| 라이브러리 | X (보통) | 소비자의 의존성 해석을 방해할 수 있음 |

### Dependency Resolution (호이스팅)

npm은 가능한 한 의존성을 최상위 `node_modules`로 끌어올린다 (hoisting).

```
# A가 lodash@4.17.0, B가 lodash@4.17.0 의존
node_modules/
├── A/
├── B/
└── lodash@4.17.0    ← 호이스팅됨, A와 B가 공유

# A가 lodash@4.17.0, B가 lodash@3.10.0 의존 (충돌)
node_modules/
├── A/
├── B/
│   └── node_modules/
│       └── lodash@3.10.0   ← B 전용
└── lodash@4.17.0            ← A가 사용
```

이 호이스팅 결과가 `package-lock.json`에 기록된다.

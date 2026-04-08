# Package Manager Comparison

npm, yarn, pnpm, bun의 의존성 관리 방식 비교.

## Lock File

| | npm | yarn (classic) | yarn (berry/4+) | pnpm | bun |
|---|---|---|---|---|---|
| lock 파일 | `package-lock.json` | `yarn.lock` | `yarn.lock` | `pnpm-lock.yaml` | `bun.lock` |
| 포맷 | JSON | 커스텀 | YAML | YAML | 바이너리 (text: `bun.lock`) |
| lockfileVersion | 3 (npm 9+) | 1 | 8+ | 9+ | - |

## node_modules 구조

패키지 매니저 간 가장 큰 차이점.

### npm — flat (hoisted)

```
node_modules/
├── A/
├── B/
├── lodash/          ← A, B 모두 사용 (호이스팅)
└── C/
    └── node_modules/
        └── lodash/  ← 버전 충돌 시에만 중첩
```

- 기본적으로 모든 의존성을 최상위로 끌어올림
- 문제: phantom dependency — 직접 설치하지 않은 패키지도 `require('lodash')` 가능

### yarn classic — flat (hoisted)

npm과 거의 같은 구조. 동일한 phantom dependency 문제 존재.

### pnpm — content-addressable store + symlink

```
node_modules/
├── .pnpm/                          ← 실제 패키지 저장소
│   ├── lodash@4.17.21/
│   │   └── node_modules/
│   │       └── lodash/ → store     ← global store에서 hard link
│   └── A@1.0.0/
│       └── node_modules/
│           ├── A/ → store
│           └── lodash/ → ../../lodash@4.17.21/...  ← symlink
├── A/ → .pnpm/A@1.0.0/node_modules/A   ← symlink
└── B/ → .pnpm/B@1.0.0/node_modules/B
```

- phantom dependency 원천 차단 — package.json에 선언하지 않은 패키지는 접근 불가
- global content-addressable store (`~/.pnpm-store`)에서 hard link → 디스크 절약
- 모노레포에서 특히 강력

### yarn berry (PnP — Plug'n'Play)

```
# node_modules 자체가 없음!
.yarn/
├── cache/
│   ├── lodash-npm-4.17.21-abc123.zip
│   └── react-npm-18.2.0-def456.zip
└── unplugged/     ← 네이티브 모듈만 여기에 풀림
.pnp.cjs           ← 모듈 해석 맵
```

- `node_modules` 폴더 없이 `.pnp.cjs`가 모듈 위치를 직접 매핑
- zip 파일에서 직접 읽음 → 설치 속도 극적으로 빠름, 디스크 사용 최소
- 단점: 일부 패키지 호환성 문제 (node_modules를 직접 탐색하는 패키지)

### bun — flat (hoisted) + 바이너리 lock

- npm 호환 구조지만, Zig로 작성된 런타임이라 설치 속도가 압도적으로 빠름
- 바이너리 lock 파일로 파싱 속도 최적화

## 설치 명령어 비교

| 동작 | npm | yarn | pnpm | bun |
|------|-----|------|------|-----|
| 설치 | `npm install` | `yarn` | `pnpm install` | `bun install` |
| CI 설치 | `npm ci` | `yarn --frozen-lockfile` | `pnpm install --frozen-lockfile` | `bun install --frozen-lockfile` |
| 패키지 추가 | `npm install pkg` | `yarn add pkg` | `pnpm add pkg` | `bun add pkg` |
| dev 추가 | `npm install -D pkg` | `yarn add -D pkg` | `pnpm add -D pkg` | `bun add -d pkg` |
| peer 추가 | 수동 | `yarn add --peer pkg` | `pnpm add --save-peer pkg` | 수동 |
| 글로벌 설치 | `npm install -g pkg` | `yarn global add pkg` | `pnpm add -g pkg` | `bun add -g pkg` |
| 스크립트 실행 | `npm run dev` | `yarn dev` | `pnpm dev` | `bun run dev` |
| npx 대체 | `npx pkg` | `yarn dlx pkg` | `pnpm dlx pkg` | `bunx pkg` |

## peerDependencies 처리 비교

| | npm 7+ | yarn classic | yarn berry | pnpm | bun |
|---|---|---|---|---|---|
| 자동 설치 | O | X (경고만) | O | X (경고만) | O |
| 충돌 시 | 에러 | 경고 | 에러 | 경고 | 에러 |
| 우회 플래그 | `--legacy-peer-deps` | - | - | `--no-strict-peer-dependencies` | - |

### peerDependencies가 없으면 어떻게 되는가? (What Happens When peerDeps Are Missing?)

peerDependencies가 설치되지 않으면 해당 패키지는 제대로 동작하지 않는다.

**런타임 에러 발생 (Runtime Error):**
```js
// my-react-component-lib 내부 코드
import React from 'react';  // ← react가 없으면 여기서 터짐

// Error: Cannot find module 'react'
```

**버전 불일치 (Version Mismatch):**
```
peerDependencies: { "react": "^18.0.0" }
설치된 react: 17.0.2

→ 설치는 되지만, 런타임에 API 불일치로 에러 가능
  예: React 18의 useId() 훅을 쓰는 라이브러리 + React 17 = TypeError
```

**중복 설치 (Duplicate Installation) — peerDep 대신 dependencies로 넣은 경우:**
```
node_modules/
├── react@18.2.0/                    ← 앱의 React
└── my-lib/
    └── node_modules/
        └── react@18.3.0/           ← 라이브러리의 React (다른 인스턴스!)

→ React가 2개 → Context 공유 안 됨, hooks 규칙 위반 (Rules of Hooks Violation)
  "Invalid hook call. Hooks can only be called inside the body of a function component."
```

이게 peerDependencies가 존재하는 이유. 싱글턴이어야 하는 패키지 (Singleton Packages)의 중복을 방지한다.

**자동 설치하지 않는 패키지 매니저 (pnpm, yarn classic) 사용 시:**
```bash
# pnpm — 경고 출력, 직접 설치 필요
pnpm install my-react-component-lib
#   WARN  Issues with peer dependencies found
#   └── ✕ missing peer react@"^18.0.0"

pnpm add react    # 직접 설치해야 함
```

**라이브러리 개발자가 해야 할 일 (What Library Authors Should Do):**
```json
{
  "peerDependencies": {
    "react": "^17.0.0 || ^18.0.0 || ^19.0.0"
  },
  "devDependencies": {
    "react": "^18.2.0"
  }
}
```
- `peerDependencies`: 소비자에게 "이거 필요해"라고 알림 (Declaration)
- `devDependencies`: 개발/테스트 시에는 직접 설치해서 사용 (Development)
- 버전 범위를 넓게 잡아서 호환성 극대화 (Wide Version Range)

## Phantom Dependency 문제

```js
// package.json에 lodash를 직접 설치하지 않았는데
// A 패키지가 lodash에 의존하고 있어서 호이스팅됨
const _ = require('lodash'); // npm, yarn classic: 동작함 (위험!)
                              // pnpm: Error! (안전)
                              // yarn PnP: Error! (안전)
```

| | npm | yarn classic | yarn berry (PnP) | pnpm | bun |
|---|---|---|---|---|---|
| phantom dependency | 허용 (위험) | 허용 (위험) | 차단 | 차단 | 허용 (위험) |

## 모노레포 워크스페이스 지원

| | npm | yarn | pnpm | bun |
|---|---|---|---|---|
| 워크스페이스 | `workspaces` (7+) | `workspaces` | `pnpm-workspace.yaml` | `workspaces` |
| 호이스팅 제어 | 제한적 | `.yarnrc.yml` | 기본 격리 | 제한적 |
| 패키지 간 링크 | symlink | symlink/PnP | symlink | symlink |
| 성숙도 | 보통 | 높음 | 높음 | 성장 중 |

## 성능 비교 (일반적 경향)

```
설치 속도 (cold):  bun >> pnpm > yarn berry > yarn classic ≈ npm
설치 속도 (warm):  bun >> pnpm > yarn berry >> yarn classic ≈ npm
디스크 사용량:     yarn PnP < pnpm << npm ≈ yarn classic ≈ bun
```

- bun: Zig 네이티브 바이너리라 I/O 자체가 빠름
- pnpm: content-addressable store로 중복 제거
- yarn PnP: zip 압축 + node_modules 없음

## 어떤 걸 선택할까

| 상황 | 추천 | 이유 |
|------|------|------|
| 새 프로젝트, 안정성 우선 | npm | 생태계 호환성 최고, 별도 설치 불필요 |
| 모노레포 | pnpm | 엄격한 의존성 격리 + 디스크 절약 + 빠른 설치 |
| 대규모 모노레포 (Meta 스타일) | yarn berry | PnP로 극한의 성능, zero-install 가능 |
| 빠른 프로토타이핑 | bun | 압도적 설치 속도, 런타임 겸용 |
| 기존 프로젝트 | 현재 사용 중인 것 유지 | 마이그레이션 비용 > 이점인 경우가 많음 |

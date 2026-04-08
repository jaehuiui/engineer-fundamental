# Frontend Engineer Fundamentals

## Project Purpose
프론트엔드 엔지니어 핵심 지식을 정리하는 학습 노트 레포지토리.

## Language Rules
- 제목, 키워드, 코드 용어: English
- 설명, 본문: 한국어
- 코드 주석: English
- 중요한 개념/용어는 한글과 영어를 병기 (예: "패킷 교환 (Packet Switching)", "신뢰성 (Reliability)")
- 영어 표현도 함께 이해할 수 있도록 핵심 문장이나 정의는 영어 원문도 포함

## Folder Structure
- 질문 시 필요한 폴더를 그때그때 생성 (Minimal 방식)
- 주제별 폴더: `network/`, `js/`, `ts/`, `node/`, `react/`, `browser/` 등
- 각 폴더 하위에 주제별 `.md` 파일 생성

## Document Rules
- 모든 문서는 Markdown 형식
- 파일명은 kebab-case 영어 (예: `event-loop.md`, `tcp-handshake.md`)
- 새 파일 생성 시 폴더 내 학습 순서를 고려하여 번호 prefix 부여 (예: `01-network-fundamentals.md`, `02-internet-communication.md`)
- 번호는 단순 추가 순서가 아닌, 해당 폴더 내에서 읽어야 하는 학습 순서 기준으로 결정 (기초 → 심화)
- 새 파일이 기존 파일들 사이에 들어가야 한다면, 기존 파일들의 번호를 리넘버링하여 순서를 맞춤
- 각 문서 상단에 `# Title` 포함
- 코드 예제는 가능한 한 포함

## Answer Style
- 질문에 대해 먼저 대화체로 답변한 뒤, 학습 노트에 정리
- 답변 시 "왜?"를 중심으로 설명 — 단순 나열보다 이유와 맥락을 우선
- 관련 개념 간 비교표, 다이어그램(ASCII), 코드 예시를 적극 활용
- 후속 질문이 나올 만한 내용은 선제적으로 포함
- 기존 노트와 연관되는 내용이면 기존 문서에 섹션 추가, 독립적이면 새 문서 생성

## Commit Rules
- 커밋 메시지는 영어로 작성
- prefix: `day-N:` 형식 (1일 1커밋 기준)
- 본문에 다룬 주제를 카테고리별로 구체적으로 나열

## Workflow
1. 사용자가 질문
2. 답변 제공 (대화체, 한글 + 영어 병기)
3. 학습 노트 정리 전, 어떤 문서에 추가할지 / 새 문서나 폴더를 생성할지 사용자에게 확인
4. 사용자 확인 후 해당 내용을 적절한 폴더/파일에 학습 노트로 정리
5. 기존 문서가 있으면 업데이트, 없으면 새로 생성

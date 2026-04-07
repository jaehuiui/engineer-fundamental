# Frontend Engineer Fundamentals

## Project Purpose
프론트엔드 엔지니어 핵심 지식을 정리하는 학습 노트 레포지토리.

## Language Rules
- 제목, 키워드, 코드 용어: English
- 설명, 본문: 한국어
- 코드 주석: English

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

## Workflow
1. 사용자가 질문
2. 답변 제공
3. 해당 내용을 적절한 폴더/파일에 학습 노트로 정리
4. 기존 문서가 있으면 업데이트, 없으면 새로 생성

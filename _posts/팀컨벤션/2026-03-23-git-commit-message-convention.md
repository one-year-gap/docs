---
title: Git Commit Message Convention
date: 2026-03-02
conv_type: 커밋 메시지 규칙
author: 허영현
tags:
  - 깃
  - 커밋
---

일관된 커밋 히스토리 관리를 위해 아래의 규칙을 엄격하게 준수합니다. 규칙에 어긋나는 커밋은 **Husky**에 의해 자동으로 차단됩니다.

## 1. Commit Message 구조

커밋 메시지는 타입(Type)과 본문으로 구성되며, 콜론(`:`) 뒤에는 반드시 **한 칸의 공백**을 둡니다.

\*\*Format:\*\* `{Type} {Emoji} {type}: {Commit Message}`

\*\*Example:\*\* `✨ feat: 회원가입 API 로직 구현` (Jira 티켓 번호는 자동 할당)

## 2. Commit Type (작업 종류)

| Group | Label | Color | Description | 적용 예시 |
| --- | --- | --- | --- | --- |
| Type | ✨ feat | a2eeef | 새로운 기능 추가, 기존 기능을 요구 사항에 맞추어 수정 | 신규 API 추가, 기능 플로우 확장 |
| Type | 🐛 fix | d73a4a | 기능에 대한 버그 수정 | NPE 수정, 잘못된 계산 로직 수정 |
| Type | ♻️ refactor | f29a4e | 기능 변화가 아닌 코드 리팩터링 | 클래스 분리, 중복 제거, 구조 개선 |
| Type | 🎨 style | FEF2C0 | 코드 스타일/포맷팅 수정(동작 변화 없음) | formatter 적용, import 정리 |
| Type | ✅ test | ccffc4 | 테스트 코드 추가/수정 | 단위/통합 테스트 보강 |
| Type | 📕 docs | 1D76DB | 문서 추가/수정/삭제(README 포함) | README 업데이트, ADR 작성 |
| Type | 🧹 chore | ededed | 잡일/설정 변경(빌드 스크립트, 포맷터, .gitignore 등) | lint 규칙 변경, scripts 정리 |
| Type | 🔧 config | C5DEF5 | 런타임/환경 구성 변경(환경변수, 설정파일, 프로파일 등) | application.yml 수정, profile 추가 |
| Type | 📦 deps | BFDADC | 의존성 추가/업데이트/정리(패키지 매니저 변경 포함) | spring 버전 업데이트 |
| Type | 🏗️ build | D4C5F9 | 빌드 시스템/CI 파이프라인 관련 변경 | Gradle 설정, Actions 수정 |
| Type | 🚀 deploy | 0E8A16 | 배포/런칭 작업(배포 스크립트, 롤백 포함) | 배포 스크립트/환경 배포 |
| Type | 🏷️ release | 5319E7 | 릴리즈 준비/버전 태깅/릴리즈 노트 | v1.2.0 태그, 릴리즈 노트 |
| Type | 🚑 hotfix | B60205 | 프로덕션 긴급 수정(우회/긴급 패치 포함) | 운영 장애 즉시 패치 |
| Type | 🛡️ security | 8B0000 | 보안 관련 변경(인증/인가, 취약점 대응, 비밀키/권한) | 권한 체크, 취약점 패치 |
| Type | ⚡ perf | FBCA04 | 성능 개선(쿼리/캐시/병목 제거 등) | 인덱스 튜닝, 캐시 도입 |
| Type | 🧪 spike | C2E0C6 | 조사/PoC/실험(결론 및 기록 필요) | 기술 검증, PoC 결과 문서화 |

## 3. 보조 라벨 가이드 (Issue & PR)

### 우선순위 (Priority)

| Group | Label | Color | Description | 기준 |
| --- | --- | --- | --- | --- |
| Priority | 🔥 priority: P0 | B60205 | 즉시 처리 필요(서비스/데모 블로커) | 장애/배포 차단/데모 불가 |
| Priority | ⚠️ priority: P1 | D93F0B | 빠른 처리 필요(주요 기능 영향) | 핵심 기능 영향/고객 영향 큼 |
| Priority | 📝 priority: P2 | FBCA04 | 일반 우선순위(스프린트 내 처리) | 일반 개선/기능 작업 |
| Priority | 🧊 priority: P3 | 0E8A16 | 여유 있을 때(백로그) | 장기 개선/정리성 작업 |

### 영역 (Area)

| Group | Label | Color | Description | 적용 기준 |
| --- | --- | --- | --- | --- |
| Area | 🗂️ area: BE | C5DEF5 | 백엔드 영역 | API/서버/도메인 로직 |
| Area | 🖥️ area: FE | C5DEF5 | 프론트엔드 영역 | UI/클라이언트 |
| Area | 🗄️ area: DB | C5DEF5 | 데이터베이스/쿼리/마이그레이션 영역 | SQL/인덱스/마이그레이션 |
| Area | ☁️ area: INFRA | C5DEF5 | 인프라/운영/배포 영역 | CI/CD, Terraform, 배포 |
| Area | 🔐 area: AUTH | C5DEF5 | 인증/인가(Cognito/OAuth/JWT) 영역 | 로그인/권한/토큰 |

## 4. Husky Setup 가이드

로컬 환경에서 개발을 시작하기 전, 반드시 아래 명령어를 통해 Husky 훅을 세팅해야 합니다.

```plain
# 기본 세팅 (package-lock.json 기반)
$ npm ci

# 프론트엔드 (pnpm 사용 시)
$ pnpm install

# Husky 훅 활성화
$ npm run prepare
```

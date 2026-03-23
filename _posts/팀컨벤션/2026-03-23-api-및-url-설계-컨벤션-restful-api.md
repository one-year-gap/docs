---
title: API 및 URL 설계 컨벤션 (RESTful API)
date: 2026-02-19
conv_type: API 네이밍 규칙
author: 허영현
tags:
  - API
  - URL
---

프론트엔드와 백엔드 간의 명확하고 예측 가능한 통신을 위한 API 설계 표준입니다.

## 1. Base URL

- **고객용 API:** `https://api.holliverse.site`
- **관리자용 API:** `https://api-admin.holliverse.site`

## 2. URL 설계 원칙 (Resource Naming)

> **기본 구조:** `{{baseUrl}}/api/v1/{resource}`

- **버전 명시:** URL 경로에 반드시 API 버전(`/v1/`)을 포함합니다.
- **Kebab-case:** URL 경로의 단어 조합은 케밥케이스를 사용합니다. `ex) /user-profiles`
- **명사 & 복수형 지향:** 행위(Verb) 대신 자원(Noun) 중심의 복수형 명사를 사용합니다.
    - ✅ `GET /api/v1/customers/1024/persona`
    - ❌ `GET /api/v1/getPersona/1024`

## 3. HTTP 메소드 규격

| **메소드** | **용도** | **프로젝트 적용 예시** |
| --- | --- | --- |
| **GET** | 리소스 조회 | 요금제 리스트, 고객 페르소나 캐릭터 조회 |
| **POST** | 리소스 생성 / AI 연산 | 회원가입, **상담 로그 전송 (Reflection)** |
| **PUT** | 리소스 전체 수정 | 고객 프로필 정보 전체 업데이트 |
| **PATCH** | 리소스 일부 수정 | 특정 태그 가중치 수동 조정 |
| **DELETE** | 리소스 삭제 | 상담 이력 삭제 |

## 4. 데이터 포맷 (Request & Response)

모든 JSON의 Key 값은 \*\*카멜케이스(camelCase)\*\*를 사용합니다. 응답 데이터는 성공/실패 여부에 관계없이 항상 동일한 Wrapper 구조를 유지하여 클라이언트의 파싱을 돕습니다.

**✅ 성공 응답 (Success)**

JSON

```plain
{
  "status": "success",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": { 
    // 실제 비즈니스 결과물 데이터
  },
  "timestamp": "2026-02-10T17:50:00Z",
  "requestId": "req-1234abcd"
}

```

**❌ 실패 응답 (Error)**

JSON

```plain
{
  "status": "error",
  "message": "에러 메시지",
  "errorDetail": {
    "code": "INVALID_INPUT",
    "field": "phone",
    "reason": "숫자만 입력 가능합니다."
  },
  "timestamp": "2026-02-10T17:50:05Z",
  "requestId": "req-1234abcd"
}

```

## 5. HTTP 상태 코드 (Status Codes)

| **코드** | **의미** | **프로젝트 내 상황** |
| --- | --- | --- |
| **200** | OK | 조회 및 일반적인 성공 |
| **201** | Created | 회원가입, 상담 로그 등록 완료 |
| **400** | Bad Request | 파라미터 누락, 유효성 검사 실패 |
| **401** | Unauthorized | 토큰 만료 또는 인증 실패 |
| 403 | Forbidden | 로그인 했지만 권한 없음(관리자 API 접근 등) |
| **404** | Not Found | 존재하지 않는 고객 또는 요금제 ID |
| 409 | Conflict | 중복(회원가입 중복) |
| **500** | Internal Error | **Gemini API(AI)** 장애, DB 서버 오류 |

## 6. 공통 응답 필드 명세 (Common Response Fields)

| **항목** | **필드명** | **설명** |
| --- | --- | --- |
| **관리 정보** | `requestId` | **[Required]** 요청 고유 식별자 (추적용) |
| **결과 상태** | `status` | **[Required]** success / error |
| **실제 데이터** | `data` | 요금제 리스트, 상세 정보 등 (비즈니스 결과물) |
| **발생 시점** | `timestamp` | 서버 응답 시간 |

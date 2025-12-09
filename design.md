# TO-DO 리스트 웹앱 설계 문서 (초안)

## 1. 요구사항 정리

- **기능**
  - TO-DO 추가 / 삭제
  - 지연(마감일 초과) 시 지연 표시
  - 완료 시 해당 항목 줄긋기(완료 시각 기록 가능)
- **입력 항목**
  - 제목
  - 내용
  - 완료(마감) 날짜
  - 진행 여부: `예정`, `진행중`, `완료`
- **조회**
  - 날짜별 리스트 조회 (예: 특정 날짜, 기간별)
  - 진행 상태별 리스트 조회 (예정/진행중/완료 필터)
- **참여자 기능**
  - TO-DO에 참여자 여러 명 추가 가능
  - 참여자별로 할당된 TO-DO만 필터링해서 보기
  - 참여자 본인이 포함된 TO-DO 목록 조회
- **데이터**
  - 위 요구사항을 모두 표현할 수 있는 데이터 모델 필요

---

## 2. 기술 스택 제안

### 2.1 프론트엔드(클라이언트)

- **프레임워크**
  - React + TypeScript
- **UI 라이브러리**
  - Tailwind CSS 또는 MUI(Material UI)
- **상태 관리**
  - React Query (서버 상태 관리) + 간단한 전역은 Context API

### 2.2 백엔드

- **언어 / 런타임**
  - Node.js (LTS)
- **프레임워크**
  - NestJS 또는 Express
  - 구조화/유지보수성을 고려하면 **NestJS** 추천
- **API 스타일**
  - REST API (JSON)

### 2.3 데이터베이스

- **로컬 테스트 용이성 기준**
  - SQLite (파일 기반, 설정 간단)
- **ORM**
  - Prisma 또는 TypeORM
  - NestJS와의 궁합까지 고려 시 **Prisma + SQLite** 추천

---

## 3. 전체 아키텍처

- **클라이언트(React SPA)**
  - 브라우저에서 동작
  - 백엔드 REST API 호출하여 TO-DO/사용자/참여자 데이터 처리
- **백엔드(NestJS)**
  - REST 엔드포인트 제공
  - 인증/인가(추가하려면 JWT 등) 계층
  - 서비스 계층에서 비즈니스 로직 처리 (지연 여부 계산, 필터링 등)
  - ORM을 통해 DB 접근
- **데이터베이스(SQLite)**
  - 단일 `.db` 파일
  - 개발/테스트 환경에서 간편히 사용

---

## 4. 도메인 모델 설계

### 4.1 엔티티 개요

- **User (사용자)**
  - 로그인/식별의 주체 (추후 확장 고려)
  - TO-DO의 생성자 또는 담당자
- **Participant (참여자)**
  - User와 Todo의 관계를 나타내는 중간 테이블
  - 하나의 TO-DO에 여러 명, 한 명이 여러 TO-DO에 참여 가능 (N:M)
- **Todo (할 일)**

### 4.2 필드 설계

#### 4.2.1 User

- `id` (PK, UUID 또는 auto-increment)
- `name` (표시 이름)
- `email` (로그인을 고려한다면 unique)
- `created_at`
- `updated_at`

#### 4.2.2 Todo

- `id` (PK)
- `title` (제목)
- `description` (내용, text)
- `due_date` (완료(마감) 날짜, DateTime)
- `status` (진행 여부)
  - enum: `PLANNED`(예정), `IN_PROGRESS`(진행중), `DONE`(완료)
- `is_delayed` (지연 여부, boolean)
  - 저장 시, 또는 조회 시 `now > due_date AND status != DONE` 조건으로 계산 가능
- `creator_id` (작성자 User FK)
- `created_at`
- `updated_at`
- `completed_at` (완료된 시각, status = DONE일 때 설정)

#### 4.2.3 TodoParticipant (참여 관계)

- `id` (PK)
- `todo_id` (FK → Todo)
- `user_id` (FK → User)
- `role` (optional, ex: `OWNER`, `ASSIGNEE`, `VIEWER`)
- `created_at`

---

## 5. 주요 기능 상세 설계

### 5.1 TO-DO 추가

- **입력**
  - 제목, 내용, 완료날짜, 진행상태(기본값: `예정`)
  - 선택: 참여자 목록 (User id 리스트)
- **처리**
  - Todo 생성
  - 참여자 목록이 있다면 `TodoParticipant`에 N개 row 생성
- **출력**
  - 생성된 Todo 정보 + 참여자 정보

### 5.2 TO-DO 삭제

- **처리**
  - 물리 삭제 또는 논리 삭제(soft delete) 중 선택
  - 단순 구현: 물리 삭제 (Todo / TodoParticipant 관계 함께 삭제)
- **주의**
  - 실제 서비스에서는 soft delete를 고려

### 5.3 지연 표시

- **지연 기준**
  - `now > due_date` AND `status != DONE`
- **구현 방식**
  - DB 컬럼 `is_delayed`를 두고 저장/업데이트마다 계산
  - 또는 API 응답 때 서버에서 동적으로 계산해서 내려주기
- **프론트**
  - `is_delayed` true이면 리스트에서 빨간색 태그/아이콘 등 표시

### 5.4 완료 처리(줄긋기)

- **처리**
  - `status`를 `DONE`으로 업데이트
  - `completed_at`에 현재 시각 기록
- **프론트**
  - `status === DONE`이면 텍스트에 `text-decoration: line-through` 스타일 적용

### 5.5 필터링 조회

- **날짜별**
  - 특정 날짜: `due_date`가 해당 날짜인 Todo 필터
  - 기간별: `due_date` BETWEEN `from` AND `to`
- **진행상태별**
  - `status` IN (선택된 상태 리스트)
- **참여자별**
  - `TodoParticipant.user_id` = 특정 사용자 id 조건으로 조인
  - ex. `/todos?participantId=123`

---

## 6. 화면(UI) 설계

### 6.1 메인 TO-DO 리스트 화면

- **구성 요소**
  - 상단 필터 영역
    - 날짜(단일 날짜 또는 기간 선택)
    - 진행 상태 체크박스(예정/진행중/완료)
    - 참여자 선택 드롭다운
  - TO-DO 리스트
    - 각 항목: 제목, 마감일, 상태, 지연 여부, 참여자 간단 표시(예: 아바타 + “+N”)
    - 지연 시 붉은 태그, 완료 시 줄긋기
  - 새 TO-DO 추가 버튼 (우측 하단 Floating Button 등)

### 6.2 TO-DO 생성/수정 모달/페이지

- **입력 폼**
  - 제목 (필수)
  - 내용 (텍스트 영역)
  - 완료(마감) 날짜 (DatePicker)
  - 진행상태 select (예정/진행중/완료)
  - 참여자 멀티 셀렉트 (User 리스트)
- **동작**
  - 생성/수정 후 목록 재조회

### 6.3 TO-DO 상세 화면 (선택)

- TO-DO 상세 정보, 참여자 목록, 상태 변경 버튼 등
- 필요 시 코멘트 기능 등 추가 확장 가능

---

## 7. API 설계 (REST 대략안)

### 7.1 Todo 관련

- **POST `/api/todos`**
  - body: `{ title, description, dueDate, status, participantIds[] }`
  - res: 생성된 Todo
- **GET `/api/todos`**
  - query:
    - `status` (단일 또는 콤마 구분 리스트)
    - `fromDate`, `toDate`
    - `participantId`
  - res: Todo 리스트
- **GET `/api/todos/:id`**
  - 단일 Todo + 참여자
- **PATCH `/api/todos/:id`**
  - body: 부분 업데이트 (title, description, dueDate, status, participantIds 등)
- **DELETE `/api/todos/:id`**
  - Todo 삭제

### 7.2 User/Participant 관련 (간단 버전)

- **GET `/api/users`**
  - 참여자 선택용 사용자 목록
- 참여자 추가/삭제는 `PATCH /api/todos/:id`에서 `participantIds` 전체 교체 방식으로 처리

---

## 8. 비즈니스 로직 정리

- **지연 여부 계산**
  - 서버 공통 함수
    - `isDelayed = now > dueDate && status != DONE`
  - Todo 생성/수정 시 재계산
- **상태 전이 예시**
  - `PLANNED → IN_PROGRESS → DONE`
  - 필요시 강제 전이 허용 여부 규칙 추가 가능
- **참여자 권한 (추후 확장)**
  - 참여자만 수정 가능, 조회 권한 제한 등

---

## 9. 개발/테스트 환경

- **DB**
  - 로컬: `SQLite` 파일 (`todo_app.db`)
- **실행**
  - 백엔드: `npm run start:dev`
  - 프론트: `npm run dev`
- **테스트**
  - 기본 단위 테스트: 서비스 레벨 (지연 계산, 상태 전이, 필터링)
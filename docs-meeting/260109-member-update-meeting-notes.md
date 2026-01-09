# 회원 정보 수정 기능 논의 노트
**날짜:** 2026-01-09

## 논의 내용

### 1. 수정 가능 항목

**사용자 수정 가능:**
- name, description, skills, links, image_url

**사용자 수정 불가 (read-only 표시):**
- email (이메일)
- generation (기수)
- rank (직급)

**관리자 수정 가능:**
- 모든 항목 (email, generation, rank 포함)

### 2. 인증 방식

매직링크 기반 인증:
1. 사용자가 "프로필 수정" 페이지 접속
2. 이메일 입력 후 "인증 링크 받기" 클릭
3. 이메일로 매직링크 발송 (URL: `?token=xxx`)
4. 링크 클릭 시 기존 정보가 채워진 수정 폼 표시
5. 수정 후 제출

**보안:**
- 토큰 유효기간: 30분
- 토큰 purpose: "profile_update"
- 토큰 이메일과 회원 이메일 일치 검증
- APPROVED 상태의 회원만 수정 가능

### 3. Skills/Links 수정 방식

새 리스트로 완전 교체 (전체 삭제 후 재삽입)

### 4. 관리자 기능

- 별도 관리자 페이지에서 회원 편집
- 기존 TOTP + X-Admin-Key 인증 활용
- 모든 필드 수정 가능

**상태:** 현재 단계에서는 구현하지 않음 (다음 단계로 보류)

---

## System Sequence Diagram

### 사용자 프로필 수정 (확정)

```mermaid
sequenceDiagram
    actor User
    participant Frontend as User Frontend
    participant Auth as Auth Router
    participant Member as Member Service
    participant DB as Database
    participant Email as Email Service

    Note over User, Email: Phase 1: 인증 요청
    User->>Frontend: 프로필 수정 페이지 접속
    User->>Frontend: 이메일 입력 후 "인증 링크 받기"
    Frontend->>Auth: POST /auth/magic-link/profile-update
    Auth->>Member: request_profile_update(email)
    Member->>Email: send_magic_link(email, magic_link_url)
    Email-->>User: 매직링크 이메일 발송

    Note over User, DB: Phase 2: 인증 및 데이터 로드
    User->>Frontend: 매직링크 클릭 (URL: ?token=xxx)
    Frontend->>Auth: GET /auth/verify-profile-update?token=xxx
    Auth->>Member: verify_profile_update_token(token)
    Member->>DB: 회원 조회 (이메일 기반)
    DB-->>Member: 회원 데이터
    Member-->>Auth: 회원 데이터 반환
    Auth-->>Frontend: 회원 정보
    Frontend-->>User: 기존 정보가 채워진 수정 폼

    Note over User, DB: Phase 3: 수정 제출
    User->>Frontend: 정보 수정 후 "프로필 수정"
    Frontend->>Member: PUT /members/{id}?token=xxx
    Member->>Member: 토큰 검증 및 이메일 일치 확인
    Member->>DB: update_member() - skills/links 교체
    DB-->>Member: 업데이트 완료
    Member-->>Frontend: 성공
    Frontend-->>User: 완료 메시지
```

### 관리자 프로필 수정 (보류)

> **참고:** 관리자 프로필 수정 기능은 현재 단계에서 구현하지 않고 다음 단계로 보류합니다.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminFE as Admin Frontend
    participant Members as Members Router
    participant Member as Member Service
    participant DB as Database

    Note over Admin, MemberFE: Phase 1: 회원 선택
    Admin->>AdminFE: TOTP 인증
    Admin->>AdminFE: "회원 관리" 페이지
    AdminFE->>Members: GET /members?status=all
    Members->>Member: get_all_members()
    Member-->>Members: 회원 목록
    Members-->>AdminFE: 회원 목록
    Admin->>AdminFE: "편집" 버튼 클릭

    Note over AdminFE, DB: Phase 2: 편집 폼
    AdminFE->>Members: GET /members/{id}
    Members->>Member: get_member_by_id()
    Member-->>Members: 회원 상세 정보
    Members-->>AdminFE: 회원 정보
    AdminFE-->>Admin: 편집 폼 (모든 필드)

    Note over Admin, DB: Phase 3: 수정 저장
    Admin->>AdminFE: 수정 후 "저장"
    AdminFE->>Members: PUT /admin/members/{id} (X-Admin-Key)
    Members->>Member: update_member()
    Member->>DB: update_member() - 이메일 중복 검사
    DB-->>Member: 완료
    Member-->>Members: 업데이트된 회원
    Members-->>AdminFE: 성공
    AdminFE-->>Admin: 완료 후 목록 이동
```

---

## 주요 API 엔드포인트

### 신규 추가
- `GET /auth/verify-profile-update?token=xxx`
- `PUT /admin/members/{id}`

### 기존 활성화
- `PUT /members/{id}?token=xxx` (현재 501 NOT_IMPLEMENTED)

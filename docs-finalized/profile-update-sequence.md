# 회원 정보 수정 시스템 시퀀스

## 사용자 프로필 수정 (확정)

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

---

## 관리자 프로필 수정 (보류)

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

## 비고

### 사용자 수정 제한 항목
- **email**: 변경 불가
- **generation**: 변경 불가
- **rank**: 변경 불가 (관리자만 수정 가능)

### 관리자 수정 가능 항목
- 모든 항목 수정 가능 (email, generation, rank 포함)

### 기술 스택/링크 수정 방식
- **완전 교체**: 새 리스트로 기존 데이터 전체 교체

### 보안 장치
1. **토큰 기반 인증**: 30분 유효기간
2. **이메일 검증**: 토큰 이메일 == 회원 이메일 확인
3. **상태 검증**: APPROVED 회원만 수정 가능
4. **관리자 인증**: TOTP + X-Admin-Key

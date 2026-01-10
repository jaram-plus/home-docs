# 프로필 수정 기능 구현 미팅 노트

**날짜:** 2026년 1월 10일
**참여자:** 개발팀
**주제:** 회원 프로필 수정 기능 구현 완료

## 배경

기존 회원가입 기능 이후, 회원들이 자신의 프로필 정보를 수정할 수 있는 기능이 필요함. 문서(`docs-finalized/profile-update-sequence.md`)에 정의된 요구사항에 따라 JWT 기반 매직 링크 인증 시스템을 활용한 프로필 수정 기능을 구현함.

## 구체화된 사항

### 1. 인증 시스템

**JWT 토큰 목적(Purpose) 분리:**
- 기존: `purpose="registration"` (회원가입용)
- 신규: `purpose="profile_update"` (프로필 수정용)

**보안 요구사항:**
- 승인된(APPROVED) 회원만 프로필 수정 가능
- 토큰 유효기간: 30분
- 토큰은 이메일로만 발송 (비밀번호 없는 인증)

### 2. API 엔드포인트 구조

**이메일 링크용 (HTML 리다이렉트):**
```
GET /auth/verify-profile-update?token=xxx&redirect=yyy
```
- 이메일에서 클릭 시 호출
- HTMLResponse 반환 (meta refresh + JavaScript redirect)
- 사용자를 Streamlit 프론트엔드로 리다이렉트

**프론트엔드 API용 (JSON 응답):**
```
GET /auth/verify-profile-update-json?token=xxx
```
- 프론트엔드에서 직접 호출
- MemberResponse JSON 반환
- 회원 데이터 조회 및 인증 상태 확인

**프로필 수정 (활성화):**
```
PUT /members/{member_id}?token=xxx
```
- 기존 501 NOT IMPLEMENTED → 활성화
- 토큰 검증: 토큰의 이메일과 회원 ID 매칭
- 본인의 프로필만 수정 가능

### 3. 변경 불가 필드 (Immutable)

문서와 일치하게 다음 필드는 수정 불가:
- **이메일**: 고유 식별자
- **기수 (Generation)**: 가입 시 결정
- **계급 (Rank)**: 승인 프로세스에 따름

**수정 가능 필드:**
- 이름
- 자기소개 (Description)
- 프로필 이미지 URL
- 기술 스택 (Skills)
- 링크 (Links: GitHub, LinkedIn, Blog 등)

### 4. Streamlit 프론트엔드 구조

**MultiPage 앱 구조:**
```
user_frontend/
├── app.py                      # 메인 페이지 (환영 메시지, 토큰 리다이렉트)
└── pages/
    ├── 01_회원가입.py          # 회원가입 페이지
    └── 02_프로필_수정.py        # 프로필 수정 페이지
```

**페이지 이동 흐름:**
1. 이메일 링크 클릭 → `http://localhost:8501?token=xxx`
2. `app.py`에서 토큰 감지 → `st.session_state.profile_token`에 저장
3. `st.switch_page("pages/02_프로필_수정.py")`로 자동 이동
4. 프로필 수정 페이지에서 토큰으로 자동 인증
5. 기존 데이터가 입력된 폼 표시

**Form 제약사항 해결:**
- `st.form()` 안에서 `st.button()` 사용 불가
- 해결: 성공 상태를 `session_state`에 저장
- Form 밖에서 성공 메시지와 "홈으로 가기" 버튼 표시

## 변동된 사항

### 1. Schema 변경

**MemberUpdate (schemas/member.py):**
```python
# 제거됨
rank: MemberRank | None = None  # Immutable

# 유지됨
name: str | None = None
description: str | None = None
image_url: str | None = None
skills: list[SkillCreate] | None = None
links: list[LinkCreate] | None = None
```

### 2. Repository 로직 변경

**member_repository.py:**
- `rank` 필드 업데이트 로직 제거
- 명시적 주석 추가: `# rank, email, generation cannot be updated`

### 3. Service Layer 추가

**MemberService (services/member_service.py):**
- `verify_profile_update_token()`: 프로필 수정 토큰 검증
- `_build_magic_link_url()`: `endpoint` 파라미터 추가 (유연성 확보)
- `request_profile_update()`: 프로필 수정용 매직 링크 생성

### 4. Docker 포트 노출

**이전:** 포트가 컨테이너 내부에서만 접근 가능
**변경:** localhost에서 접근 가능하도록 포트 매핑 추가

```yaml
services:
  member-service:
    ports:
      - "8000:8000"
  user-frontend:
    ports:
      - "8501:8501"
  admin-frontend:
    ports:
      - "8502:8501"
```

### 5. 이메일 서비스 구성

**개발 환경:**
- Provider: Mock (로그만 출력)
- 실제 이메일 발송 안 함
- 터미널에서 매직 링크 URL 복사 사용

**프로덕션 환경 (향후):**
- Provider: Resend
- 환경변수 설정 필요:
  - `EMAIL_PROVIDER=resend`
  - `RESEND_API_KEY=your-key`
  - `EMAIL_FROM=Jaram <team@jaram.net>`

## 기술적 해결 과정

### Issue 1: JSON 파싱 에러
**문제:** `verify_profile_update_token()`이 HTML을 JSON으로 파싱 시도
```
Expecting value: line 2 column 9 (char 9)
```
**원인:** `/auth/verify-profile-update`가 HTMLResponse 반환
**해결:** JSON용 별도 엔드포인트 `/auth/verify-profile-update-json` 생성

### Issue 2: Streamlit Form 내 Button 사용 불가
**문제:** `st.form()` 안에서 `st.button()` 사용 시 에러
```
st.button() can't be used in an st.form()
```
**해결:**
- Form 제출 후 `session_state.profile_update_success = True`
- `st.rerun()`으로 페이지 재렌더링
- Form 밖에서 성공 메시지와 버튼 표시

### Issue 3: Query Parameter 유지 안 됨
**문제:** `st.switch_page()` 호출 시 query parameter 소실
**해결:** `st.session_state.profile_token`에 저장하여 전달

## 완료된 기능

### ✅ 백엔드 (FastAPI)
- [x] JWT 토큰 목적(Purpose) 분리
- [x] `/auth/magic-link/profile-update` (POST)
- [x] `/auth/verify-profile-update` (GET, HTML)
- [x] `/auth/verify-profile-update-json` (GET, JSON)
- [x] `PUT /members/{id}` 활성화
- [x] 토큰 검증 및 회원 ID 매칭
- [x] Immutable 필드 적용 (email, generation, rank)

### ✅ 프론트엔드 (Streamlit)
- [x] 프로필 수정 페이지 구현
- [x] 매직 링크 자동 인증
- [x] 기존 데이터가 입력된 폼
- [x] 변경 불가 필드 비활성화 표시
- [x] 성공 메시지 및 네비게이션
- [x] 에러 처리 (만료된 토큰, 미승인 회원 등)

### ✅ 인프라
- [x] Docker Compose 포트 매핑
- [x] 환경변수 구성
- [x] Mock 이메일 서비스

## 다음 단계 (향후 작업)

1. **이메일 서비스 프로덕션 설정**
   - Resend API 키 발급
   - `.env` 파일에 환경변수 설정
   - 실제 이메일 발송 테스트

2. **URL 인코딩 테스트**
   - 프로덕션 도메인으로 리다이렉트 테스트
   - HTTPS 설정 확인

3. **UI/UX 개선**
   - 로딩 상태 개선
   - 에러 메시지 구체화
   - 접근성 향상

4. **테스트 코드 작성**
   - API 엔드포인트 테스트
   - 인증 흐름 테스트
   - 경계 케이스 테스트

## 참고 문서

- [프로필 수정 시퀀스](../docs-finalized/profile-update-sequence.md)
- [API 문서](../docs-finalized/api-endpoints.md)
- [데이터베이스 스키마](../docs-finalized/database-schema.md)

## 커밋 정보

- **Branch:** `conductor/baku-v1`
- **Commit:** `5140f20`
- **Message:** Implement profile update feature with magic link authentication
- **Files Changed:** 11 files, 681 insertions(+), 60 deletions(-)

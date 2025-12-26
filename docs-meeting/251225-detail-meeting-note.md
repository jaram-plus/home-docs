## database schema
* member 테이블
```sql
CREATE TABLE member (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    generation INTEGER NOT NULL,
    email TEXT NOT NULL UNIQUE,
    rank TEXT NOT NULL CHECK(rank IN ('정회원', 'OB', '준OB')),
    description TEXT,
    image_url TEXT,
    status TEXT NOT NULL DEFAULT 'unverified' CHECK(status IN ('unverified', 'pending', 'approved')),
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```
* member_skill 테이블
```sql
CREATE TABLE member_skill (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    member_id INTEGER NOT NULL,
    skill_name TEXT NOT NULL,
    FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE
);
```

* member_link 테이블
```sql
CREATE TABLE member_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    member_id INTEGER NOT NULL,
    link_type TEXT NOT NULL,
    url TEXT NOT NULL,
    FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE
);
```

## system sequence
1. 사용자가 웹의 회원가입 폼에서 이름, 기수, 이메일, 직급 등의 정보를 입력하고 제출 - Member Service의 `create_member` 호출
2. Member Service는 입력받은 정보를 바탕으로 Member Repository의 `add_member` 메서드를 호출하여 DB에 회원 정보 저장 (status는 'unverified'로 설정)
3. Member Service는 Email Service의 `send_verification_email` 메서드를 호출하여 검증 이메일 발송
4. 사용자가 이메일의 매직링크를 클릭하여 인증 - Auth Router의 `/verify` 엔드포인트 호출
5. Auth Router는 Token Utility의 `verify_token` 메서드를 사용하여 토큰 검증
6. 토큰이 유효하면 Member Service의 `approve_member` 메서드를 호출하여 해당 회원의 status를 'pending'로 업데이트
7. 관리자가 대시보드에서 회원 정보를 검토하고 승인 - Member Router의 `/members/{id}/approve` 엔드포인트 호출
8. Member Router는 Member Service의 `approve_member` 메서드를 호출하여 해당 회원의 status를 'approved'로 업데이트
9. 승인된 회원은 이제 회원 목록에 표시됨.
### 비고
* 관리자가 회원 가입을 거부할 경우, Member Service의 `reject_member` 메서드를 호출하여 해당 회원 정보를 DB에서 삭제함.

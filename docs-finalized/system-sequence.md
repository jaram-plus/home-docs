
## 회원 등록
1. 사용자가 웹의 회원가입 폼에서 이름, 기수, 이메일, 직급 등의 정보를 입력하고 제출 - Member Service의 `create_member` 호출
2. Member Service는 입력받은 정보를 바탕으로 Member Repository의 `add_member` 메서드를 호출하여 DB에 회원 정보 저장 (status는 'unverified'로 설정)
3. Member Service는 Email Service의 `send_verification_email` 메서드를 호출하여 검증 이메일 발송
4. 사용자가 이메일의 매직링크를 클릭하여 인증 - Auth Router의 `/verify` 엔드포인트 호출
5. Auth Router는 Token Utility의 `verify_token` 메서드를 사용하여 토큰 검증
6. 토큰이 유효하면 Member Service의 `approve_member` 메서드를 호출하여 해당 회원의 status를 'pending'로 업데이트
7. 관리자가 대시보드에서 회원 정보를 검토하고 승인 - Member Router의 `/members/{id}/approve` 엔드포인트 호출
8. Member Router는 Member Service의 `approve_member` 메서드를 호출하여 해당 회원의 status를 'approved'로 업데이트
9. 승인된 회원은 이제 회원 목록에 표시됨.



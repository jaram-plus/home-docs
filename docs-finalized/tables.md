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
    status TEXT NOT NULL DEFAULT 'UNVERIFIED' CHECK(status IN ('UNVERIFIED', 'PENDING', 'APPROVED')),
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
# 5강 — `CREATE TABLE`로 테이블 만들기

> **목표:** SQL로 테이블을 직접 설계하고 만든다.
> **선수 지식:** 1~4강

---

## 1. DDL이 뭔가?

SQL 명령은 **목적**에 따라 분류된다. 자주 듣게 될 약어들이라 한 번 정리하자.

| 약어 | 풀이 | 하는 일 | 대표 명령 |
|------|------|---------|----------|
| **DDL** | Data **Definition** Language | 테이블 **구조** 정의 | `CREATE`, `ALTER`, `DROP` |
| **DML** | Data **Manipulation** Language | 데이터 **조작** | `INSERT`, `UPDATE`, `DELETE`, `SELECT` |
| **DCL** | Data **Control** Language | 권한 제어 | `GRANT`, `REVOKE` |
| **TCL** | Transaction **Control** Language | 트랜잭션 제어 | `COMMIT`, `ROLLBACK` |

이번 강의는 **DDL**, 그중에서도 `CREATE TABLE`이다.

---

## 2. 기본 문법

```sql
CREATE TABLE 테이블이름 (
    열이름1  데이터타입,
    열이름2  데이터타입,
    열이름3  데이터타입
);
```

규칙:

- 열 정의는 **쉼표(`,`)** 로 구분.
- 마지막 열 뒤엔 쉼표 **없음.**
- 문장 끝엔 **세미콜론(`;`).**

### C++과 비교

```cpp
// C++의 struct
struct FUser
{
    int32   Id;
    FString Name;
    int32   Level;
};
```

```sql
-- PostgreSQL의 CREATE TABLE
CREATE TABLE users (
    id    INTEGER,
    name  TEXT,
    level INTEGER
);
```

타입 표기 순서가 반대다. C++은 `타입 이름`, SQL은 `이름 타입`. 헷갈리기 쉬우니 입에 익혀두자.

---

## 3. 실습 — 첫 테이블 만들기

`psql`에서 `study=#` 상태에서:

```sql
CREATE TABLE users (
    id    INTEGER,
    name  TEXT,
    level INTEGER,
    gold  INTEGER
);
```

성공하면:

```text
CREATE TABLE
```

확인:

```sql
\dt           -- 테이블 목록
\d users      -- users의 구조 보기
```

`\d users` 결과 예시:

```text
                Table "public.users"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 id     | integer |           |          |
 name   | text    |           |          |
 level  | integer |           |          |
 gold   | integer |           |          |
```

오, 잘 만들어졌다. 진짜 `public` 스키마에 들어갔다.

---

## 4. 자주 쓰는 데이터 타입 — 맛보기

6강에서 자세히 다루지만, 일단 자주 쓰는 것만.

| 타입 | 설명 | C++ 비유 |
|------|------|---------|
| `INTEGER` | 정수 (-21억 ~ +21억) | `int32` |
| `BIGINT`  | 큰 정수 (±9경) | `int64` |
| `NUMERIC(p, s)` | 정확한 소수 (돈 계산용) | (C++엔 없음) |
| `REAL` / `DOUBLE PRECISION` | 부동소수점 | `float` / `double` |
| `TEXT` | 가변 길이 문자열 | `FString` |
| `VARCHAR(n)` | 길이 제한 문자열 | `FString` (길이 제한 있음) |
| `BOOLEAN` | 참/거짓 | `bool` |
| `DATE` | 날짜 (2026-05-20) | (FDateTime 일부) |
| `TIMESTAMP` | 날짜+시간 | `FDateTime` |

> 💡 PostgreSQL에선 **문자열은 그냥 `TEXT` 쓰는 게 보통.** MySQL에서 넘어온 사람은 `VARCHAR(n)`을 습관처럼 쓰는데, PG에선 둘이 성능 차이 없고 `TEXT`가 더 깔끔하다.

---

## 5. 같은 이름 테이블 다시 만들기?

이미 `users` 테이블이 있는데 또 `CREATE TABLE users (...)` 하면:

```text
ERROR:  relation "users" already exists
```

해결책 두 가지.

### 방법 A — 안전하게: 있으면 그냥 두기

```sql
CREATE TABLE IF NOT EXISTS users (
    id   INTEGER,
    name TEXT
);
```

이미 있으면 에러 없이 그냥 넘어간다. **스크립트 재실행 시 유용.**

### 방법 B — 지우고 새로 만들기

```sql
DROP TABLE users;
CREATE TABLE users (...);
```

⚠️ `DROP`은 **데이터까지 다 날린다.** 학습 중엔 편하지만 실무에선 신중.

---

## 6. 옵션 한 줄로 살펴보기 — 제약조건 미리보기

`CREATE TABLE` 안에서 열에 **옵션(제약조건)** 을 붙일 수 있다. 자세한 건 21~23강에서, 여기선 모양만.

```sql
CREATE TABLE users (
    id    INTEGER     PRIMARY KEY,        -- 고유 식별자
    name  TEXT        NOT NULL,           -- NULL 금지
    level INTEGER     DEFAULT 1,          -- 기본값 1
    gold  INTEGER     DEFAULT 0 CHECK (gold >= 0),  -- 음수 금지
    email TEXT        UNIQUE              -- 중복 금지
);
```

이런 식으로 **데이터의 규칙**을 테이블 정의 단계에서 강제할 수 있다. C++로 치면 setter 안에서 `check(value >= 0)` 하는 걸 DB가 자동으로 해주는 셈.

---

## 7. 명명 규칙 — 실무 관례

문법은 아니지만 안 지키면 두고두고 후회한다.

| 권장 | 비권장 |
|------|-------|
| `users` (복수형, 소문자) | `User`, `USER` (대소문자 섞임) |
| `snake_case` (`user_items`) | `camelCase` (`userItems`) |
| 영어 단어 | 한글, 약어 남발 |
| 명확한 이름 (`order_items`) | `tbl1`, `data`, `temp` |

PostgreSQL은 **식별자를 큰따옴표로 감싸지 않으면 자동으로 소문자로 바꾼다.**

```sql
CREATE TABLE Users (...);  -- 실제로는 users로 저장됨
SELECT * FROM USERS;       -- 동작함 (소문자 변환)
```

그러니 **그냥 처음부터 소문자로 쓰는 게 마음 편하다.**

---

## 8. 실습 — 게임 DB 테이블 3개 만들기

다음 강의들에서 계속 쓸 테이블을 미리 만들어두자.

```sql
-- 기존 거 정리
DROP TABLE IF EXISTS users;
DROP TABLE IF EXISTS items;
DROP TABLE IF EXISTS guilds;

-- 길드
CREATE TABLE guilds (
    id         INTEGER,
    guild_name TEXT
);

-- 유저
CREATE TABLE users (
    id        INTEGER,
    name      TEXT,
    level     INTEGER,
    gold      INTEGER,
    guild_id  INTEGER     -- guilds.id를 가리킬 예정
);

-- 아이템
CREATE TABLE items (
    id        INTEGER,
    user_id   INTEGER,    -- users.id를 가리킬 예정
    item_name TEXT,
    quantity  INTEGER
);

-- 확인
\dt
\d users
```

> 일부러 제약조건은 빼두었다. 21강에서 다시 손볼 거다.

---

## 9. 이번 강의에서 기억할 것

1. **DDL은 구조 정의 / DML은 데이터 조작.** 우리가 다룬 건 DDL.
2. 문법: `CREATE TABLE 이름 ( 열이름 타입, ... );` — **이름이 먼저, 타입이 나중.**
3. `IF NOT EXISTS`로 중복 생성 방지 가능.
4. **테이블·열 이름은 소문자 + snake_case** — PostgreSQL 관례.
5. 열에 제약조건(`NOT NULL`, `DEFAULT`, `PRIMARY KEY` 등)을 같이 붙일 수 있다.

---

## 10. 다음 강의 예고

**6강 — 데이터 타입 심화**
오늘 살짝 본 타입들을 본격적으로 파헤친다. 정수/실수/문자열/날짜/불리언/그리고 PostgreSQL만의 특별한 타입들(`SERIAL`, `UUID`, `JSONB` 등) 까지.

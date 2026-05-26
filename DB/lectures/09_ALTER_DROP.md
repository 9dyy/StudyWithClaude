# 9강 — `ALTER TABLE`, `DROP TABLE`로 구조 바꾸기

> **목표:** 이미 만든 테이블의 구조를 바꾸거나 지운다.
> **선수 지식:** 5~8강

---

## 1. 왜 구조를 바꿔야 하나?

처음 설계가 완벽할 리 없다. 실제로 이런 일이 자주 일어난다.

- "유저 테이블에 이메일 컬럼이 빠졌네…"
- "닉네임을 `VARCHAR(20)`으로 했는데 부족하다, `TEXT`로 바꾸자."
- "테스트 테이블 다 지우고 새로 만들어야겠다."

C++로 비유하면 `struct` 멤버를 추가·제거·타입 변경하는 거다. 차이점은 **DB엔 이미 데이터가 들어 있다**는 것. 그래서 변경이 더 조심스럽다.

---

## 2. `ALTER TABLE` 한눈에 보기

`ALTER TABLE`의 기본형:

```sql
ALTER TABLE 테이블이름 동작;
```

자주 쓰는 "동작"들:

| 동작 | 하는 일 |
|------|---------|
| `ADD COLUMN 열 타입` | 열 추가 |
| `DROP COLUMN 열` | 열 제거 |
| `RENAME COLUMN 옛이름 TO 새이름` | 열 이름 변경 |
| `ALTER COLUMN 열 TYPE 새타입` | 열 타입 변경 |
| `ALTER COLUMN 열 SET DEFAULT 값` | 기본값 설정 |
| `ALTER COLUMN 열 DROP DEFAULT` | 기본값 제거 |
| `ALTER COLUMN 열 SET NOT NULL` | NOT NULL 부여 |
| `ALTER COLUMN 열 DROP NOT NULL` | NOT NULL 해제 |
| `RENAME TO 새이름` | 테이블 이름 변경 |
| `ADD CONSTRAINT ...` | 제약조건 추가 (21강~) |
| `DROP CONSTRAINT ...` | 제약조건 제거 |

---

## 3. 열 추가하기 — `ADD COLUMN`

```sql
ALTER TABLE users ADD COLUMN email TEXT;
```

이미 있는 행들의 `email`은 자동으로 **NULL**이 채워진다.

기본값을 같이 주면 그 값으로 채워짐:

```sql
ALTER TABLE users ADD COLUMN exp INTEGER DEFAULT 0;
```

여러 개 한 번에:

```sql
ALTER TABLE users
    ADD COLUMN phone TEXT,
    ADD COLUMN address TEXT;
```

> ⚠️ **`NOT NULL` 열을 기본값 없이 추가**하면 에러. 기존 행에 채울 값이 없기 때문.
> ```sql
> ALTER TABLE users ADD COLUMN nickname TEXT NOT NULL;  -- ❌
> ALTER TABLE users ADD COLUMN nickname TEXT NOT NULL DEFAULT 'noname'; -- ✅
> ```

---

## 4. 열 삭제하기 — `DROP COLUMN`

```sql
ALTER TABLE users DROP COLUMN address;
```

**열에 들어있던 데이터까지 다 사라진다.** 되돌릴 방법은 백업밖에 없으니 신중히.

연결된 제약조건이나 인덱스가 같이 사라지길 원할 땐:

```sql
ALTER TABLE users DROP COLUMN guild_id CASCADE;
```

`CASCADE`는 "줄줄이 따라 지움". 강력하니 평소엔 생략하고, 에러 메시지가 시킬 때만 쓰자.

---

## 5. 열 이름 바꾸기 — `RENAME COLUMN`

```sql
ALTER TABLE users RENAME COLUMN gold TO coin;
```

**데이터는 그대로**, 이름만 바뀐다.

> ⚠️ 다른 곳(앱 코드, 뷰, 다른 쿼리)에서 옛 이름을 쓰고 있으면 **죄다 깨진다.** 이름 변경은 의외로 영향 범위가 크다.

---

## 6. 열 타입 바꾸기 — `ALTER COLUMN ... TYPE`

```sql
ALTER TABLE users ALTER COLUMN level TYPE BIGINT;
```

PostgreSQL은 **자동으로 변환 가능하면** 알아서 처리한다.

자동이 안 되면 `USING`으로 변환식을 직접 지정:

```sql
-- TEXT를 INTEGER로 변환 (값이 모두 숫자 형태일 때만 성공)
ALTER TABLE users
ALTER COLUMN level TYPE INTEGER USING level::INTEGER;
```

`::타입` 은 **타입 캐스팅 연산자**. C++의 `static_cast`와 비슷한 PostgreSQL 문법.

```sql
SELECT '42'::INTEGER + 1;   -- 결과: 43
SELECT '2026-05-20'::DATE;  -- 텍스트 → 날짜
```

> 💡 운영 중인 큰 테이블의 타입 변경은 **느리고 위험**할 수 있다. 학습 단계에선 부담 없이 연습.

---

## 7. 기본값과 NULL 허용/금지 토글

```sql
-- 기본값 설정/해제
ALTER TABLE users ALTER COLUMN level SET DEFAULT 1;
ALTER TABLE users ALTER COLUMN level DROP DEFAULT;

-- NOT NULL 부여/해제
ALTER TABLE users ALTER COLUMN name SET NOT NULL;
ALTER TABLE users ALTER COLUMN name DROP NOT NULL;
```

`SET NOT NULL` 하기 전에 **기존 행에 NULL이 있으면 실패**한다. 먼저 NULL을 다 채우고 적용해야 함:

```sql
UPDATE users SET name = '미기입' WHERE name IS NULL;
ALTER TABLE users ALTER COLUMN name SET NOT NULL;
```

---

## 8. 테이블 이름 바꾸기

```sql
ALTER TABLE users RENAME TO players;
```

이제 `users`는 `players`다. 다시 되돌리려면 반대로 하면 됨.

---

## 9. `DROP TABLE` — 테이블 자체를 지운다

```sql
DROP TABLE items;
```

**데이터, 구조, 인덱스, 제약조건 전부 사라진다.** 8강의 `DELETE`/`TRUNCATE`와 다른 점: 테이블 자체가 없어진다.

옵션:

```sql
DROP TABLE IF EXISTS items;       -- 없어도 에러 안 남
DROP TABLE items CASCADE;         -- 참조하는 외래 키도 같이 정리
DROP TABLE items, guilds;         -- 여러 개 동시에
```

---

## 10. 실습 — 진화하는 테이블

7강에서 만든 `users`를 단계별로 진화시켜보자.

```sql
-- 1) email, exp 열 추가
ALTER TABLE users ADD COLUMN email TEXT;
ALTER TABLE users ADD COLUMN exp   INTEGER DEFAULT 0;

\d users

-- 2) gold를 coin으로 이름 변경
ALTER TABLE users RENAME COLUMN gold TO coin;

-- 3) name에 NOT NULL 부여 (이미 모두 값 있으므로 안전)
ALTER TABLE users ALTER COLUMN name SET NOT NULL;

-- 4) level 타입을 BIGINT로 변경
ALTER TABLE users ALTER COLUMN level TYPE BIGINT;

-- 5) email 채우고, UNIQUE 제약 부여
UPDATE users SET email = name || '@game.com';   -- '김철수@game.com' 식
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);

-- 6) 잘못 만든 임시 테이블 제거
CREATE TABLE temp_log (id INT, msg TEXT);
DROP TABLE temp_log;

-- 7) 구조 최종 확인
\d users
```

> `||`는 PostgreSQL의 **문자열 결합 연산자**. C++의 `+`나 `FString::Printf`와 비슷. `'안녕' || '하세요'` → `'안녕하세요'`.

---

## 11. 운영 환경에서 알아두면 좋은 점

학습용은 아니지만 짧게.

- **큰 테이블의 `ALTER`는 시간이 걸린다.** 일부 변경은 잠금(lock) 때문에 서비스 중단이 필요.
- 그래서 실무에선 **마이그레이션 도구**(Flyway, Liquibase, Alembic 등)로 변경 이력을 관리한다.
- 결국 `ALTER TABLE`은 **DB의 버전 관리**다. Git 커밋처럼 한 단계씩 신중하게.

지금은 "이런 게 있구나" 정도면 충분.

---

## 12. 이번 강의에서 기억할 것

1. **`ALTER TABLE ... ADD/DROP/RENAME/ALTER COLUMN`** — 구조 변경 4총사.
2. **`NOT NULL` 부여는 기존 NULL부터 채우고** 해야 안전.
3. **타입 변환은 `USING`** 과 **`::캐스팅 연산자`** 로.
4. `DROP TABLE`은 **테이블 자체를 삭제** — 데이터 포함.
5. **`IF EXISTS` / `CASCADE`** 같은 옵션을 적재적소 활용.
6. 실무에선 변경 이력을 **마이그레이션 도구**로 관리한다 (지금은 개념만).

---

## 13. 2단계 마무리

축하한다. 여기까지 끝내면 **CRUD 중 C, U, D를 다 다룬 셈**이다.

| 강의 | 핵심 |
|------|------|
| 5강 | `CREATE TABLE`로 구조 정의 |
| 6강 | 데이터 타입 — 정수/실수/문자열/날짜/불리언/JSONB |
| 7강 | `INSERT`로 데이터 채우기, `RETURNING`, `ON CONFLICT` |
| 8강 | `UPDATE` / `DELETE` 안전하게 — `WHERE` 절 사수, 트랜잭션 |
| 9강 | `ALTER TABLE` / `DROP TABLE`로 구조 변경·삭제 |

---

## 14. 다음 강의 예고 — 3단계 시작

**10강 — `SELECT` 기본**
드디어 **데이터를 꺼내 보는** 시간. `SELECT`는 SQL에서 압도적으로 많이 쓰는 명령이고, 깊이도 가장 깊다. 10~15강이 통째로 `SELECT` 시리즈가 될 정도다. 차근차근 정복해보자.

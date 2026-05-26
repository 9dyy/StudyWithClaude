# 7강 — `INSERT INTO`로 데이터 넣기

> **목표:** 테이블에 새 행을 추가하는 모든 방식을 익힌다.
> **선수 지식:** 5~6강 (`CREATE TABLE`, 데이터 타입)

---

## 1. 가장 기본 형태

```sql
INSERT INTO 테이블이름 (열1, 열2, 열3) VALUES (값1, 값2, 값3);
```

C++로 비유:

```cpp
Users.Add(FUser{ .Name = "철수", .Level = 1, .Gold = 100 });
```

이걸 SQL로 옮기면:

```sql
INSERT INTO users (name, level, gold) VALUES ('철수', 1, 100);
```

`INSERT INTO 테이블` 다음에 **(어느 열에)** 적고, `VALUES`로 **(무엇을)** 넣는다.

---

## 2. 실습 — 첫 데이터 넣기

6강에서 만든 테이블에 데이터를 넣어보자.

```sql
INSERT INTO guilds (guild_name) VALUES ('흑사자단');
INSERT INTO guilds (guild_name) VALUES ('백호회');

INSERT INTO users (name, level, gold, guild_id) VALUES ('김철수', 12, 500, 1);
INSERT INTO users (name, level, gold, guild_id) VALUES ('이영희', 45, 12000, 2);
INSERT INTO users (name, level, gold, guild_id) VALUES ('박민수', 7, 100, 1);

SELECT * FROM users;
```

결과:

```text
 id |  name  | level | gold | is_active | guild_id |         created_at
----+--------+-------+------+-----------+----------+----------------------------
  1 | 김철수 | 12    | 500  | t         | 1        | 2026-05-20 14:30:00+09
  2 | 이영희 | 45    | 12000| t         | 2        | 2026-05-20 14:30:01+09
  3 | 박민수 | 7     | 100  | t         | 1        | 2026-05-20 14:30:02+09
```

주목할 점:

- **`id`는 안 넣었는데 자동으로 1, 2, 3.** `SERIAL` 덕분.
- **`is_active`와 `created_at`도 안 넣었는데 채워졌다.** `DEFAULT` 덕분.

이게 `DEFAULT`의 힘이다.

---

## 3. 열 이름 생략하기 (비추천)

```sql
INSERT INTO guilds VALUES (10, '청룡파', NOW());
```

열 이름을 안 쓰면 **테이블 정의 순서대로** 값을 넣는다.

문제점:

- 열 순서가 바뀌면 코드가 깨진다.
- 새 열이 추가되면 인자 개수가 안 맞는다.
- 읽는 사람이 어느 값이 어느 열인지 모른다.

**그래서 실무에선 거의 항상 열 이름을 적는다.** 학습 중엔 짧은 예시에서만 생략.

---

## 4. 여러 행 한 번에 넣기

`VALUES` 뒤에 쉼표로 여러 튜플을 나열.

```sql
INSERT INTO items (user_id, item_name, quantity) VALUES
    (1, '롱소드',  1),
    (1, '체력포션', 5),
    (2, '매직스태프', 1),
    (2, '마나포션', 10),
    (3, '단검',    1);
```

한 번에 5건 입력. **여러 번 호출보다 훨씬 빠르다.** (네트워크 왕복 줄어듦)

---

## 5. `DEFAULT` 키워드 명시

기본값을 쓰고 싶지만 다른 열엔 값을 넣고 싶을 때.

```sql
INSERT INTO users (name, level, gold, is_active)
VALUES ('신입유저', DEFAULT, DEFAULT, TRUE);
```

`DEFAULT`라고 쓰면 그 열의 기본값(`1`, `0`)이 들어간다.

---

## 6. `RETURNING` — 방금 넣은 행 받아오기

PostgreSQL의 강력한 기능. **`INSERT` 직후 그 행의 정보를 바로 받을 수 있다.**

```sql
INSERT INTO users (name, level)
VALUES ('새유저', 1)
RETURNING id, created_at;
```

결과:

```text
 id |         created_at
----+----------------------------
 99 | 2026-05-20 15:00:00+09
```

C++에서 비유하면 `Add()` 직후 새 ID를 반환받는 느낌. **자동 증가 ID를 알아내야 할 때 무조건 이거 쓴다.**

```sql
RETURNING *;   -- 모든 열을 반환
```

---

## 7. 다른 테이블에서 복사해 넣기 — `INSERT ... SELECT`

```sql
-- archive 스키마의 백업용 테이블에 활성 유저만 복사
INSERT INTO archive_users (name, level)
SELECT name, level FROM users WHERE is_active = TRUE;
```

`VALUES` 대신 **`SELECT` 결과**를 넣을 수 있다. 데이터 마이그레이션이나 백업에 유용. `SELECT`는 10강부터 본격적으로 다룬다.

---

## 8. 자주 만나는 에러

### ① `null value in column "xxx" violates not-null constraint`

`NOT NULL` 제약이 걸린 열에 값을 안 줬다.

```sql
INSERT INTO users (level) VALUES (1);
-- ERROR: name이 NOT NULL인데 빠짐
```

### ② `invalid input syntax for type integer: "abc"`

타입이 안 맞는다. 정수 자리에 문자열 넣음.

### ③ `duplicate key value violates unique constraint`

`UNIQUE` 또는 `PRIMARY KEY` 중복.

```sql
INSERT INTO users (id, name) VALUES (1, '중복');
-- 이미 id=1이 있으면 에러
```

### ④ `column "xxx" of relation "users" does not exist`

오타. 없는 열 이름을 적었다.

---

## 9. 충돌 처리 — `ON CONFLICT` (살짝 보기)

같은 PK를 다시 넣으면 보통 에러. 하지만 "이미 있으면 무시" 또는 "있으면 업데이트"를 원할 때:

```sql
-- 있으면 무시
INSERT INTO users (id, name) VALUES (1, '재시도')
ON CONFLICT (id) DO NOTHING;

-- 있으면 업데이트 (이걸 UPSERT라고 부름)
INSERT INTO users (id, name) VALUES (1, '업데이트된이름')
ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name;
```

`EXCLUDED`는 "INSERT 하려던 값"을 가리키는 특수 별칭.
지금은 **이런 게 있다** 정도만 알고, 21강 이후 제약조건과 함께 다시 본다.

---

## 10. 실습 — 게임 데이터 가득 채우기

다음 강의들에서 쓸 풍부한 데이터를 만들어두자.

```sql
-- 깔끔하게 다시 시작
TRUNCATE items, users, guilds RESTART IDENTITY CASCADE;
```

> `TRUNCATE`는 테이블의 모든 행을 빠르게 삭제. `RESTART IDENTITY`는 `SERIAL`을 1부터 다시. `CASCADE`는 연결된 테이블도 함께. (강력하니 학습 외엔 조심)

```sql
INSERT INTO guilds (guild_name) VALUES
    ('흑사자단'),
    ('백호회'),
    ('청룡파');

INSERT INTO users (name, level, gold, guild_id) VALUES
    ('김철수', 12,   500,  1),
    ('이영희', 45, 12000,  2),
    ('박민수',  7,   100,  1),
    ('최지수', 30,  3000,  3),
    ('정대훈', 50, 25000,  2),
    ('홍길동',  1,    50, NULL);   -- 길드 없음

INSERT INTO items (user_id, item_name, quantity) VALUES
    (1, '롱소드',     1),
    (1, '체력포션',   5),
    (2, '매직스태프', 1),
    (2, '마나포션',  10),
    (3, '단검',       1),
    (4, '활',         1),
    (4, '화살',     100),
    (5, '전설검',     1);

-- 확인
SELECT * FROM users;
SELECT * FROM items;
```

이 데이터는 **3단계(`SELECT`) 내내 계속 쓸 거다.** 잘 보관.

---

## 11. 이번 강의에서 기억할 것

1. **`INSERT INTO 테이블 (열들) VALUES (값들);`** — 가장 기본.
2. **열 이름은 항상 명시하자.** 순서 의존하면 위험.
3. **여러 행은 한 `INSERT`에 묶어 넣는 게 빠르다.**
4. **`RETURNING`으로 자동 생성된 ID 즉시 회수 가능.**
5. **`ON CONFLICT`로 충돌 처리** (UPSERT 패턴).
6. `TRUNCATE`는 강력한 전체 삭제.

---

## 12. 다음 강의 예고

**8강 — `UPDATE`, `DELETE`로 데이터 수정·삭제**
이미 넣은 데이터를 바꾸거나 지운다. **`WHERE` 빼먹는 사고**가 가장 많이 나는 곳. 안전하게 쓰는 법을 같이 본다.

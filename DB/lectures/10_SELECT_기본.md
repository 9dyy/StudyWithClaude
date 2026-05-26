# 10강 — `SELECT` 기본

> **목표:** 테이블에서 원하는 열을 골라 조회한다.
> **선수 지식:** 1~9강 (특히 7강에서 넣은 데이터)

> 📌 **`SELECT`는 SQL의 핵심.** 실무 쿼리의 70%가 여기 있다. 10~15강이 통째로 `SELECT` 시리즈다. 천천히 가자.

---

## 1. 가장 기본 형태

```sql
SELECT 열목록 FROM 테이블이름;
```

읽는 법: **"테이블에서 / 이 열들을 / 가져와."**

C++로 비유하면 컬렉션에서 원하는 필드만 뽑아 새 리스트를 만드는 것과 비슷하다.

```cpp
// 개념적 비유
TArray<FString> Names;
for (const FUser& U : Users)
    Names.Add(U.Name);   // name 열만 뽑기
```

---

## 2. 모든 열 조회 — `*`

```sql
SELECT * FROM users;
```

`*`는 **"모든 열"** 이라는 뜻. 결과:

```text
 id |  name  | level | coin | ...
----+--------+-------+------+----
  1 | 김철수 | 20    | ...
  2 | 이영희 | 45    | ...
```

> ⚠️ 편하지만 실무에선 **`*` 남용을 피한다.** 필요 없는 열까지 다 끌어와 느려지고, 테이블 구조가 바뀌면 결과도 바뀐다. **필요한 열만 명시**하는 게 좋은 습관.

---

## 3. 원하는 열만 골라 조회

```sql
SELECT name, level FROM users;
```

결과:

```text
  name  | level
--------+-------
 김철수 | 20
 이영희 | 45
 ...
```

열 순서는 **내가 적은 순서대로** 나온다. 테이블 정의 순서와 무관.

```sql
SELECT level, name FROM users;   -- level이 먼저 나옴
```

---

## 4. 별칭 — `AS`

열이나 결과의 이름을 **임시로 바꿔서** 표시한다.

```sql
SELECT name AS 유저이름, level AS 레벨 FROM users;
```

결과의 헤더가 바뀐다:

```text
 유저이름 | 레벨
----------+------
 김철수   | 20
```

`AS`는 **생략 가능**하다.

```sql
SELECT name 유저이름, level 레벨 FROM users;   -- AS 없어도 동작
```

> 💡 별칭에 공백이나 한글, 대문자를 쓰려면 큰따옴표로 감싼다: `SELECT name AS "User Name" ...`

`AS`는 **계산 결과에 이름 붙일 때** 특히 유용하다 (다음 절).

---

## 5. 계산하면서 조회하기

`SELECT`에는 열 이름뿐 아니라 **계산식**도 쓸 수 있다.

```sql
SELECT
    name,
    level,
    level * 100 AS exp_needed,   -- 레벨 × 100
    coin / 1000  AS coin_k       -- 코인을 천 단위로
FROM users;
```

테이블에 `exp_needed`라는 열이 실제로 있는 게 아니다. **조회하는 순간 계산해서 보여주는** 가상의 열이다.

상수만으로도 가능 (테이블 없이):

```sql
SELECT 1 + 1 AS sum, 'hello' AS greeting;
```

```text
 sum | greeting
-----+----------
   2 | hello
```

> PostgreSQL은 `FROM` 없이 `SELECT`만으로 계산기처럼 쓸 수 있다. 함수 테스트할 때 자주 활용.

---

## 6. 문자열 결합 — `||`

9강에서 잠깐 본 그것.

```sql
SELECT name || ' (Lv.' || level || ')' AS display
FROM users;
```

결과:

```text
       display
----------------------
 김철수 (Lv.20)
 이영희 (Lv.45)
```

C++의 `FString` 더하기나 `Printf`와 비슷한 역할. **숫자도 자동으로 문자열로 변환**되어 붙는다.

---

## 7. 함수 써보기

PostgreSQL엔 내장 함수가 아주 많다. 맛보기.

```sql
SELECT
    UPPER(name)        AS upper_name,    -- 대문자 (영문)
    LENGTH(name)       AS name_length,   -- 글자 수
    NOW()              AS now,           -- 현재 시각
    ROUND(coin / 3.0, 2) AS divided      -- 반올림 (소수 2자리)
FROM users;
```

자주 쓰는 함수 미리보기:

| 함수 | 설명 |
|------|------|
| `UPPER(s)` / `LOWER(s)` | 대/소문자 변환 |
| `LENGTH(s)` | 문자열 길이 |
| `TRIM(s)` | 앞뒤 공백 제거 |
| `SUBSTRING(s FROM 1 FOR 3)` | 부분 문자열 |
| `ROUND(n, d)` | 반올림 |
| `ABS(n)` | 절댓값 |
| `COALESCE(a, b)` | a가 NULL이면 b (32강) |
| `NOW()` | 현재 timestamp |

집계 함수(`COUNT`, `SUM` 등)는 14강에서 따로 다룬다.

---

## 8. 중복 없이 — `DISTINCT` (맛보기)

```sql
SELECT DISTINCT guild_id FROM users;
```

같은 `guild_id` 값은 한 번만 보여준다. 자세한 건 15강에서.

---

## 9. 주석 쓰는 법

SQL에도 주석이 있다.

```sql
-- 한 줄 주석 (대시 두 개)

/*
   여러 줄 주석
   이렇게
*/

SELECT name  -- 줄 끝에도 가능
FROM users;
```

---

## 10. 실습

```sql
-- 1) 전체 조회
SELECT * FROM users;

-- 2) 이름과 레벨만
SELECT name, level FROM users;

-- 3) 별칭 + 계산
SELECT
    name        AS 이름,
    level       AS 레벨,
    level * 100 AS 필요경험치
FROM users;

-- 4) 문자열 조합으로 표시용 문구 만들기
SELECT name || '님 환영합니다!' AS welcome FROM users;

-- 5) 함수 활용
SELECT name, LENGTH(name) AS 글자수 FROM users;

-- 6) FROM 없이 계산기
SELECT 100 * 1.1 AS with_tax;
```

---

## 11. 이번 강의에서 기억할 것

1. **`SELECT 열목록 FROM 테이블;`** — 조회의 기본형.
2. `*`는 모든 열이지만 **실무에선 필요한 열만 명시**.
3. **`AS`로 별칭** — 계산 결과에 이름 붙일 때 특히 유용.
4. `SELECT`에 **계산식·함수·문자열 결합(`||`)** 다 쓸 수 있다.
5. `FROM` 없이 `SELECT`만으로 계산기처럼 활용 가능.

---

## 12. 다음 강의 예고

**11강 — `WHERE`로 조건 필터링**
지금은 전체 행을 다 가져왔다. 이제 **"레벨 30 이상인 유저만"** 처럼 조건으로 걸러내는 법을 배운다. `SELECT`의 진짜 힘은 여기서 나온다.

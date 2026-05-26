# 11강 — `WHERE`로 조건 필터링

> **목표:** 원하는 조건에 맞는 행만 골라낸다.
> **선수 지식:** 10강 (`SELECT` 기본)

---

## 1. `WHERE`의 위치

```sql
SELECT 열목록
FROM 테이블
WHERE 조건;
```

C++의 `if`와 같다. **조건이 참(TRUE)인 행만** 결과에 포함된다.

```cpp
for (const FUser& U : Users)
    if (U.Level >= 30)        // ← WHERE level >= 30
        Result.Add(U);
```

```sql
SELECT * FROM users WHERE level >= 30;
```

---

## 2. 비교 연산자

| 연산자 | 의미 | 예시 |
|--------|------|------|
| `=` | 같다 (C++은 `==`, SQL은 `=` 하나!) | `level = 10` |
| `<>` 또는 `!=` | 같지 않다 | `level <> 10` |
| `>` `<` | 크다 / 작다 | `coin > 1000` |
| `>=` `<=` | 크거나/작거나 같다 | `level >= 30` |

> ⚠️ **SQL의 "같다"는 `=` 하나다.** C++ 습관으로 `==` 쓰면 에러. 자주 하는 실수.

```sql
SELECT * FROM users WHERE level = 45;     -- 레벨 정확히 45
SELECT * FROM users WHERE coin > 1000;    -- 코인 1000 초과
SELECT * FROM users WHERE name <> '홍길동'; -- 홍길동 제외
```

---

## 3. 문자열 조건

문자열은 작은따옴표로 감싼다.

```sql
SELECT * FROM users WHERE name = '김철수';
```

대소문자를 구분한다 (`'Kim' <> 'kim'`). 대소문자 무시하고 비교하려면 `LOWER()`:

```sql
SELECT * FROM users WHERE LOWER(name) = LOWER('KimCheolsu');
```

---

## 4. 논리 연산자 — `AND`, `OR`, `NOT`

C++의 `&&`, `||`, `!`에 해당.

```sql
-- 레벨 30 이상 AND 코인 1000 이상
SELECT * FROM users
WHERE level >= 30 AND coin >= 1000;

-- 레벨 10 미만 OR 코인 100 미만
SELECT * FROM users
WHERE level < 10 OR coin < 100;

-- 흑사자단(1)이 아닌 유저
SELECT * FROM users
WHERE NOT guild_id = 1;
```

### 괄호로 우선순위 명확히

`AND`가 `OR`보다 먼저 묶인다 (C++과 동일). 헷갈리면 괄호.

```sql
-- (길드1 또는 길드2) 이면서 레벨 20 이상
SELECT * FROM users
WHERE (guild_id = 1 OR guild_id = 2) AND level >= 20;
```

괄호 없으면 `guild_id = 1 OR (guild_id = 2 AND level >= 20)`로 해석되어 결과가 달라진다.

---

## 5. 범위 — `BETWEEN`

```sql
-- 레벨 10 ~ 40 사이 (양 끝 포함)
SELECT * FROM users WHERE level BETWEEN 10 AND 40;
```

`level >= 10 AND level <= 40`과 같다. **양 끝값 포함**이라는 점 주의.

부정:

```sql
SELECT * FROM users WHERE level NOT BETWEEN 10 AND 40;
```

---

## 6. 목록 — `IN`

여러 값 중 하나라도 일치하면.

```sql
-- 길드가 1, 2, 3 중 하나
SELECT * FROM users WHERE guild_id IN (1, 2, 3);
```

`guild_id = 1 OR guild_id = 2 OR guild_id = 3`과 같다. 훨씬 간결.

```sql
SELECT * FROM users WHERE name IN ('김철수', '이영희');
SELECT * FROM users WHERE guild_id NOT IN (1, 2);   -- 부정
```

> 💡 `IN` 괄호 안에 **서브쿼리**도 넣을 수 있다 (24~25강).

---

## 7. 패턴 매칭 — `LIKE`

문자열의 일부로 검색. 두 가지 와일드카드:

| 기호 | 의미 |
|------|------|
| `%` | 0글자 이상 아무 문자 |
| `_` | 정확히 1글자 |

```sql
-- '김'으로 시작
SELECT * FROM users WHERE name LIKE '김%';

-- '수'로 끝남
SELECT * FROM users WHERE name LIKE '%수';

-- '영'을 포함
SELECT * FROM users WHERE name LIKE '%영%';

-- 정확히 3글자이면서 '김'으로 시작
SELECT * FROM users WHERE name LIKE '김__';
```

### 대소문자 무시 — `ILIKE`

PostgreSQL 전용. (`I` = Insensitive)

```sql
SELECT * FROM users WHERE name ILIKE 'kim%';   -- Kim, KIM, kim 다 매칭
```

---

## 8. NULL 조건 — `IS NULL`

6강에서 강조한 그것. **NULL은 `=`로 비교 못 한다.**

```sql
SELECT * FROM users WHERE guild_id IS NULL;       -- 길드 없는 유저
SELECT * FROM users WHERE guild_id IS NOT NULL;   -- 길드 있는 유저
```

```sql
SELECT * FROM users WHERE guild_id = NULL;   -- ❌ 항상 0건! (NULL은 = 로 비교 불가)
```

---

## 9. `WHERE`는 `UPDATE`/`DELETE`에도 똑같이

8강에서 봤듯, `WHERE`는 `SELECT`만의 것이 아니다.

```sql
UPDATE users SET coin = coin + 100 WHERE level >= 40;
DELETE FROM users WHERE is_active = FALSE;
```

여기서 배운 모든 조건 문법이 그대로 적용된다. **`SELECT`로 먼저 확인 → 같은 `WHERE`로 `UPDATE`/`DELETE`** 습관!

---

## 10. 실습

```sql
-- 1) 레벨 30 이상
SELECT name, level FROM users WHERE level >= 30;

-- 2) 코인 1000 이상이고 레벨 40 이상
SELECT * FROM users WHERE coin >= 1000 AND level >= 40;

-- 3) 레벨 10~50 범위
SELECT name, level FROM users WHERE level BETWEEN 10 AND 50;

-- 4) 길드 1번 또는 2번 소속
SELECT name, guild_id FROM users WHERE guild_id IN (1, 2);

-- 5) 이름이 '김'으로 시작
SELECT name FROM users WHERE name LIKE '김%';

-- 6) 길드 없는 유저
SELECT name FROM users WHERE guild_id IS NULL;

-- 7) 복합: (길드1 또는 길드3) 이고 코인 500 초과
SELECT name, guild_id, coin
FROM users
WHERE (guild_id = 1 OR guild_id = 3) AND coin > 500;
```

---

## 11. 이번 강의에서 기억할 것

1. **`WHERE 조건`** — 조건이 참인 행만 통과. C++의 `if`.
2. **같다는 `=` 하나** (`==` 아님), 다르다는 `<>` 또는 `!=`.
3. **`AND`/`OR`/`NOT`** + 괄호로 복합 조건. `AND`가 먼저 묶인다.
4. **`BETWEEN`**(범위, 양끝 포함), **`IN`**(목록), **`LIKE`**(패턴, `%`/`_`).
5. **NULL 조건은 `IS NULL` / `IS NOT NULL`** 만.
6. `WHERE`는 `UPDATE`/`DELETE`에도 동일하게 쓰인다.

---

## 12. 다음 강의 예고

**12강 — `ORDER BY`, `LIMIT`, `OFFSET`**
조회 결과를 **정렬**하고, **개수를 제한**하고, **페이지 단위로 끊어** 보는 법. "랭킹 TOP 10" 같은 걸 만들 수 있게 된다.

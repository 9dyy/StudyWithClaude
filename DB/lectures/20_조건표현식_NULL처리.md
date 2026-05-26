# 20강 — 조건 표현식과 NULL 처리 (`CASE`, `COALESCE`, `NULLIF`, `IS DISTINCT FROM`)

> **목표:** SQL 안에서 if-else를 쓰고, NULL을 안전하게 다룬다.
> **선수 지식:** 10~19강 (특히 6강의 NULL 함정, 11강의 `WHERE`)

---

## 1. 왜 SQL 안에서 조건 처리가 필요한가?

데이터를 그대로 보여주기보단 **가공해서** 보여줘야 할 때가 많다.

- 레벨에 따라 "초보 / 중수 / 고수"로 분류
- `coin`이 NULL이면 0으로 대체해서 합계
- "값이 같으면 NULL로, 다르면 그대로" 같은 트릭

C++의 `if`/`switch`/`?:` 같은 도구가 SQL에도 있다. 오늘 다룰 네 가지.

| 도구 | 한 줄 |
|------|------|
| `CASE WHEN` | SQL의 if-else / switch |
| `COALESCE` | "NULL이면 대체값" |
| `NULLIF` | "두 값이 같으면 NULL" |
| `IS DISTINCT FROM` | NULL-안전한 "다름" 비교 |

추가로 `GREATEST` / `LEAST`도 보너스로 짚는다.

---

## 2. `CASE WHEN` — SQL의 if-else

두 가지 형태가 있다.

### 검색형(searched) CASE — 더 일반적

```sql
CASE
  WHEN 조건1 THEN 값1
  WHEN 조건2 THEN 값2
  ...
  ELSE 기본값
END
```

```sql
SELECT name, level,
  CASE
    WHEN level >= 50 THEN '고수'
    WHEN level >= 30 THEN '중수'
    ELSE '초보'
  END AS tier
FROM users;
```

결과:

```text
   name   | level | tier
----------+-------+------
 김철수   |  30   | 중수
 이영희   |  45   | 중수
 박무소속 |  12   | 초보
 최백호   |  50   | 고수
```

C++의 if-else if-else 사슬과 의미가 같다:

```cpp
FString Tier;
if (Level >= 50)      Tier = TEXT("고수");
else if (Level >= 30) Tier = TEXT("중수");
else                  Tier = TEXT("초보");
```

> 💡 **위에서 아래로 평가하다가 처음 TRUE가 되는 WHEN을 선택**한다. C++의 if-else if와 동일. 순서가 의미를 좌우.

### 단순형(simple) CASE — switch 같은 것

```sql
CASE 컬럼
  WHEN 값1 THEN 결과1
  WHEN 값2 THEN 결과2
  ELSE 기본값
END
```

```sql
SELECT name, guild_id,
  CASE guild_id
    WHEN 1 THEN '흑사자단'
    WHEN 2 THEN '백호단'
    ELSE   '무소속'
  END AS guild_name_inline
FROM users;
```

C++의 `switch`와 같은 감각. **하지만 NULL 비교가 안 된다** (`= NULL`은 항상 NULL이라 매칭 X). NULL 처리하려면 **검색형으로 바꿔야**:

```sql
CASE
  WHEN guild_id IS NULL THEN '무소속'
  WHEN guild_id = 1     THEN '흑사자단'
  WHEN guild_id = 2     THEN '백호단'
END
```

> 💡 **헷갈리면 검색형 하나만 알고 있어도 충분**. 단순형은 검색형의 특수 케이스.

### `ELSE` 생략하면?

`ELSE`를 안 쓰면 매칭 안 된 경우 **NULL**이 된다. 명시적으로 `ELSE`를 쓰는 게 안전.

```sql
SELECT name,
  CASE WHEN level >= 50 THEN '고수' END AS top_only
FROM users;
-- 레벨 50 미만 유저는 top_only가 NULL
```

---

## 3. `CASE`는 어디서든 쓸 수 있다

`CASE`는 **표현식(expression)** 이라서 값이 들어갈 만한 곳이면 어디든 가능.

```sql
-- SELECT (방금 본 예)
SELECT name, CASE ... END AS tier FROM users;

-- WHERE — 조건부 필터
SELECT * FROM users
WHERE
  CASE
    WHEN level >= 40 THEN coin >= 1000
    ELSE              coin >= 100
  END;
-- 의미: 고렙은 코인 1000+, 저렙은 100+ 만 통과

-- ORDER BY — 조건부 정렬
SELECT * FROM users
ORDER BY
  CASE guild_id WHEN 1 THEN 0 ELSE 1 END,   -- 흑사자단 우선
  level DESC;

-- GROUP BY — 조건부 그룹핑
SELECT
  CASE
    WHEN level >= 50 THEN '고수'
    WHEN level >= 30 THEN '중수'
    ELSE '초보'
  END AS tier,
  COUNT(*) AS cnt
FROM users
GROUP BY 1;   -- 1은 첫 번째 SELECT 컬럼

-- UPDATE — 조건부 갱신
UPDATE users
SET status = CASE
  WHEN last_login_at < NOW() - INTERVAL '90 days' THEN 'inactive'
  ELSE 'active'
END;

-- 집계 함수 안에 — 조건부 카운트/합계 (★★★ 매우 자주 씀)
SELECT
  COUNT(*) AS total,
  COUNT(CASE WHEN level >= 50 THEN 1 END) AS high_level_count,
  SUM(CASE WHEN guild_id = 1 THEN coin ELSE 0 END) AS guild1_total_coin
FROM users;
```

> 💡 **마지막 패턴이 핵심이다.** 13~14강에서 본 `GROUP BY` + 집계를 `CASE`로 확장하면 **"조건별 카운트/합계를 한 줄에 옆으로 펼쳐서"** 출력하는 리포트가 가능해진다. 흔히 말하는 **피벗(pivot)** 의 기본 형태.

---

## 4. `COALESCE` — NULL 대체

`COALESCE(a, b, c, ...)`는 **인자를 순서대로 보다가 처음으로 NOT NULL인 값을 반환**한다. 전부 NULL이면 NULL.

```sql
SELECT name, COALESCE(coin, 0) AS coin_safe FROM users;
-- coin이 NULL이면 0으로 표시
```

```sql
SELECT
  user_id,
  COALESCE(nickname, name, email, '익명') AS display_name
FROM users;
-- nickname 있으면 그거, 없으면 name, 그것도 없으면 email, 다 없으면 '익명'
```

C++로 비유:

```cpp
FString Display = !Nickname.IsEmpty() ? Nickname
                : !Name.IsEmpty()     ? Name
                : !Email.IsEmpty()    ? Email
                :                       TEXT("익명");
```

### 집계할 때 특히 유용

NULL은 `SUM`/`AVG`에서 무시되지만, **`+` 연산에선 결과를 NULL로 오염**시킨다.

```sql
SELECT coin + bonus FROM users;
-- bonus가 NULL인 행은 결과도 NULL

SELECT COALESCE(coin, 0) + COALESCE(bonus, 0) AS total FROM users;
-- 안전
```

> 💡 PostgreSQL엔 `NVL`(Oracle), `IFNULL`(MySQL) 같은 함수가 없다. **`COALESCE`가 표준**이고 가장 강력하다 (2개 이상 인자 지원).

---

## 5. `NULLIF` — 두 값이 같으면 NULL

`NULLIF(a, b)`는 **`a = b`이면 NULL, 다르면 `a`** 를 반환한다. 사용 빈도는 `COALESCE`보다 낮지만, 한 가지 강력한 용도가 있다.

### 0으로 나누기 방지

```sql
SELECT total, success_count,
       success_count * 100.0 / NULLIF(total, 0) AS success_rate
FROM stats;
-- total이 0이면 NULLIF가 NULL을 반환 → NULL/0 에러 대신 결과만 NULL
```

만약 `NULLIF` 없이 `success_count * 100.0 / total`로 쓰면 `total=0`인 행에서 **division by zero 에러**가 난다.

### `COALESCE`와 짝지어 쓰기

```sql
SELECT
  COALESCE(success_count * 100.0 / NULLIF(total, 0), 0) AS rate
FROM stats;
-- 0으로 나누면 NULL → COALESCE로 0 대체
```

> 💡 `NULLIF + COALESCE` 조합은 **0으로 나누기 안전 가드의 표준 패턴**.

---

## 6. `IS DISTINCT FROM` — NULL-안전 비교

6강에서 강조했던 함정. **`NULL = NULL`은 TRUE가 아니라 NULL**이다.

| 비교 | `=` 결과 | `IS [NOT] DISTINCT FROM` 결과 |
|------|---------|-------------------------------|
| `1 vs 2` | TRUE (다름) | TRUE (다름) |
| `1 vs 1` | FALSE (같음) | FALSE (같음) |
| `1 vs NULL` | **NULL** (모름) | TRUE (다름) |
| `NULL vs NULL` | **NULL** (모름) | FALSE (같음) |

즉 `IS DISTINCT FROM`은 **NULL을 일반 값처럼** 다르다/같다로 판정.

### 언제 쓰나?

```sql
-- 변경 감지: 새 값이 기존 값과 다르면 UPDATE (NULL 포함해서)
UPDATE users
SET nickname = '신규닉'
WHERE id = 1
  AND nickname IS DISTINCT FROM '신규닉';
-- 기존 nickname이 NULL이어도 '신규닉'과 "다르다"고 인식해서 UPDATE 됨
-- 일반 <> 였다면 NULL이라 WHERE가 NULL → 행이 안 잡힘 → UPDATE 안 됨

-- 백필: 이미 올바른 값이면 스킵
UPDATE users SET account_type = '계산된값'
WHERE account_type IS DISTINCT FROM '계산된값';
```

11강에서 본 `NOT`/`<>`로는 NULL 행을 못 잡지만, `IS DISTINCT FROM`은 잡는다.

---

## 7. `GREATEST` / `LEAST` — 인자들 중 최댓/최솟값

집계 함수 `MAX`/`MIN`은 **열의 여러 행** 중 최댓값이고, `GREATEST`/`LEAST`는 **한 행의 여러 열(인자)** 중 최댓값.

```sql
SELECT
  user_id,
  GREATEST(login_at, last_action_at, last_chat_at) AS last_activity
FROM user_activity;
-- 한 행 안의 세 시각 중 가장 최근 시각

SELECT LEAST(price, sale_price, member_price) AS final_price
FROM products;
```

> ⚠️ **NULL 인자가 하나라도 있으면 `GREATEST`/`LEAST` 결과도 NULL**(표준 동작). NULL을 무시하고 싶다면 `COALESCE`로 감싸자.
> ```sql
> GREATEST(COALESCE(a, '1900-01-01'), COALESCE(b, '1900-01-01'), ...)
> ```

---

## 8. 실습

```sql
-- 1) 레벨에 따른 등급 표시
SELECT name, level,
  CASE
    WHEN level >= 50 THEN '고수'
    WHEN level >= 30 THEN '중수'
    WHEN level >= 10 THEN '초보'
    ELSE                 '입문'
  END AS tier
FROM users
ORDER BY level DESC;

-- 2) 길드명을 항상 채워서 (NULL이면 '무소속')
SELECT u.name,
       COALESCE(g.guild_name, '무소속') AS guild_display
FROM   users u
LEFT JOIN guilds g  ON g.id = u.guild_id;

-- 3) 조건부 카운트 — 한 줄 리포트
SELECT
  COUNT(*)                                                AS total_users,
  COUNT(CASE WHEN guild_id IS NULL THEN 1 END)            AS no_guild,
  COUNT(CASE WHEN level >= 50 THEN 1 END)                 AS high_level,
  SUM  (CASE WHEN guild_id = 1 THEN 1 ELSE 0 END)         AS guild1_count
FROM users;

-- 4) 0으로 나누기 안전 — 길드별 평균 레벨 (가입자 0인 길드도 표시)
SELECT g.guild_name,
       COUNT(u.id)                                AS member_cnt,
       COALESCE(SUM(u.level) * 1.0
                / NULLIF(COUNT(u.id), 0), 0)      AS avg_level
FROM guilds g
LEFT JOIN users u ON u.guild_id = g.id
GROUP BY g.id, g.guild_name;

-- 5) NULL-안전 변경 감지 (실제로는 같은 값이면 UPDATE 스킵)
UPDATE users
SET    status = 'active'
WHERE  id = 1
  AND  status IS DISTINCT FROM 'active';

-- 6) ORDER BY에 CASE — 흑사자단 먼저, 같은 길드 안에선 레벨 내림차순
SELECT name, guild_id, level
FROM users
ORDER BY
  CASE guild_id WHEN 1 THEN 0 ELSE 1 END,
  level DESC;

-- 7) GREATEST — 마지막 활동 시각
-- (예시 테이블이 없으므로 의사 쿼리)
-- SELECT user_id, GREATEST(login_at, action_at, chat_at) AS last_seen FROM ...
```

---

## 9. 자주 하는 실수

1. **단순형 `CASE`로 NULL 비교**
   ```sql
   CASE guild_id WHEN NULL THEN '무소속' END   -- ❌ 절대 매칭 안 됨
   ```
   → 검색형으로 `WHEN guild_id IS NULL THEN ...`

2. **`COALESCE`를 안 쓰고 NULL을 그대로 산술 연산**
   ```sql
   SELECT coin + bonus FROM users;             -- bonus NULL이면 결과 NULL
   SELECT COALESCE(coin,0) + COALESCE(bonus,0) -- 안전
   ```

3. **0으로 나누기**
   ```sql
   amount / total                              -- total=0 시 에러
   amount / NULLIF(total, 0)                   -- 안전
   ```

4. **`<>` / `NOT =` 로 NULL 행 잡으려고 하기**
   ```sql
   WHERE x <> 'A'              -- x=NULL 행 누락
   WHERE x IS DISTINCT FROM 'A' -- x=NULL 행도 포함
   ```

5. **`ELSE` 누락**
   ```sql
   CASE WHEN level >= 50 THEN '고수' END
   ```
   → 50 미만은 전부 NULL이 됨. 의도였다면 OK, 아니면 `ELSE` 명시.

---

## 10. 이번 강의에서 기억할 것

1. **`CASE WHEN ... THEN ... ELSE ... END`** = SQL의 if-else. 어디서든 쓸 수 있는 표현식.
2. **검색형이 단순형보다 일반적이고 NULL도 다룰 수 있다.** 단순형은 switch처럼 깔끔할 때만.
3. **집계 함수 안의 `CASE`** = 조건부 COUNT/SUM. 한 줄 리포트의 핵심 패턴.
4. **`COALESCE(a, b, ...)`** = 처음 NOT NULL 값 반환. NULL 대체의 표준.
5. **`NULLIF(a, b)`** = 두 값 같으면 NULL. **0으로 나누기 방지의 표준** (`NULLIF` + `COALESCE` 조합).
6. **`IS DISTINCT FROM`** = NULL을 일반 값처럼 비교하는 "다름". 변경 감지/백필에 유용.
7. **`GREATEST`/`LEAST`** = 한 행 안의 인자들 중 최댓/최솟값. NULL 주의.
8. NULL 함정 (6강·11강) + 오늘의 도구들 = **실무 SQL의 안전망**.

---

## 11. 4단계 마무리

여기까지 끝내면 **SELECT 시리즈와 관계 설계의 핵심**을 다 통과한 셈이다.

| 강의 | 핵심 |
|------|------|
| 10강 | `SELECT` 기본 |
| 11강 | `WHERE` 조건 필터링 |
| 12강 | `ORDER BY` / `LIMIT` / `OFFSET` |
| 13강 | `GROUP BY` / `HAVING` |
| 14강 | 집계 함수 |
| 15강 | `DISTINCT` |
| 16강 | `PRIMARY KEY` / `FOREIGN KEY` |
| 17강 | `JOIN` 기본 (INNER, LEFT) |
| 18강 | `JOIN` 심화 (RIGHT/FULL/CROSS/SELF, 다중 JOIN) |
| 19강 | 집합 연산 (UNION / INTERSECT / EXCEPT) |
| 20강 | 조건 표현식과 NULL 처리 |

이제 **실무의 거의 모든 SELECT 쿼리를 읽고 쓸 수 있다.** 다음 단계는 제약조건(21~23강) / 서브쿼리(24~25강) / 트랜잭션(26강~) 같은 더 깊은 주제들로 이어진다.

---

## 12. 다음 강의 예고

**21강 — 제약조건 1 (`NOT NULL`, `DEFAULT`, `UNIQUE`)**
5강에서 살짝 본 제약조건을 본격적으로 파헤친다. 데이터 품질을 DB 레벨에서 강제하는 도구들. **"앱 코드가 깜빡해도 DB가 막아준다"** 라는 안전망의 첫 단추.

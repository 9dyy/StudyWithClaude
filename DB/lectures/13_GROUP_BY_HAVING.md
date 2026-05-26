# 13강 — `GROUP BY`, `HAVING`

> **목표:** 데이터를 그룹으로 묶어 집계한다.
> **선수 지식:** 10~12강

> 📌 이 강의는 14강(집계 함수)과 한 몸이다. 여기서 `GROUP BY`의 **개념**을, 14강에서 함수들을 **자세히** 본다.

---

## 1. "집계"가 뭔가?

지금까지는 행 하나하나를 봤다. 그런데 이런 질문은?

- "길드별로 **몇 명씩** 있지?"
- "길드별 **평균 레벨**은?"
- "전체 유저의 **총 코인**은?"

이건 개별 행이 아니라 **여러 행을 묶어서 하나의 숫자로** 만드는 작업이다. 이걸 **집계(aggregation)** 라고 한다.

C++로 비유:

```cpp
// 길드별 인원수 세기
TMap<int32, int32> CountByGuild;
for (const FUser& U : Users)
    CountByGuild.FindOrAdd(U.GuildId)++;
```

이 `TMap`으로 묶어 세는 작업을 SQL에선 `GROUP BY`가 한다.

---

## 2. 집계 함수 미리보기

| 함수 | 의미 |
|------|------|
| `COUNT(*)` | 행 개수 |
| `SUM(열)` | 합계 |
| `AVG(열)` | 평균 |
| `MAX(열)` | 최댓값 |
| `MIN(열)` | 최솟값 |

(자세한 건 14강. 여기선 `GROUP BY`와 함께 쓰는 모습만 본다.)

---

## 3. 전체를 하나로 집계 — `GROUP BY` 없이

먼저 그룹 없이 전체 집계부터.

```sql
SELECT COUNT(*) FROM users;          -- 전체 유저 수
SELECT AVG(level) FROM users;        -- 전체 평균 레벨
SELECT SUM(coin) AS 총코인 FROM users; -- 전체 코인 합
SELECT MAX(level), MIN(level) FROM users;  -- 최고/최저 레벨
```

결과는 **딱 한 줄**. 모든 행이 하나로 뭉쳐졌다.

```text
 count
-------
     6
```

---

## 4. 그룹별 집계 — `GROUP BY`

이제 "길드별로" 나눠서 집계해보자.

```sql
SELECT guild_id, COUNT(*) AS 인원수
FROM users
GROUP BY guild_id;
```

결과:

```text
 guild_id | 인원수
----------+--------
        1 |      2
        2 |      2
        3 |      1
   (null) |      1
```

**`GROUP BY guild_id`** 가 같은 `guild_id`끼리 행을 묶고, **`COUNT(*)`** 가 각 묶음의 개수를 센다.

### 그림으로 이해

```
원본:                    그룹화:               집계:
guild_id=1 (김철수)  ┐
guild_id=1 (박민수)  ┘→ [길드1 묶음] → COUNT=2
guild_id=2 (이영희)  ┐
guild_id=2 (정대훈)  ┘→ [길드2 묶음] → COUNT=2
guild_id=3 (최지수)   → [길드3 묶음] → COUNT=1
guild_id=NULL(홍길동) → [NULL 묶음] → COUNT=1
```

---

## 5. 여러 집계를 한 번에

```sql
SELECT
    guild_id,
    COUNT(*)    AS 인원수,
    AVG(level)  AS 평균레벨,
    SUM(coin)   AS 총코인,
    MAX(level)  AS 최고레벨
FROM users
GROUP BY guild_id;
```

길드별로 인원수, 평균 레벨, 총 코인, 최고 레벨을 한 방에.

---

## 6. ⭐ 핵심 규칙 — 가장 헷갈리는 부분

**`GROUP BY`를 쓰면, `SELECT`에 올 수 있는 건 두 가지뿐이다:**

1. **`GROUP BY`에 적은 열** (그룹 기준)
2. **집계 함수** (`COUNT`, `SUM` 등)

이걸 어기면 에러가 난다.

```sql
-- ❌ 에러! name은 그룹 기준도 아니고 집계 함수도 아님
SELECT guild_id, name, COUNT(*)
FROM users
GROUP BY guild_id;
-- ERROR: column "users.name" must appear in the GROUP BY clause
--        or be used in an aggregate function
```

**왜?** 길드1 묶음엔 김철수, 박민수 둘이 있다. 그럼 `name`은 누구 걸 보여줄지 정할 수 없다. 그래서 DB가 거부한다.

해결: 묶거나(`GROUP BY name`도 추가) 집계하거나(`MAX(name)` 등). 보통은 **그룹 기준에 추가**.

```sql
-- 길드 + 레벨 조합으로 묶기
SELECT guild_id, level, COUNT(*)
FROM users
GROUP BY guild_id, level;
```

---

## 7. 집계 결과를 거르기 — `HAVING`

"인원수가 2명 이상인 길드만 보고 싶다." 어떻게?

`WHERE`로 하면 될 것 같지만 **안 된다.**

```sql
-- ❌ 에러! WHERE에선 집계 함수를 못 쓴다
SELECT guild_id, COUNT(*)
FROM users
WHERE COUNT(*) >= 2
GROUP BY guild_id;
```

**집계 결과를 거를 땐 `HAVING`을 쓴다.**

```sql
SELECT guild_id, COUNT(*) AS 인원수
FROM users
GROUP BY guild_id
HAVING COUNT(*) >= 2;
```

### `WHERE` vs `HAVING` — 결정적 차이

| 구분 | `WHERE` | `HAVING` |
|------|---------|----------|
| 거르는 대상 | **개별 행** (그룹화 전) | **그룹** (그룹화 후) |
| 집계 함수 | 못 씀 | 쓸 수 있음 |
| 실행 시점 | `GROUP BY` 이전 | `GROUP BY` 이후 |

비유: `WHERE`는 **재료를 고르는 단계**, `HAVING`은 **완성된 요리를 고르는 단계**.

---

## 8. `WHERE` + `GROUP BY` + `HAVING` 다 함께

세 개를 같이 쓰면 이런 흐름이다.

```sql
SELECT guild_id, AVG(level) AS 평균레벨
FROM users
WHERE level >= 5             -- 1) 레벨 5 이상 행만 추리고
GROUP BY guild_id            -- 2) 길드별로 묶어서
HAVING AVG(level) >= 20      -- 3) 평균레벨 20 이상인 길드만
ORDER BY 평균레벨 DESC;       -- 4) 평균 높은 순 정렬
```

### 실행 순서 (완성판)

```
FROM       테이블 읽기
WHERE      개별 행 필터
GROUP BY   그룹으로 묶기
HAVING     그룹 필터
SELECT     열·집계 선택
ORDER BY   정렬
LIMIT      자르기
```

이 순서를 머리에 넣으면 "왜 WHERE에선 집계를 못 쓰지?"가 자연히 이해된다. **`WHERE`는 그룹화 전, 집계는 그룹화 후**에 계산되니까.

---

## 9. 실습

```sql
-- 1) 전체 유저 수
SELECT COUNT(*) AS 전체유저 FROM users;

-- 2) 길드별 인원수
SELECT guild_id, COUNT(*) AS 인원수
FROM users
GROUP BY guild_id;

-- 3) 길드별 평균 레벨 + 총 코인
SELECT guild_id, ROUND(AVG(level), 1) AS 평균레벨, SUM(coin) AS 총코인
FROM users
GROUP BY guild_id;

-- 4) 인원이 2명 이상인 길드만
SELECT guild_id, COUNT(*) AS 인원수
FROM users
GROUP BY guild_id
HAVING COUNT(*) >= 2;

-- 5) 아이템 테이블: 유저별 보유 아이템 종류 수
SELECT user_id, COUNT(*) AS 아이템종류
FROM items
GROUP BY user_id
ORDER BY 아이템종류 DESC;

-- 6) 종합: 레벨 5 이상 유저 대상, 길드별 평균레벨 15 이상인 곳을 평균 높은 순
SELECT guild_id, ROUND(AVG(level), 1) AS 평균레벨
FROM users
WHERE level >= 5
GROUP BY guild_id
HAVING AVG(level) >= 15
ORDER BY 평균레벨 DESC;
```

---

## 10. 이번 강의에서 기억할 것

1. **`GROUP BY 열`** — 같은 값끼리 행을 묶는다.
2. `GROUP BY` 후 `SELECT`엔 **그룹 기준 열 또는 집계 함수만** 올 수 있다.
3. **`WHERE`는 그룹화 전(개별 행), `HAVING`은 그룹화 후(그룹)** 를 거른다.
4. 집계 함수 조건은 `WHERE`가 아니라 **`HAVING`**.
5. 실행 순서: **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.**

---

## 11. 다음 강의 예고

**14강 — 집계 함수 자세히**
`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`을 깊이 파헤친다. `COUNT(*)`와 `COUNT(열)`의 차이, NULL이 집계에 미치는 영향, `COUNT(DISTINCT ...)` 같은 실전 디테일까지.

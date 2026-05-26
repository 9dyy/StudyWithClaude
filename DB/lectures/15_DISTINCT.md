# 15강 — `DISTINCT`로 중복 제거

> **목표:** 조회 결과에서 중복을 없앤다.
> **선수 지식:** 10~14강

---

## 1. 중복이 왜 생기나?

테이블엔 같은 값이 여러 행에 반복될 수 있다. 예를 들어 `guild_id`만 뽑으면:

```sql
SELECT guild_id FROM users;
```

```text
 guild_id
----------
        1
        2
        1
        3
        2
   (null)
```

길드 1, 2가 중복으로 나온다. "어떤 길드들이 있나?"를 알고 싶을 땐 중복이 거슬린다.

---

## 2. `DISTINCT` — 중복 제거

`SELECT` 바로 뒤에 `DISTINCT`를 붙인다.

```sql
SELECT DISTINCT guild_id FROM users;
```

```text
 guild_id
----------
        1
        2
        3
   (null)
```

중복이 사라졌다. (NULL은 하나로 취급되어 한 번 나온다.)

---

## 3. 여러 열에 대한 `DISTINCT`

`DISTINCT`는 **나열한 열들의 조합**이 같은 행을 하나로 친다.

```sql
SELECT DISTINCT guild_id, level FROM users;
```

이건 `guild_id` 하나만이 아니라 **(guild_id, level) 쌍**이 똑같은 경우에만 중복으로 본다.

> ⚠️ 흔한 오해: `SELECT DISTINCT a, b`가 "a만 중복 제거하고 b는 아무거나"라고 생각하기 쉬운데, **아니다.** `(a, b)` 조합 전체 기준이다.

---

## 4. `DISTINCT`와 집계 — `COUNT(DISTINCT ...)`

14강에서 봤다. "길드 종류가 몇 개?"

```sql
SELECT COUNT(DISTINCT guild_id) FROM users;   -- 3 (NULL 제외)
```

`DISTINCT`를 집계 함수 안에 넣으면 **고유값만 집계**한다.

```sql
SELECT
    COUNT(DISTINCT guild_id) AS 길드수,
    COUNT(DISTINCT level)    AS 레벨종류수
FROM users;
```

---

## 5. `DISTINCT` vs `GROUP BY`

이 둘은 비슷한 일을 할 수 있다.

```sql
-- 둘 다 "존재하는 guild_id 목록"을 준다
SELECT DISTINCT guild_id FROM users;

SELECT guild_id FROM users GROUP BY guild_id;
```

결과가 같다. 그럼 언제 뭘 쓰나?

| 상황 | 추천 |
|------|------|
| 단순히 중복만 없애고 싶다 | **`DISTINCT`** (의도가 명확) |
| 그룹별로 **집계(COUNT/SUM 등)** 도 필요하다 | **`GROUP BY`** |

```sql
-- 집계가 붙으면 GROUP BY가 자연스럽다
SELECT guild_id, COUNT(*) FROM users GROUP BY guild_id;
```

> 💡 "중복 제거가 목적" → `DISTINCT`, "묶어서 계산이 목적" → `GROUP BY`. 의도를 코드로 드러내자.

---

## 6. `DISTINCT ON` — PostgreSQL 전용 강력 기능

"각 길드에서 **레벨이 가장 높은 유저 한 명씩**"을 뽑고 싶다면?

```sql
SELECT DISTINCT ON (guild_id)
    guild_id, name, level
FROM users
ORDER BY guild_id, level DESC;
```

결과 (길드별 1명씩):

```text
 guild_id |  name  | level
----------+--------+-------
        1 | 김철수 | 20
        2 | 정대훈 | 50
        3 | 최지수 | 30
   (null) | 홍길동 | 1
```

### 작동 원리

- `DISTINCT ON (guild_id)` = "guild_id가 같은 것 중 **첫 번째 행만**" 남긴다.
- "첫 번째"가 누구냐는 **`ORDER BY`가 결정**한다.
- 그래서 `ORDER BY guild_id, level DESC`로 정렬하면, 각 길드에서 레벨 내림차순의 맨 위(=최고 레벨)가 선택된다.

> ⚠️ **규칙:** `DISTINCT ON`의 괄호 안 열이 **`ORDER BY`의 맨 앞**에 와야 한다. 안 그러면 에러나 엉뚱한 결과.

이건 "그룹별 대표 1행 뽑기"에 아주 편하다. 윈도우 함수(27강)로도 같은 걸 할 수 있지만, `DISTINCT ON`이 더 간결하다.

---

## 7. 흔한 실수

### ① `DISTINCT`는 함수가 아니다

```sql
SELECT DISTINCT(name), level FROM users;   -- 헷갈리는 표기
```
`DISTINCT(name)`처럼 괄호를 써도 동작은 하지만, 이건 `DISTINCT name, level` (두 열 조합)과 같다. **`DISTINCT`는 특정 열에만 적용되는 게 아니라 `SELECT` 행 전체에 적용**된다. 괄호가 오해를 부르니 쓰지 말자.

### ② 정렬 없이 `DISTINCT ON`

```sql
SELECT DISTINCT ON (guild_id) * FROM users;  -- ORDER BY 없음 → 어느 행이 남을지 불확실
```
`DISTINCT ON`은 거의 항상 `ORDER BY`와 짝.

---

## 8. 실습

```sql
-- 1) 존재하는 길드 목록
SELECT DISTINCT guild_id FROM users;

-- 2) 유저들이 보유한 아이템 이름 종류 (중복 없이)
SELECT DISTINCT item_name FROM items ORDER BY item_name;

-- 3) 길드 종류 개수
SELECT COUNT(DISTINCT guild_id) AS 길드수 FROM users;

-- 4) (길드, 레벨) 조합의 고유 목록
SELECT DISTINCT guild_id, level FROM users ORDER BY guild_id, level;

-- 5) 각 길드의 최고 레벨 유저 (DISTINCT ON)
SELECT DISTINCT ON (guild_id) guild_id, name, level
FROM users
ORDER BY guild_id, level DESC;

-- 6) 각 유저가 가장 최근에 얻은 아이템 1개씩
SELECT DISTINCT ON (user_id) user_id, item_name, acquired_at
FROM items
ORDER BY user_id, acquired_at DESC;
```

---

## 9. 이번 강의에서 기억할 것

1. **`SELECT DISTINCT`** 는 결과 행의 중복을 제거한다.
2. 여러 열을 적으면 **그 열들의 조합** 기준으로 중복 판단.
3. **`COUNT(DISTINCT 열)`** 로 고유값 개수.
4. **중복 제거 목적은 `DISTINCT`, 집계 목적은 `GROUP BY`** — 의도에 맞게.
5. **`DISTINCT ON (열)`** 은 "그룹별 대표 1행 뽑기" (PG 전용, `ORDER BY` 필수).

---

## 10. 3단계 마무리

**3단계(SELECT 시리즈) 완료!** SQL에서 가장 많이 쓰는 영역을 다 훑었다.

| 강의 | 핵심 |
|------|------|
| 10강 | `SELECT` 기본 — 열 선택, `AS`, 계산식, `||` |
| 11강 | `WHERE` — 비교/논리/`BETWEEN`/`IN`/`LIKE`/`IS NULL` |
| 12강 | `ORDER BY`/`LIMIT`/`OFFSET` — 정렬과 페이징 |
| 13강 | `GROUP BY`/`HAVING` — 그룹 집계, WHERE vs HAVING |
| 14강 | 집계 함수 — COUNT/SUM/AVG/MAX/MIN, NULL 주의, FILTER |
| 15강 | `DISTINCT` — 중복 제거, `DISTINCT ON` |

이제 **하나의 테이블**은 자유자재로 조회할 수 있다. 다음 단계에선 **여러 테이블을 연결**하는 법을 배운다.

---

## 11. 다음 강의 예고 — 4단계 시작

**16강 — Primary Key / Foreign Key**
지금까지 `guild_id`로 테이블을 연결하는 시늉만 했다. 이제 **제대로 된 테이블 간 관계**를 정의한다. 기본 키와 외래 키 — JOIN(17강~)을 배우기 위한 토대다.

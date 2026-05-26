# 12강 — `ORDER BY`, `LIMIT`, `OFFSET`

> **목표:** 조회 결과를 정렬하고, 개수를 제한하고, 페이지로 나눈다.
> **선수 지식:** 10~11강

---

## 1. 정렬 — `ORDER BY`

```sql
SELECT 열목록
FROM 테이블
WHERE 조건
ORDER BY 정렬기준;
```

> ⚠️ **중요한 사실:** `ORDER BY`를 안 쓰면 **행의 순서는 보장되지 않는다.** "넣은 순서대로 나오겠지"는 착각. 순서가 중요하면 반드시 `ORDER BY`.

```sql
-- 레벨 낮은 순 (오름차순, 기본값)
SELECT name, level FROM users ORDER BY level;

-- 레벨 높은 순 (내림차순)
SELECT name, level FROM users ORDER BY level DESC;
```

| 키워드 | 의미 | 비유 |
|--------|------|------|
| `ASC` | 오름차순 (작은→큰). **기본값** | 1, 2, 3 / A, B, C |
| `DESC` | 내림차순 (큰→작은) | 3, 2, 1 / Z, Y, X |

`ASC`는 생략 가능 (기본값이라서). 내림차순은 `DESC`를 꼭 써야 한다.

---

## 2. 여러 기준으로 정렬

쉼표로 우선순위를 둔다. **앞쪽이 1순위.**

```sql
-- 길드별로 묶고(1순위), 같은 길드 안에선 레벨 높은 순(2순위)
SELECT name, guild_id, level
FROM users
ORDER BY guild_id ASC, level DESC;
```

1순위 값이 같을 때만 2순위가 작동한다. C++의 다중 키 정렬과 같은 개념.

---

## 3. 정렬 기준 다양하게

### 별칭으로 정렬

```sql
SELECT name, level * 100 AS exp
FROM users
ORDER BY exp DESC;     -- 계산된 별칭으로 정렬 가능
```

### 열 번호로 정렬 (간편하지만 비추)

```sql
SELECT name, level FROM users ORDER BY 2 DESC;  -- 2번째 열(level)
```

읽기 어려워서 실무에선 잘 안 쓴다. 이름으로 쓰자.

### 표현식으로 정렬

```sql
SELECT name FROM users ORDER BY LENGTH(name) DESC;  -- 이름 긴 순
```

---

## 4. NULL의 정렬 위치

NULL은 정렬에서 어디로 갈까? PostgreSQL 기본:

- `ASC`일 때 → NULL이 **맨 뒤**
- `DESC`일 때 → NULL이 **맨 앞**

직접 지정도 가능:

```sql
SELECT name, guild_id
FROM users
ORDER BY guild_id ASC NULLS FIRST;   -- NULL을 맨 앞으로

SELECT name, guild_id
FROM users
ORDER BY guild_id DESC NULLS LAST;   -- NULL을 맨 뒤로
```

---

## 5. 개수 제한 — `LIMIT`

결과의 **앞에서 N개만** 가져온다.

```sql
-- 레벨 상위 3명 (랭킹 TOP 3!)
SELECT name, level
FROM users
ORDER BY level DESC
LIMIT 3;
```

> 💡 `LIMIT`은 거의 항상 `ORDER BY`와 함께 쓴다. 정렬 없이 `LIMIT 3`만 하면 "아무 3개"가 나온다 (의미 없음).

---

## 6. 건너뛰기 — `OFFSET`

앞에서 N개를 **건너뛰고** 그 다음부터.

```sql
-- 4등부터 6등까지 (3개 건너뛰고 3개)
SELECT name, level
FROM users
ORDER BY level DESC
LIMIT 3 OFFSET 3;
```

`OFFSET 3` = 앞 3개 스킵, `LIMIT 3` = 거기서 3개. 즉 4~6번째.

---

## 7. 페이징(Pagination) — 실전 패턴

게시판이나 랭킹의 "1페이지, 2페이지"가 바로 이거다.

페이지당 10개를 보여준다고 하면:

```sql
-- 1페이지 (1~10등)
SELECT name, level FROM users ORDER BY level DESC LIMIT 10 OFFSET 0;

-- 2페이지 (11~20등)
SELECT name, level FROM users ORDER BY level DESC LIMIT 10 OFFSET 10;

-- 3페이지 (21~30등)
SELECT name, level FROM users ORDER BY level DESC LIMIT 10 OFFSET 20;
```

공식: **`OFFSET = (페이지번호 - 1) × 페이지크기`**

| 페이지 | LIMIT | OFFSET |
|--------|-------|--------|
| 1 | 10 | 0 |
| 2 | 10 | 10 |
| 3 | 10 | 20 |
| N | 10 | (N-1)×10 |

> ⚠️ **성능 주의:** `OFFSET`이 매우 커지면(예: `OFFSET 1000000`) 느려진다. DB가 앞의 100만 개를 다 세고 버리기 때문. 대용량 페이징은 "커서 기반(keyset) 페이징"이라는 다른 기법을 쓰는데, 지금은 개념만 알아두자.

---

## 8. 실행 순서 한 번 짚기

SQL은 **적는 순서**와 **실제 실행 순서**가 다르다. 알아두면 이해가 깊어진다.

적는 순서:
```
SELECT → FROM → WHERE → ORDER BY → LIMIT
```

실제 실행 순서:
```
FROM (테이블 읽기)
  → WHERE (행 필터)
  → SELECT (열 선택)
  → ORDER BY (정렬)
  → LIMIT/OFFSET (자르기)
```

그래서 **`WHERE`로 거른 다음 → 정렬 → 자르기** 순으로 처리된다. (13강에서 `GROUP BY`/`HAVING`까지 합쳐 더 정교하게 본다.)

---

## 9. 실습

```sql
-- 1) 레벨 높은 순 전체 랭킹
SELECT name, level FROM users ORDER BY level DESC;

-- 2) 코인 부자 TOP 3
SELECT name, coin FROM users ORDER BY coin DESC LIMIT 3;

-- 3) 길드별 정렬 후 길드 내 레벨 순
SELECT name, guild_id, level
FROM users
ORDER BY guild_id, level DESC;

-- 4) 레벨 20 이상만 추려서 코인 적은 순 5명
SELECT name, level, coin
FROM users
WHERE level >= 20
ORDER BY coin ASC
LIMIT 5;

-- 5) 페이징: 2페이지 (페이지당 2명, 레벨 순)
SELECT name, level
FROM users
ORDER BY level DESC
LIMIT 2 OFFSET 2;

-- 6) 길드 없는 사람을 맨 뒤로
SELECT name, guild_id
FROM users
ORDER BY guild_id ASC NULLS LAST;
```

---

## 10. 이번 강의에서 기억할 것

1. **`ORDER BY`가 없으면 순서는 보장되지 않는다.**
2. **`ASC`(오름차, 기본) / `DESC`(내림차)**, 여러 기준은 쉼표로 우선순위.
3. **`LIMIT N`** 으로 개수 제한, 보통 `ORDER BY`와 함께.
4. **`OFFSET N`** 으로 건너뛰기 → `LIMIT` + `OFFSET` = 페이징.
5. `OFFSET`이 크면 느려진다 (대용량은 다른 기법 필요).
6. 실행 순서: **FROM → WHERE → SELECT → ORDER BY → LIMIT.**

---

## 11. 다음 강의 예고

**13강 — `GROUP BY`, `HAVING`**
지금까진 개별 행을 봤다. 이제 **"길드별 인원수"**, **"길드별 평균 레벨"** 처럼 **묶어서 집계**하는 법을 배운다. 14강의 집계 함수와 짝을 이루는 핵심 개념.
